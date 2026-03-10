---
title: "What a Decade of ML Infrastructure Taught Me About LLMs"
date: 2026-03-07 00:01:00
og_image: /images/microservices.png
tags: [ai, llms, mlops, platform engineering, system design]
toc: true
description: "After close to a decade working on ML infrastructure, including GPU clusters, autoscaling pipelines, and model serving systems, the transition into LLM-based production systems turned out to be less of a clean break than the hype suggests. The problems do not change so much as evolve, and they get harder in specific ways. This post works through the areas where classical ML intuitions transfer directly into LLM operations, where they break down and need updating, and where the failure surfaces are genuinely new. Covering latency, reproducibility, data lineage, cost modeling, observability, and the unique challenges of agent systems, written for engineers who have operated traditional ML infrastructure and want an honest map of what carries over."
excerpt_separator: <!--more-->
---

I have spent close to a decade working on ML infrastructure, and by 2018, before MLOps was a job title most people recognized, the work already followed a familiar pattern: migrating pipelines into cloud environments, wiring together GPU clusters, and making autoscaling behave sensibly for workloads that spiked unpredictably. The models were rarely the hard part. Getting them to run reliably, cheaply, and observably in production was.

A few years ago I founded HADI Technology, a consultancy where that work continued across production AI platforms, model serving systems, and agent infrastructure. Over time, engagements shifted increasingly into LLM-based systems: serving open-source models on vLLM, building agent infrastructure, and operating AI-driven platforms at scale. What became clear through that transition is that the problems do not change as much as the current hype suggests. They evolve. They get harder in specific ways. But the years spent wrestling with traditional ML infrastructure turned out to be more relevant than I expected.

<!--more-->

{% include article-visual.html %}

Here is what transferred, and where the gaps are.

---

## Serving Latency

**In traditional ML,** latency was usually profileable. For a given model and input size, you could reason about throughput, queue depth, and resource saturation with reasonable confidence. Designing autoscaling clusters for GPU-intensive ML tasks meant developing precise instincts about when to scale out versus when to buffer. For a fixed-output classifier or embedding model, the latency distribution across requests was tight enough that p50 and p99 told a coherent story. You could set SLAs with confidence and hold them.

The autoscaling logic followed naturally from that predictability. If batch size is fixed and model forward passes take roughly constant time, you can translate queue depth into expected latency and scale accordingly. Infrastructure teams built dashboards that made sense, and the gap between what the metrics showed and what users experienced was manageable.

**With LLMs,** the variable that breaks those instincts is output length. It is determined at runtime, which means a batch of requests that looks uniform at ingress can produce wildly different completion times depending on how the model responds to each one. A user asking for a one-sentence summary and a user asking for a full report may submit requests that are identical in size but produce completions that differ by two orders of magnitude in token count.

This has cascading effects. First, it splits latency into two meaningfully different measurements: time to first token (TTFT) and total completion time. For interactive applications, TTFT is usually the SLA target that matters most, because users perceive the stream starting as the system responding. Total completion time matters for throughput and cost accounting, but it is a poor proxy for user experience. Traditional ML serving did not need to make this distinction.

Second, the decode phase of LLM inference is fundamentally sequential in a way that classifier inference is not. Prefill, the phase where the prompt is processed, is parallelizable across the GPU and runs quickly. Decode, where the model produces output one token at a time, is memory-bandwidth bound and cannot be easily accelerated by batching in the same way. The implication is that per-request GPU utilization curves look nothing like traditional ML serving, and autoscaling logic built around GPU utilization thresholds needs to be rethought from scratch.

Third, timeout handling becomes considerably more nuanced. A fixed request timeout that worked well for a classifier will aggressively cut off legitimate long-form completions. Per-token timeouts or streaming-aware deadlines are often the right primitive, but they require deeper integration with the serving layer than a simple HTTP timeout. *The habit of tracking p95 and p99 latency separately from p50, built over years of operating ML endpoints, turns out to matter even more, not less. It just needs to be applied to TTFT and time-per-output-token separately, not only to total request duration.*

