---
title: "The ML and Infrastructure Architecture Behind striff.io"
date: 2026-04-28 14:00:00
og_image: /images/striff-io-screenshot.png
tags: [mlops, kubernetes, redis, graph neural networks, ml engineering, system design]
toc: true
description: "The production pipeline behind [striff.io](https://striff.io) in spring 2026, when a graph neural network scored the changed edges of a dependency graph, and the three places we chose the boring option: a Redis list instead of Kafka, in-process ONNX instead of a model server, and graceful degradation."
excerpt_separator: <!--more-->
---

> *Update, September 2026.* This post describes Striff as it ran in spring 2026. The graph neural network scorer was retired in August 2026 and the detector pipeline followed in September; Striff now checks each pull request against the architecture documents already in the repository. The sections on the ONNX scorer describe a component that no longer exists. The queue, the scoped parse and the heap sizing are still how the service runs. The current system is described in [a post on the Striff blog](https://striff.io/blog/design-docs-are-enforceable-now).

Until August 2026, [striff.io](https://striff.io) ran a pipeline that parsed GitHub pull requests into typed dependency graphs, scored the changed dependencies with a graph neural network, and posted the structural findings as a GitHub check next to your CI. This post is a walkthrough of how that system was built, the specific problems that forced each design decision, and the tradeoffs we lived with. The infrastructure patterns here build directly on the MLOps blueprint described in an [earlier post]({% post_url 2024-01-09-mlops-blueprint %}) (single-build artifacts, Vault, Argo CD, blue/green rollouts). The GNN model itself is covered in a [companion post]({% post_url 2026-04-28-detecting-architectural-anomalies-gnn %}).

One thing to set expectations: striff has no message bus and no model server. A fair amount of this post is about why not.

![striff.io screenshot](/images/striff-io-screenshot.png){: .light-border }

<!--more-->

---

## How the Architecture Evolved

Like most pipelines, striff.io started synchronous. A GitHub webhook arrived, the API parsed the repository, built a subgraph, ran inference, called the LLM, enriched the diagram, and returned the result in a single request thread. That is a reasonable starting point and it worked well enough at low volume.

Three things pushed us toward the architecture described below.

Repository parsing has wide tail latency. A small TypeScript service parses in milliseconds. A Java monorepo with thousands of files and deep transitive imports can take several seconds, and the variance is hard to predict in advance. Under concurrent load, the slow parses pile up and thread pool exhaustion becomes a real ceiling.

LLM annotation adds five to thirty seconds depending on provider load. That is not a tail latency concern, it is the median case. Holding a request thread open while waiting on an external API does not hold up at scale.

The subtler issue is failure coupling. When everything runs in a single synchronous request, every component fails together. A timeout in the LLM call returns an error to the user even when the graph was built and the symbolic facts were computed correctly. The most reliable parts of the pipeline become invisible behind the least reliable one. Early on, I spent a week chasing what looked like a parsing failure before realizing the synchronous LLM timeout was swallowing every upstream error with it.

---

## What striff.io Does

[striff.io](https://striff.io) takes a GitHub pull request and reviews what it does to the codebase's structure. Changed files become seed nodes in a typed dependency graph extracted by striff-lib. A symbolic analysis layer computes deterministic facts: dependency cycles, package boundary crossings, fan-in blast radius, OOP metric deltas. A distilled GCN with an edge-prediction head, running on ONNX Runtime, scores the dependency edges touching changed components for structural surprise. Findings are posted as a GitHub check and rendered as notes on an SVG class diagram.

One rule shapes the rest of the pipeline: the deterministic layer decides what gets said, and the LLM only decides how it reads. A finding exists because a detector computed it, not because a model found it plausible. That keeps the output reproducible, and it means a clean PR gets no findings at all.

The whole system runs on DigitalOcean Kubernetes, provisioned with Terraform, deployed via ArgoCD, and monitored with Prometheus and Grafana.

<a class="post-figure__link" href="/images/striff-why-visual.svg" target="_blank" rel="noopener"><img src="/images/striff-why-visual.svg" class="post-figure"/></a>

<a class="post-figure__link" href="/images/striff-architecture-diagram.svg" target="_blank" rel="noopener"><img src="/images/striff-architecture-diagram.svg" class="post-figure"/></a>

---

## Async Without a Message Bus

The API does not do ML work on the request thread. A GitHub webhook arrives at `GitHubAppController`, which verifies the HMAC signature, drops duplicates through `GitHubWebhookDedupService`, pushes a job onto a **Redis list**, and returns 202 Accepted. Everything downstream is asynchronous.

Durability is the reason for Redis rather than an in-process queue, and it is worth being precise about the failure it prevents. The first version used a `ThreadPoolTaskExecutor` directly: 8 core threads, 64 queue slots, jobs living purely in JVM memory. Two things break at that design. Jobs are lost on pod restart, and GitHub only retries a webhook about three times over roughly twenty seconds, so a deploy during a busy minute silently drops reviews that nobody will ever ask for again. And each pod has its own queue, so one pod can saturate while another sits idle, with no way to shed work between them.

A Redis list fixes both without introducing a broker. Workers `BRPOP` from a shared key, so any pod can pick up any job and load balances itself. Redis was already in the cluster for striff caching and prefetch storage, and was already configured for this to be safe: AOF persistence with `appendfsync everysec`, and `volatile-lru` eviction so that only keys with a TTL are evictable and queue entries cannot be dropped under memory pressure. Each pod runs a small pool of long-lived worker threads (`GitHubEventWorker`, a `SmartLifecycle` that drains in-flight jobs on shutdown).

**Why not Kafka.** Kafka gives you ordering, consumer groups, and replay. We don't need any of the three: reviews are independent per PR, and a webhook event is worthless once it's a few minutes stale. What we would get for certain is a broker to run and a third stateful system in the cluster. A list and `BRPOP` covers the requirement.

The AI review runs one hop further in, on a small bounded executor: core 2, max 4, queue 50 (`AsyncConfig`). The bound matters more than the numbers. Each review holds a parsed graph, a feature matrix and an ONNX session in heap, so unbounded concurrency here doesn't degrade latency gracefully, it runs out of memory. When the queue fills, the rejection handler marks the operation `FAILED` with error code `QUEUE_FULL` rather than dropping it silently — a state the client can see and retry against.

We alert on `striff_ai_review_executor_queued > 25` for 10 minutes. That fires when reviews are arriving faster than they complete, which in practice is either an LLM provider slowdown or a run of large repositories, and the queue-depth signal catches it well before any request errors.

The review status endpoint is a MongoDB read; the browser extension polls it, and the GitHub App posts a check run when the review completes.

{% include striff-flow-demo.html %}

---

## Inference Stays In-Process

The GNN runs inside the application JVM on ONNX Runtime. `OnnxArchitecturalScorer` loads the model from the packaged resources at startup, holds the session for the pod's lifetime, and scores synchronously inside the review task. There is no inference service, no gRPC hop, no GPU.

This is the decision most likely to look wrong on a diagram and be right in production. Per-review inference is a single forward pass over a subgraph capped at 500 nodes, which is milliseconds of CPU. Standing up a model server would add a network hop and a second deployment to that, plus a new failure mode for a call that currently cannot fail independently of the process making it. The scaling argument for a model server (batching many small requests into one GPU pass) requires request arrival rates we do not have: reviews arrive at human-PR frequency, so the effective batch size would be one almost always, and batching one request is just latency with extra steps.

There is a real cost to in-process inference and it is memory, not latency. At six replicas under the HPA maximum, six copies of the model weights sit in six JVMs doing nothing most of the time. That is the price of the simplicity, it is currently a few hundred megabytes, and it is the number to watch: **the day model weights grow enough that per-replica duplication dominates the memory budget, or inference latency starts landing in the request path, is the day a model server earns its place.** Neither is true yet.

The contract risk that a service boundary would have introduced still exists, just in a different form. The scorer's input contract is a 403-dimensional feature vector with a specific layout (text embedding, then metrics, then type one-hot, then language one-hot, then the synthetic flag) plus an `edge_queries` tensor. A retrained model exported with the OOP metrics in a different order produces plausible, wrong scores and throws nothing. Silent numerical wrongness does not appear in error rates. Two things guard it: `ModelMetadata` pins the expected dimension and `OnnxArchitecturalScorer` hard-fails at startup if the loaded model disagrees or if the `edge_queries` input is missing, so a mismatched model kills the pod rather than quietly scoring garbage. Crashing on a contract violation is the correct behaviour for a model whose output nobody can eyeball.

---

## Parsing Large Codebases Without Drowning

Parsing is the most expensive operation in the pipeline and the one most teams building code analysis tools get wrong. The naive approach parses the entire repository on every PR event. That does not scale and it is also wrong: a review does not need the full repository graph, it needs the structural neighbourhood of what changed. Parse less, but parse smarter. On a 1000-file Java codebase, striff-lib's full pipeline (file I/O, Clarpse parsing, reference classification, relationship extraction, diff computation, model merge) takes roughly four seconds. Most of that time is in relationship extraction, which is why scoping the parse set matters so much.

The pipeline builds this neighbourhood in three steps via ScopedFileSelector, ScopedParseService, and NeighborhoodExpander.

**Scoped file selection.** Changed files are parsed first to extract the set of component names they declare. A fast text scan then runs across the full repository to find every file containing any of those names. This is a string search, not a parse. A tier-based budget trims the resulting candidate set: PR files first, same-directory files second, text-match files last. Lower-priority files are dropped when the budget is exhausted.

**Time-boxed parsing.** ScopedParseService parses the candidate set under a hard per-language deadline. If parsing does not finish in time, it returns whatever it has completed. This is a deliberate design choice: a partial graph is better than a timeout. The review that follows will be weaker but the user gets something rather than an error.

**Neighbourhood expansion.** NeighborhoodExpander runs a bidirectional BFS from the seed nodes (the changed components), following edges in both directions for three hops. This captures both what the changed components depend on and what depends on them: the structural blast radius of the PR. Node count is capped at 500, with non-seed nodes dropped in hop-distance order when the cap is hit. The subgraph is deterministic and reproducible across runs.

For most PRs on most repositories this produces a subgraph of a few dozen nodes. For large cross-cutting refactors it might reach the cap.

![BFS neighbourhood expansion](/images/bfs-neighbourhood-diagram.svg){: .light-border }

The memory side matters here. A 403-dimensional float feature matrix for a 500-node subgraph is about 800KB. Under concurrent reviews, several of those live in the JVM heap at once alongside cached MongoDB documents and the ONNX model weights, which is the real reason the review executor is bounded at four threads.

The heap is sized explicitly, `-Xms1g -Xmx4g`, rather than with `-XX:MaxRAMPercentage`. That choice gets made the other way in most Kubernetes deployments, so it is worth the sentence: a percentage of the container limit silently re-sizes the heap whenever someone edits the pod's memory request, and this workload has significant *native* memory outside the heap (ONNX Runtime's arenas and the parser's buffers) that the percentage does not know about. A heap that grows to consume the headroom native allocation needs produces an OOM kill that looks like a memory leak and is not. Pinning the heap makes the native budget explicit and the failure reproducible. `-XX:+ExitOnOutOfMemoryError` is set alongside it: if the JVM does exhaust heap, the pod dies and Kubernetes replaces it, rather than limping along in a state where inference might silently produce garbage.

---

## Shipping a New Model

Model weights ship inside the application image. A new model is a new build, a new image tag, and a normal deployment, which sounds unsophisticated until you consider what it buys: the model version and the code version can never disagree. The feature-vector layout in `FeatureBuilder` and the tensor the model expects are the single most coupled pair of things in this system, and packaging them together makes a mismatched pair impossible to deploy rather than merely unlikely.

The tradeoff is real and worth naming. Retraining requires an application deploy, so the model cannot be updated independently or rolled back on its own, and image size grows with the weights. For a model that changes a few times a quarter, that is a good trade. For one retrained nightly, it would not be, and that is the second condition (alongside memory pressure) that would push inference out into its own service.

Model behaviour changes discretely between versions, which is the argument against a rolling update here: mixing old and new scoring inside one traffic window produces reviews that differ for reasons no user can see. A blue/green cutover with an instant rollback path is worth briefly running two stacks. Experiment tracking (MLflow, covered in the MLOps blueprint post) keeps each shipped model traceable to the training run that produced it, which matters because "the scores changed and we do not know why" is otherwise unanswerable months later.

---

## Degradation Modes

AIReviewService degrades rather than failing. Every failure mode below the top tier produces a weaker but still honest output.

<a class="post-figure__link" href="/images/striff-reviewnote.png" target="_blank" rel="noopener"><img src="/images/striff-reviewnote.png" class="post-figure"/></a>

**Full pipeline.** The scoped parse completes within budget, NeighborhoodExpander produces a valid subgraph, OnnxArchitecturalScorer returns edge scores, and `LlmReviewCoordinator` receives symbolic facts plus scored edges as structured context. This is the highest-quality path.

**Symbolic-only fallback.** If the scoped parse times out, if NeighborhoodExpander produces an empty result, or if the ONNX scorer throws for any reason, the review continues with deterministic symbolic facts only. SymbolicFactsComputer runs Kosaraju SCC on JGraphT, computes boundary crossings and fan-in blast radius, and assembles the payload without anomaly scores. Because the GNN's scores are evidence rather than an origin of findings (the [companion post]({% post_url 2026-04-28-detecting-architectural-anomalies-gnn %}) explains why), losing them costs prioritisation detail, not correctness. The findings that survive are the deterministic ones, which were the only ones users ever saw.

Each degradation mode is logged explicitly and surfaces in metrics. Monitoring which path was taken is not optional: consistent fallback to symbolic-only is a signal worth alerting on, because it means graph construction is systematically failing and every review is quietly thinner without any individual request producing an error.

### The failure that is not a failure, and the one that is

An empty review is a legitimate outcome. Most pull requests do not damage the architecture, and the correct output for those is silence: no findings, no invented "considerations", no note manufactured so the tool looks busy. Striff staying quiet on a clean PR is the product working.

This creates a subtle hazard that took a redesign to close properly. A review that found nothing and a review that *could not analyse anything* produce byte-identical empty artifacts, and they mean opposite things. "Analysed, clean" is a result. "Never analysed" is an outage wearing a result's clothing, and the moment it is persisted as READY it is indistinguishable from good news forever after.

The early fix was a quality gate: reject any review that produced no visible note. That was the wrong lever, because it treats the legitimate case (clean PR) as broken in order to catch the illegitimate one. The current design makes the bad state unrepresentable instead: a review can only be produced by a live analysis that actually ran, so there is no code path that reaches the surfacing stage without a real diff behind it. A review scheduled without an analysis result throws rather than publishing an empty artifact. Nothing needs to be checked at the end, because nothing can get there wrongly.

That is the general shape worth stealing from this section: when two states are externally identical but semantically opposite, adding a validation is weaker than removing the path that produces the ambiguous one.

Two alerts cover the rest. `StriffAPIAIReviewFailuresHigh` fires when more than ten reviews fail in fifteen minutes, and `StriffAPIAIReviewQueueBacklog` fires when the executor queue depth exceeds 25 for ten minutes. The queue-depth alert catches the failure mode that pure error-rate monitoring misses: the system falling behind on review generation without any individual request returning an error.

![Degradation tier flowchart](/images/degradation-flowchart.svg){: .light-border }

---

## Code Referenced in This Post

<div style="border:1px solid rgba(15,23,42,0.08);border-radius:12px;padding:14px 18px;margin:16px 0;background:rgba(255,255,255,0.6);">
<div style="display:flex;flex-wrap:wrap;gap:6px;align-items:center;">
<a href="https://github.com/hadi-technology/striff-gnn"><img src="https://img.shields.io/badge/GNN%20Training-striff--gnn-blue?logo=github" alt="striff-gnn"></a>
<a href="https://github.com/hadi-technology/striff-lib"><img src="https://img.shields.io/badge/Graph%20Parsing-striff--lib-blueviolet?logo=github" alt="striff-lib"></a>
<a href="https://github.com/hadi-technology/clarpse"><img src="https://img.shields.io/badge/Static%20Analysis-clarpse-6a0dad?logo=github" alt="clarpse"></a>
<a href="https://github.com/hadi-technology/mlops-blueprint"><img src="https://img.shields.io/badge/MLOps%20Pipeline-mlops--blueprint-teal?logo=github" alt="mlops-blueprint"></a>
</div>
</div>

---

## Where to Go From Here

[striff.io](https://striff.io) is live. You can run it on any public GitHub repository today. There is also a [Chrome extension](https://github.com/hadi-technology/striff-browser-extension) for inline PR review on GitHub.

[striff-lib](https://github.com/hadi-technology/striff-lib) is open source (available on Maven as `io.github.hadi-technology:striff-lib`). The parsing and diagram generation core is available if you want to explore the graph extraction layer or build on it.

---

*Muntazir Fadhel builds production AI and ML infrastructure. He is the founder of HADI Technology. [Technical Profile](/downloads/MFadhel_Engagement_Brief.pdf) · [Get in Touch](/contact/)*
