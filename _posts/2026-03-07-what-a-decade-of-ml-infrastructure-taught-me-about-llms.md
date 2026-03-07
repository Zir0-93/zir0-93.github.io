---
title: "What a Decade of ML Infrastructure Taught Me About LLMs"
date: 2026-03-07 00:01:00
og_image: /images/microservices.png
tags: [ai, llms, mlops, platform engineering, system design]
toc: true
description: What carries over from classical ML infrastructure into LLM systems, where the operational problems get harder, and which new failure surfaces appear in production.
excerpt_separator: <!--more-->
---

I have spent close to a decade working on ML infrastructure, and by 2018, before MLOps was a job title most people recognized, the work already followed a familiar pattern: migrating pipelines into cloud environments, wiring together GPU clusters, and making autoscaling behave sensibly for workloads that spiked unpredictably. The models were rarely the hard part. Getting them to run reliably, cheaply, and observably in production was.

A few years ago I founded HADI Technology, a consultancy where that work continued across production AI platforms, model serving systems, and agent infrastructure. Over time, engagements shifted increasingly into LLM-based systems: serving open-source models on vLLM, building agent infrastructure, and operating AI-driven platforms at scale. What became clear through that transition is that the problems do not change as much as the current hype suggests. They evolve. They get harder in specific ways. But the years spent wrestling with traditional ML infrastructure turned out to be more relevant than I expected.

<!--more-->

{% include article-visual.html %}

Here is what transferred, and where the gaps are.

---

## Serving Latency

**In traditional ML,** latency was usually profileable. For a given model and input size, you could reason about throughput, queue depth, and resource saturation with reasonable confidence. At Buzz Indexes, designing autoscaling ECS clusters for GPU-intensive ML tasks meant developing precise instincts about when to scale out versus when to buffer. That reasoning still holds.

**With LLMs,** the variable that breaks those instincts is output length. It is determined at runtime, which means a batch of requests that looks uniform at ingress can produce wildly different completion times depending on how the model responds. SLA guarantees and timeout handling become considerably more nuanced than they were for a fixed-output classifier. *The habit of tracking p95 and p99 latency separately from p50, built over years of operating ML endpoints, turns out to matter even more, not less.*

## Pipeline Reproducibility

**In traditional ML,** one of the hardest lessons from retraining pipelines at BorealisAI was that environment drift is among the most common sources of silent failure. A pipeline that ran cleanly three months ago, run again today with subtly different library versions or data distributions, can produce a model that looks fine until it reaches production. Treating pipelines as reproducible artifacts, with pinned environments, versioned data, and logged parameters, was not a best practice. It was survival.

**With LLMs,** that discipline carries over directly, but the surface area for drift is considerably larger. In a traditional pipeline, the training code and data are the main variables. In an LLM-based system, the prompt, model version, context window handling, tool definitions, and the behavior of any external APIs those tools call are all variables too. *Any of them can shift between runs and produce meaningfully different outputs without raising a single exception.* The [mlops-blueprint](https://github.com/hadi-technology/mlops-blueprint) was an attempt to codify what end-to-end reproducibility looks like. The same thinking applies directly to LLM pipelines.

## Observability

**In traditional ML,** deploying a federated Prometheus environment at Teranet across OpenShift clusters made one thing clear: observability is not optional infrastructure. It is the difference between knowing your system is healthy and guessing. The metrics were relatively clean to instrument: request rate, latency, prediction distribution, error rate.

**With LLMs,** all of that is still necessary, plus a category of signals that did not exist before. Output quality metrics, whether the model is completing tasks, whether tool calls are structured correctly, and whether response length is drifting cannot be inferred from infrastructure metrics alone. *A healthy Prometheus dashboard can coexist with an agent pipeline that has been silently producing malformed outputs for hours because a prompt assumption broke after a model update.* The infrastructure skills transfer. The instrumentation targets change.

## Cost Management

**In traditional ML,** GPU cost was mostly a provisioning problem. You sized clusters, managed utilization, and the cost profile was relatively predictable over a billing cycle. Surprises usually came from infrastructure decisions.

**With LLMs,** cost is often proportional to token volume per request, which pushes the problem upward into the application layer. A single misbehaving agent that generates unnecessarily long outputs, gets stuck in a retry loop, or calls a tool repeatedly can spike costs in ways that no infrastructure alarm catches, because from the infrastructure's perspective everything looks normal. *Token budgets, output length constraints, and model selection based on task complexity are infrastructure decisions now, not application afterthoughts.*

## Failure Propagation

**In traditional ML,** failures are loud. A preprocessing step errors out, a training run crashes, or a serving endpoint returns a `500`. The failure propagates through code, producing traces you can actually follow.

**With LLMs,** a model can generate a plausible-looking but incorrect tool call. The tool executes, returns a result, and the agent continues reasoning on bad data, producing confident output several steps later that is wrong in a way that only makes sense when you trace back through the full execution chain. No exception was raised. Nothing in the infrastructure metrics changed. *The failure propagated through reasoning steps, not code steps.*

This is the sharpest way LLM systems amplify what was already difficult about traditional ML. The instinct to distrust silent success, built from years of watching pipelines complete without errors but produce wrong results, is exactly right. The layer where that distrust needs to be applied has shifted upward, from data and infrastructure into the model's reasoning chain. Checkpoints, structured output validation, and human review gates are the equivalent of dead letter queues and retry logic in any serious async pipeline. The [icp-collab](https://github.com/hadi-technology/icp-collab) framework was built with exactly this in mind.

{% include architecture-diagram.html %}

## What Transfers and What Does Not

Most foundational disciplines transfer cleanly: reproducible pipelines, proper observability, cost-aware capacity planning, async infrastructure designed for failure, and the general disposition that AI systems fail in non-obvious ways.

What does not transfer automatically is the mental model for *where* failures occur. Traditional ML engineers watch data quality, feature distributions, model drift, and infrastructure health. LLM systems need all of that plus application-layer correctness monitoring that the current tooling is still catching up to.

Engineers coming to LLMs from web development backgrounds are often rediscovering problems the ML community worked through years ago. Which is, in its own way, a reminder that fundamentals matter more than the current generation of tooling.