---

## Pipeline Reproducibility

**In traditional ML,** one of the most reliable sources of production failures was environment drift. A pipeline that ran cleanly three months ago, run again today with subtly different library versions, data distributions, or infrastructure configuration, could produce a model that passes offline evaluation but degrades quietly in production. The fix was treating pipelines as reproducible artifacts: pinned environments captured in lockfiles, versioned data with lineage tracked from ingestion through feature generation, and logged hyperparameters that could reconstruct any training run from scratch. This was not a best practice so much as a survival habit. Every team that skipped it eventually paid for the omission.

The discipline extended to the full pipeline, not just the training step. Feature preprocessing logic that drifted between training and serving caused more subtle failures than most data issues. Data snapshots tied to specific model versions prevented the common mistake of evaluating a new model against data it had already seen. Reproducibility was treated as a system property, not a documentation task.

**With LLMs,** that discipline carries over directly, but the surface area for drift is considerably larger. In a traditional pipeline, the primary variables are training code and data. In an LLM-based system, the prompt, the model version, the context window handling strategy, the tool definitions, the behavior of any external APIs those tools call, and the temperature and sampling parameters are all variables too. Any of them can shift between runs and produce meaningfully different outputs without raising a single exception.

Prompt versioning deserves particular attention because it is frequently treated casually. A prompt is not documentation. It is a functional component of the system, and changes to it can alter model behavior as significantly as changes to a preprocessing function would in a traditional pipeline. Prompts should be versioned in the same repository as the code that calls them, with changes tracked and attributed. Running evaluations before and after prompt changes, rather than after deployment, is the equivalent of running unit tests before merging.

Model version pinning is the other area where teams consistently underinvest until something breaks. Referring to a model by an alias that points to the current stable release creates a dependency on an external party's release schedule. A model update by the provider can change output behavior in ways that break downstream parsing, tool call formatting, or response length assumptions. Pinning to a specific model version and upgrading deliberately, with evaluations, is the same discipline as pinning library versions in production software.

