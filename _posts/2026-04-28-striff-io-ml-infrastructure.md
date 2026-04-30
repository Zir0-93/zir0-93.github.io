---
title: "The ML and Infrastructure Architecture Behind striff.io"
date: 2026-04-28 14:00:00
og_image: /images/striff-io-screenshot.png
tags: [mlops, kubernetes, kafka, graph neural networks, ml engineering, system design]
toc: true
description: "A walkthrough of the async Kafka-staged pipeline, Triton-based inference serving, and degradation hierarchy that powers [striff.io](https://striff.io)'s architectural review system. Covers why the pipeline moved from synchronous to event-driven, how three independent Kafka worker tiers decouple graph construction, GNN scoring, and LLM annotation, the distributed systems problems that Triton separation introduces, and the three-tier degradation strategy that ensures every failure mode still produces a useful review."
excerpt_separator: <!--more-->
---

[striff.io](https://striff.io) runs a neurosymbolic ML pipeline that parses GitHub pull requests into typed dependency graphs, scores changed components with a graph neural network, and renders AI-generated architectural review notes directly onto the diagram. This post is a walkthrough of how the system is built, the specific problems that forced each design decision, and the tradeoffs we are living with. The infrastructure patterns here build directly on the MLOps blueprint described in an [earlier post]({% post_url 2024-01-09-mlops-blueprint %}) -- single-build artifacts, Vault, Argo CD, blue/green rollouts -- extended with Kafka staging and Triton inference. The GNN model itself is covered in a [companion post]({% post_url 2026-04-28-detecting-architectural-anomalies-gnn %}).

![striff.io screenshot](/images/striff-io-screenshot.png){: .light-border }

<div style="border:1px solid rgba(15,23,42,0.08);border-radius:12px;padding:14px 18px;margin:16px 0;background:rgba(255,255,255,0.6);">
<p style="margin:0 0 8px 0;font-size:0.75rem;font-weight:700;letter-spacing:0.08em;text-transform:uppercase;color:#6b7280;">Relevant Repos</p>
<div style="display:flex;flex-wrap:wrap;gap:6px;align-items:center;">
<a href="https://github.com/hadi-technology/striff-gnn"><img src="https://img.shields.io/badge/GNN%20Training-striff--gnn-blue?logo=github" alt="striff-gnn"></a>
<a href="https://github.com/hadi-technology/striff-lib"><img src="https://img.shields.io/badge/Graph%20Parsing-striff--lib-blueviolet?logo=github" alt="striff-lib"></a>
<a href="https://github.com/hadi-technology/clarpse"><img src="https://img.shields.io/badge/Static%20Analysis-clarpse-6a0dad?logo=github" alt="clarpse"></a>
<a href="https://github.com/hadi-technology/mlops-blueprint"><img src="https://img.shields.io/badge/MLOps%20Pipeline-mlops--blueprint-teal?logo=github" alt="mlops-blueprint"></a>
</div>
</div>
<!--more-->

---

## How the Architecture Evolved

Like most pipelines, striff.io started synchronous. A GitHub webhook arrived, the API parsed the repository, built a subgraph, ran inference, called the LLM, enriched the diagram, and returned the result in a single request thread. That is a reasonable starting point and it worked well enough at low volume.

Three things pushed us toward the architecture described below.

Repository parsing has wide tail latency. A small TypeScript service parses in milliseconds. A Java monorepo with thousands of files and deep transitive imports can take several seconds, and the variance is hard to predict in advance. Under concurrent load, the slow parses pile up and thread pool exhaustion becomes a real ceiling.

LLM annotation adds five to thirty seconds depending on provider load. That is not a tail latency concern, it is the median case. Holding a request thread open while waiting on an external API does not hold up at scale.

The subtler issue is failure coupling. When everything runs in a single synchronous request, every component fails together. A timeout in the LLM call returns an error to the user even when the graph was built and the symbolic facts were computed correctly. The most reliable parts of the pipeline become invisible behind the least reliable one. I spent a week debugging what looked like a parsing failure before realizing the LLM timeout was swallowing everything upstream.

---

## What striff.io Does

[striff.io](https://striff.io) takes a GitHub pull request and generates a visual architecture diff. Changed files become seed nodes in a typed dependency graph extracted by striff-lib. A symbolic analysis layer computes deterministic facts: dependency cycles, package boundary crossings, fan-in blast radius, OOP metric deltas. A distilled R-GCN running on ONNX Runtime scores each changed component for structural anomaly. Both signals feed a structured LLM agent payload. The agent produces tiered review notes rendered as sticky notes directly on the SVG class diagram.

The whole system runs on DigitalOcean Kubernetes, provisioned with Terraform, deployed via ArgoCD, and monitored with Prometheus and Grafana.

<img src="/images/striff-why-visual.svg" style="margin-left:auto; margin-right:auto; display: block; max-width: 700px;"/>

<img src="/images/striff-architecture-diagram.svg" style="margin-left:auto; margin-right:auto; display: block;"/>

---

## An Async, Kafka-Staged Pipeline

The API does not do ML work. When a GitHub webhook arrives, AIReviewService validates the event, writes a job record to MongoDB with status PENDING, publishes a `review.requested` event to Kafka, and returns 202 Accepted. Everything downstream is asynchronous.

The pipeline has three independent worker tiers, each with its own Kafka topic and consumer group:

- **Graph workers** consume `review.requested`, run ScopedFileSelector and TimeBoxedParser to build the subgraph, publish the result to `graph.ready`
- **GNN workers** consume `graph.ready`, call the Triton Inference Server, publish anomaly scores to `scores.ready`
- **LLM workers** consume `scores.ready`, assemble the structured payload via OptimizedReviewPayloadMapper, call GradientAIReviewCoordinator, publish the enriched review to `review.complete`

Each tier scales independently. Graph construction is CPU and I/O bound. Inference is compute-bound and benefits from GPU batching. LLM annotation is latency-bound on an external API and needs more replicas, not faster hardware. Kafka provides natural backpressure between them: if LLM calls are slow, the `scores.ready` topic grows and annotation workers slow down without any effect on how fast webhooks are ingested upstream.

We alert on `striff_ai_review_executor_queued > 25` for 10 minutes. When that fires, it means the annotation tier is falling behind the inference tier. That is almost always an LLM API rate limit or timeout issue, and it is a completely different operational response than a graph construction backlog. Having the tiers observable separately is what makes that distinction fast.

The review status endpoint is a MongoDB read. Clients poll or receive a webhook callback when the job reaches `review.complete`.

![Kafka topic topology](/images/kafka-topology-diagram.svg){: .light-border }

---

## Inference as a Separate Service

Separating Triton Inference Server from the application tier is not just a scaling decision. It introduces distributed systems problems that did not exist when everything ran in one process.

When Triton is unavailable, GNN workers must decide what to do. The answer connects directly to the degradation hierarchy: a worker that cannot reach Triton publishes a signal downstream indicating scoring failed, and the LLM annotation tier falls back to symbolic-only facts. The review completes with reduced quality rather than not completing at all. This is the right behaviour, but it requires that every service boundary in the pipeline has an explicit failure contract, not just an implicit assumption that the downstream service is always reachable.

Kafka's at-least-once delivery guarantee introduces idempotency requirements that in-process queues do not. A worker that crashes after processing a message but before committing its offset will re-process that message on restart. AIReviewService handles this by claiming operations before work starts and checking claim state before publishing downstream. A message that arrives for an already-claimed operation is discarded. The cost of re-processing a message is a no-op rather than a duplicate review appearing for the same PR.

Service boundaries also create contract risk that in-process calls do not have. The OnnxArchitecturalScorer input contract -- the 404-dimensional feature vector layout and typed edge index format -- is only valid for the model currently registered with Triton. Deploying a retrained model with a different input shape without updating FeatureBuilder on the worker side produces wrong anomaly scores without throwing an exception. This is a silent failure that does not appear in error rates. We caught this once in staging when a training run exported the feature vector with the OOP metrics in a different order. The scores looked plausible. They were wrong. The fix is contract tests between FeatureBuilder output and the ONNX model's expected input, run as part of CI on every change to either.

---

## Parsing Large Codebases Without Drowning

Parsing is the most expensive operation in the pipeline and the one most teams building code analysis tools get wrong. The naive approach parses the entire repository on every PR event. That does not scale and it is also wrong: a review does not need the full repository graph, it needs the structural neighbourhood of what changed. Parse less, but parse smarter. On a 1000-file Java codebase, striff-lib's full pipeline (file I/O, Clarpse parsing, reference classification, relationship extraction, diff computation, model merge) takes roughly four seconds. Most of that time is in relationship extraction, which is why scoping the parse set matters so much.

The pipeline builds this neighbourhood in three steps via ScopedFileSelector, TimeBoxedParser, and NeighborhoodExpander.

**Scoped file selection.** Changed files are parsed first to extract the set of component names they declare. A fast text scan then runs across the full repository to find every file containing any of those names. This is a string search, not a parse. A tier-based budget trims the resulting candidate set: PR files first, same-directory files second, text-match files last. Lower-priority files are dropped when the budget is exhausted.

**Time-boxed parsing.** The candidate file set is parsed under a hard deadline enforced by TimeBoxedParser. If parsing does not finish in time, it returns whatever it has completed. This is a deliberate design choice: a partial graph is better than a timeout. The review that follows will be weaker but the user gets something rather than an error.

**Neighbourhood expansion.** NeighborhoodExpander runs a bidirectional BFS from the seed nodes (the changed components), following edges in both directions for three hops. This captures both what the changed components depend on and what depends on them -- the structural blast radius of the PR. Node count is capped at 500, with non-seed nodes dropped in hop-distance order when the cap is hit. The subgraph is deterministic and reproducible across runs.

For most PRs on most repositories this produces a subgraph of a few dozen nodes. For large cross-cutting refactors it might reach the cap.

![BFS neighbourhood expansion](/images/bfs-neighbourhood-diagram.svg){: .light-border }

The memory side matters here. A 404-dimensional feature matrix for a 500-node subgraph is about 800KB. Under concurrent reviews, multiple feature matrices live in the JVM heap simultaneously alongside cached MongoDB documents and the ONNX model weights. The deployment runs with `-XX:MaxRAMPercentage=75` to leave headroom for the OS and ONNX Runtime's native memory. `-XX:+ExitOnOutOfMemoryError` is set deliberately: if the JVM runs out of heap, the pod crashes and Kubernetes restarts it clean rather than leaving a live pod where inference might silently produce garbage. We alert on JVM heap above 90% sustained for ten minutes before that happens.

---

## Model Serving: Why Triton

ONNX Runtime running in-process is the right starting point. It stops being the right answer once you scale. At six replicas under the HPA maximum, each replica carries the full model weights in its own JVM heap. Six copies of the same ONNX model doing nothing most of the time. Inference also has no batching: each review request runs a separate forward pass.

Triton Inference Server solves both. The R-GCN ONNX model is registered with Triton without changes to the model itself. GNN workers become thin gRPC clients: serialise the feature matrix and typed edge index, call Triton, receive anomaly scores. Triton accumulates requests within a configurable time window and batches them into a single forward pass on GPU. At low traffic the effective batch size is one and latency matches in-process inference. At high traffic the amortised cost per review drops substantially.

Model versioning also becomes a first-class operational concern once inference is separated from the application. Triton's version routing runs a new model version alongside the current one, routing a configurable traffic fraction to the candidate before cutover. Deploying a new model without a clear comparison to the current production model on held-out data is how you introduce silent quality regressions. Paired with MLflow for experiment tracking (covered in the MLOps blueprint post), every model version in production has a traceable lineage back to a training run with logged metrics.

For the rollout itself we use Argo Rollouts with a blue/green strategy. Model behaviour changes discretely between versions. A rolling update that mixes old and new scoring in the same traffic window produces review notes that are inconsistent in ways users cannot explain. A clean cutover with an instant rollback path is worth the cost of briefly running two serving stacks.

---

## Degradation Modes

AIReviewService implements a three-tier degradation hierarchy. Every failure mode produces a weaker but still useful output rather than returning an error.

<img src="/images/striff-reviewnote.png" style="margin-left:auto; margin-right:auto; display: block; max-width: 650px;"/>

**Full pipeline.** TimeBoxedParser completes within budget, NeighborhoodExpander produces a valid subgraph, OnnxArchitecturalScorer returns anomaly scores, and GradientAIReviewCoordinator receives symbolic facts plus scored components as structured context. This is the highest-quality path.

**Symbolic-only fallback.** If the scoped parse times out, if NeighborhoodExpander produces an empty result, or if the ONNX scorer throws for any reason, the review continues with deterministic symbolic facts only. SymbolicFactsComputer runs Kosaraju SCC on JGraphT, computes boundary crossings and fan-in blast radius, and assembles the agent payload without anomaly scores. Review notes are less precise about which components to prioritise, but they are grounded in real structural evidence.

**Retroactive symbolic-only.** When there is no live parse result at all -- because the review is being regenerated for a historical operation or because the service restarted mid-review -- persisted striff-lib artifacts are reloaded from MongoDB and a compact payload is rebuilt from them. `NeuroSymbolicService.annotateRetroactive()` handles this path. No fresh parse, no GNN scoring. This path exists so that historical operations remain reviewable without requiring the original repository parse state to be present. It is the thinnest path, but it means a service restart does not orphan old reviews.

Each degradation mode is logged explicitly and surfaces in metrics. Monitoring which path was taken is not optional: consistent fallback to symbolic-only is a signal worth alerting on, because it means graph construction is systematically failing and the quality of every review is degraded without any individual request producing an error. If every review on the dashboard shows "symbolic-only," something is broken even though no error fired.

There is a quality gate at the end of all three paths. A review is not considered complete until it produces at least one visible sticky note in the final enriched SVG. A nominally successful review that produces no visible output is treated as a failure rather than silently persisted. Silent successes that users cannot perceive are harder to detect than explicit errors because they do not raise error rates and they do not fire alerts. The alert `StriffAPIAIReviewFailuresHigh` fires when more than ten reviews fail in fifteen minutes, and `StriffAPIAIReviewQueueBacklog` fires when executor queue depth exceeds 25 for ten minutes. The queue depth alert catches the failure mode that pure error-rate monitoring misses: the system falling behind on review generation without any individual request returning an error.

![Degradation tier flowchart](/images/degradation-flowchart.svg){: .light-border }

---

## Where to Go From Here

[striff.io](https://striff.io) is live. You can run it on any public GitHub repository today. There is also a [Chrome extension](https://github.com/hadi-technology/striff-browser-extension) for inline PR review on GitHub.

[striff-lib](https://github.com/hadi-technology/striff-lib) is open source (available on Maven as `io.github.hadi-technology:striff-lib`). The parsing and diagram generation core is available if you want to explore the graph extraction layer or build on it.