Non-determinism adds a further complication that traditional ML pipelines did not face to the same degree. Setting temperature to zero helps in testing contexts, but production systems often require some sampling variability for quality reasons. Evaluations need to account for this by running multiple samples and aggregating, rather than treating a single run as representative. The [mlops-blueprint](https://github.com/hadi-technology/mlops-blueprint) was an attempt to codify what end-to-end reproducibility looks like in practice. The same thinking applies directly to LLM pipelines, with prompt snapshots added to the list of artifacts that need versioning.

---

## Observability

**In traditional ML,** the lesson learned from operating production serving infrastructure across multiple environments was that observability is not optional. It is the difference between knowing a system is healthy and guessing. For model serving endpoints, the instrumentation targets were relatively well-defined: request rate, latency percentiles, error rate, prediction distribution, and model-level metrics like feature drift and output confidence histograms. A properly configured monitoring stack made failures diagnosable within minutes rather than hours.

The infrastructure side was also reasonably tractable. GPU utilization, memory pressure, pod restart frequency, and autoscaling events gave a clear picture of resource health. When something degraded, the metrics usually pointed to a cause quickly enough to act before user impact was severe.

**With LLMs,** all of that is still necessary, and a new category of signals is required that did not exist before. The infrastructure metrics are still worth collecting: GPU utilization, KV cache hit rate, queue depth, prefill throughput, and decode throughput all tell you whether the serving layer is healthy. But they say nothing about whether the model is doing what the application needs it to do.

Application-layer correctness metrics are the gap. These include the rate at which structured outputs parse successfully against the expected schema, the rate at which tool calls are formatted correctly and return expected results, the distribution of response lengths over time, and task completion rates where those can be defined. These signals require instrumentation at the application layer, not the infrastructure layer, and they require thought about what correctness actually means for each use case before the system goes to production.

Distributed tracing becomes significantly more important in agent systems than in traditional ML pipelines. A single user request may involve multiple model calls, tool invocations, external API calls, and reasoning steps that span several seconds. Understanding what happened in a failed or degraded request requires a trace that captures each step with timing, inputs, and outputs. Without that, debugging a subtle failure in an agent chain is largely guesswork. *A healthy Prometheus dashboard can coexist with an agent pipeline that has been silently producing malformed outputs for hours, because a prompt assumption broke after a model update, and nothing in the infrastructure metrics reflects that.* The infrastructure skills transfer. The instrumentation targets change substantially.

Alerting strategy also needs to be rethought. For traditional ML serving, infrastructure alarms were usually sufficient to catch failures worth paging on. For LLM systems, structured output parse failure rate, tool call error rate, and task completion rate are often better leading indicators of user-visible degradation than any infrastructure metric. The teams that instrument these signals before a production incident are in a significantly better position than the ones who add them afterward.

---

## Cost Management

**In traditional ML,** GPU cost was primarily a provisioning and utilization problem. You sized clusters to handle peak load, managed utilization to avoid wasting reserved capacity, and the cost profile across a billing period was predictable enough to budget against. Surprises usually came from infrastructure decisions, like choosing the wrong instance type or over-provisioning for a workload that turned out to be smaller than expected. The unit economics were relatively easy to reason about: cost per training run, cost per serving hour, cost per prediction at a given throughput.

Optimization levers were well-understood. Spot instances for training, reserved capacity for serving, bin-packing to maximize GPU utilization, mixed-precision inference to reduce memory footprint. Each of these had measurable impact on cost that could be estimated in advance and verified afterward.

**With LLMs,** cost is often proportional to token volume per request, which moves a significant portion of the cost management problem upward into the application layer. Input token cost and output token cost are billed separately in most managed APIs, and output tokens are typically more expensive because they require sequential decode computation. A request that generates a long response costs materially more than a structurally identical request that generates a short one, even if both arrive with the same input.

This has several practical consequences. First, output length constraints become infrastructure decisions, not application afterthoughts. Setting a maximum token limit is straightforward, but setting one that does not truncate valid completions while still bounding runaway outputs requires understanding the distribution of expected response lengths for each task type. Instructions in the prompt that ask the model to be concise have measurable cost impact when the model follows them.

Second, model selection based on task complexity is a cost lever that did not exist in traditional ML. A capable small model that handles routing, classification, or short-form extraction tasks correctly is substantially cheaper per token than a frontier model. Building a system that routes requests to an appropriately sized model based on task type, rather than sending everything to the most capable model available, can reduce serving cost significantly without degrading output quality on tasks where the smaller model is sufficient.

Third, a single misbehaving component in an agent system can produce cost spikes that no infrastructure alarm catches, because from the infrastructure's perspective everything looks normal. An agent stuck in a retry loop, a tool that returns an error that causes repeated reattempts, or a prompt that systematically elicits longer responses than necessary will all show up as elevated token volume rather than as an error rate. *Token budgets, retry limits, and response length monitoring are necessary cost controls in LLM systems.* They have no direct equivalent in traditional ML serving.

For self-hosted deployments using models like DeepSeek or Llama variants running on vLLM, the cost structure shifts back toward the traditional GPU provisioning model, but with added complexity. Prompt caching, which reuses the KV cache for repeated prompt prefixes across requests, can substantially reduce effective compute per request for applications where a long system prompt or few-shot context is shared across many users. Structuring prompts to maximize cache hits is a meaningful optimization that requires coordination between the application and infrastructure layers.

---

## Failure Propagation

**In traditional ML,** failures are loud. A preprocessing step errors out, a training run crashes, or a serving endpoint returns a `500`. The failure propagates through code, producing stack traces, log entries, and metric anomalies you can actually follow. Even silent failures, the ones where the model produces wrong predictions without raising exceptions, tend to surface quickly in offline evaluation if the evaluation is well-designed, because the output of a classifier is a value that can be compared against ground truth mechanically.

The operational response was straightforward in principle: instrument the likely failure points, define what a bad output looks like, and alert when failure rates exceed thresholds. Dead letter queues captured messages that could not be processed. Retry logic handled transient errors. Circuit breakers prevented cascading failures downstream. These patterns were well-established and the failure modes they addressed were reasonably predictable.

**With LLMs,** a model can generate a plausible-looking but incorrect tool call. The tool executes, returns a result, and the agent continues reasoning on bad data, producing confident output several steps later that is wrong in a way that only makes sense when you trace back through the full execution chain. No exception was raised. Nothing in the infrastructure metrics changed. The failure propagated through reasoning steps, not code steps.

This failure mode is qualitatively different from anything in traditional ML systems, and it requires different mitigations. Structured output schemas enforced at the API level, rather than validated after the fact, catch a class of malformation errors before they enter the reasoning chain. Output validation layers between agent steps can verify that intermediate results meet basic sanity criteria before they are passed to the next step. These are not perfect defenses, but they reduce the distance between a failure and its detection.

The compounding nature of reasoning chain failures makes early detection especially important. An error in step two of a six-step agent process does not just affect step two. It biases all subsequent reasoning steps, because the model is operating on a corrupted context that it has no way to identify as corrupted. By the time the output arrives at the user or the downstream system, the original error may be several levels of inference removed from anything visible. The longer the reasoning chain, the more a small early error can diverge from the correct answer by the end.

Logging and replay infrastructure matters more here than in traditional ML. When an agent produces a wrong result, the ability to replay the exact execution with the same inputs, model version, and tool responses is the difference between diagnosing the failure in minutes and spending hours reconstructing what happened. Full execution traces, including every model call, every tool input and output, and every intermediate state, should be stored for any agent system where failures have real consequences.

Human review gates remain the most reliable mitigation for high-stakes decisions in agent systems, the same role that human-in-the-loop validation plays in traditional ML pipelines where automated evaluation is insufficient. The instinct to distrust silent success, built from years of watching pipelines complete without errors but produce wrong results, is exactly right. The layer where that distrust needs to be applied has shifted upward, from data and infrastructure into the model's reasoning chain.

{% include architecture-diagram.html %}

---

## What Transfers and What Does Not

Most foundational disciplines transfer cleanly: reproducible pipelines, proper observability, cost-aware capacity planning, async infrastructure designed for failure, and the general disposition that AI systems fail in non-obvious ways that automated metrics often do not catch immediately.

The mental model for where failures occur is what does not transfer automatically. Traditional ML engineers watch data quality, feature distributions, model drift, and infrastructure health. LLM systems need all of that, plus application-layer correctness monitoring that the current tooling is still catching up to. Infrastructure health and model behavioral health are decoupled in LLM systems in a way they largely were not in traditional ML. A perfectly healthy serving cluster can be running a prompt that has drifted into producing wrong answers at a high rate, and nothing in the infrastructure layer will reflect that.

The other thing that does not transfer is the assumption that evaluation is primarily an offline activity. In traditional ML, evaluation happened before deployment and was updated periodically. In LLM systems, prompts, models, and tool configurations change more frequently, often without the ceremony of a full deployment, and the effects are harder to verify without running actual completions. Continuous evaluation, running a representative set of test cases against the live system on a regular schedule, is closer to the right model than periodic offline benchmarking.

Engineers coming to LLMs from web development backgrounds are often rediscovering problems the ML community worked through years ago, around reproducibility, evaluation methodology, and silent degradation. Engineers coming from traditional ML backgrounds have a meaningful head start on the infrastructure and reliability side. The open question for most teams is whether they can bring both disciplines to bear at the same time, which requires organizational structure as much as technical knowledge.

The fundamental lesson from a decade of ML infrastructure holds: the model is rarely the hardest part. Building the systems around it that make it run reliably, cheaply, and observably is where most of the engineering effort actually goes.
