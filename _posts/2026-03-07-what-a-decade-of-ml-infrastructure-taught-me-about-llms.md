---
title: "What a decade of ML infrastructure did, and did not, prepare me for with LLMs"
date: 2026-03-07 00:01:00
og_image: /images/og-green-dashboard.png
cover: /images/cover-green-dashboard.png
tags: [llms, mlops, observability, platform engineering, system design]
toc: true
description: "Two production incidents where every infrastructure metric stayed healthy while the model layer quietly failed, and what they showed about which classical ML habits transfer to LLM systems and which do not."
excerpt_separator: <!--more-->
---

I have worked on ML infrastructure for close to a decade: migrating pipelines into cloud environments, wiring together GPU clusters, making autoscaling behave for workloads that spiked without warning. The models were rarely the hard part. Running them reliably, cheaply and observably was. When the work shifted into LLM systems, serving open models on vLLM and operating agent pipelines under real load, most of that experience carried over. Two incidents taught me where it did not.

<!--more-->

![Every infrastructure metric healthy while the agent's outputs were malformed](/images/hero-green-dashboard.svg){: .light-border }
*The infrastructure layer and the model layer fail independently. A dashboard built for the first says nothing about the second.*

## The parser noticed before the model layer did

On one engagement the model provider shipped a silent update overnight. Nothing in our system changed. The model's tool-call formatting did, and the first signal we got was the error rate of a downstream parser climbing. Nothing at the model layer reported anything, because from the model layer's point of view nothing was wrong: requests went out, completions came back, latency was normal, no exception was raised anywhere near the model.

{% comment %}PLACEHOLDER, verify against notes before publishing{% endcomment %}
The errors ran for most of a working day before anyone looked. The alert that finally fired was the parser's own threshold, set at one in twenty requests failing to parse, and it fired hours after the rate had crossed it because the threshold had been chosen for a parser bug, not for a model change.

We found it by reading the parser's failures and working backwards. The fix was small: pin the model to a specific version and upgrade deliberately, with evaluations, the same way we had pinned library versions in every production system for years. The lesson was that a hosted model is a dependency, and an unpinned dependency that changes under you is a class of failure classical ML had already taught me to fear. I had just not filed the model under "dependency".

## Hours of malformed output under a healthy dashboard

The second incident was on a project with a good Prometheus setup. GPU utilisation, memory pressure, queue depth, request rate, latency percentiles, error rate, pod restarts: every panel was green. Underneath it an agent pipeline had been producing malformed output for hours, because a prompt assumption stopped holding after a model update, and not one infrastructure metric moved.

{% comment %}PLACEHOLDER, verify against notes before publishing{% endcomment %}
It was noticed when a downstream report came out empty and someone read the raw outputs behind it. By then a few thousand requests had gone through, every one of them accepted by the pipeline as a success.

That one changed what I monitor. The infrastructure metrics were still right and still necessary. They answered the question they were built for, whether the serving layer was healthy, and it was. The question nobody had instrumented was whether the model was doing what the application needed. Those two questions had been the same question for most of my career, because a classifier that returns a value is either up or it is not. For an agent they are decoupled, and a healthy cluster can run a broken prompt indefinitely.

What we added, and what I now add by default on any LLM system:

- The rate at which structured outputs parse against their expected schema.
- The rate at which tool calls are well formed and return the result the next step expects.
- The distribution of response lengths over time, since a shift there is often the first sign a prompt has drifted.
- Task completion rate, where the task has a definition that can be checked.

These live at the application layer. They need a decision, before the system ships, about what correct means for each use case. And they page. In both incidents, an alert on parse-failure rate would have fired before anyone was looking, and no infrastructure alarm ever would have.

## Failures now compound through reasoning, not code

Both incidents have the same shape, and it is the shape that distinguishes LLM systems from the ML systems I ran before. In classical ML, failures are loud. A preprocessing step errors out, a training run crashes, an endpoint returns a `500`, and the failure leaves stack traces, log lines and metric anomalies you can follow. Even the quiet failures, wrong predictions with no exception, surface in offline evaluation because a classifier's output can be compared with ground truth mechanically.

An agent's failures are quiet by construction. The model generates a plausible but wrong tool call, the tool executes, and the agent keeps reasoning on bad data. Several steps later it produces confident output that is wrong in a way that only makes sense once you trace the full chain. An error in step two of a six-step process biases every step after it, because the model is working from a corrupted context it cannot recognise as corrupted.

Three mitigations have earned their place:

- Structured output schemas enforced at the API boundary, so malformed output is rejected before it enters the reasoning chain rather than validated after the fact.
- Validation between agent steps, so an intermediate result has to meet basic sanity checks before it is handed on.
- Full execution traces for any agent whose failures have consequences: every model call, every tool input and output, every intermediate state. Without them, debugging a chain is guesswork.

Human review still sits above all three for high-stakes decisions. The instinct to distrust silent success, built from years of watching pipelines finish cleanly and produce the wrong answer, is the right instinct. It just has to be applied one layer higher than it used to be.

## Evaluation: golden transcripts, re-run on every prompt change

The received wisdom is continuous evaluation, a representative test set run against the live system on a schedule. I think that is mostly theatre below a certain team size, and I would rather say what I actually do.

For a small team, evaluation is a handful of golden transcripts, re-run on every prompt change and every model version bump, with the diff read by a person. A prompt is a functional component, and a change to it can alter behaviour as much as a change to a preprocessing function would have. So prompts live in the same repository as the code that calls them, changes are attributed, and the transcripts run before merge, the way unit tests do. Because production usually needs some sampling variability, each transcript is run more than once and the results aggregated rather than treating one sample as the answer.

{% comment %}PLACEHOLDER, verify against notes before publishing{% endcomment %}
On the systems I run today that is about forty transcripts per system, and the diff turns up something worth reading on roughly one prompt change in five.

That is less than a benchmark suite. It is also honest about what a small team will maintain, and it catches the two failure modes above, because both were a prompt or model change that would have shown up in a re-run transcript before it shipped.

## The parts that transfer without drama

Most of the rest of classical ML infrastructure applies to LLM systems with small adjustments, and a short list is enough.

Latency splits into two measurements. Time to first token is what an interactive user perceives; total completion time is what throughput and cost accounting need. Output length is decided at runtime, so two requests that look identical at ingress can differ by two orders of magnitude in tokens produced. Prefill is parallel and quick; decode is memory-bandwidth bound and runs one token per step. Serving engines batch decode steps across requests (that is what continuous batching in vLLM does), but per-request utilisation still looks nothing like a classifier's, so autoscaling on GPU utilisation thresholds needs rethinking. The habit of tracking p95 and p99 separately from p50 matters as much as ever; it now applies to first-token time and time per output token as well as to total duration. Sequence models and speech systems had streaming latency metrics long before LLMs, so this is a return to an older discipline rather than a new one.

Reproducibility has a larger surface. Classical pipelines drifted on code, data and environment. LLM systems also drift on the prompt, the model version, the context-window handling, the tool definitions, the behaviour of the external APIs those tools call, and the sampling parameters. Any of them can change between runs and produce different output without an exception. Pinned environments, versioned data with lineage and logged parameters remain the survival kit; prompt snapshots and model versions join the list of artifacts. The [mlops-blueprint](https://github.com/hadi-technology/mlops-blueprint) is my attempt to write that end-to-end discipline down, and the additions for LLM pipelines are small.

Cost moves up into the application layer, because it is proportional to token volume per request. Output tokens cost more than input tokens because each one is a separate decode step that reads the whole KV cache, which is the memory-bandwidth cost from the latency section showing up on the bill. Three consequences follow. A maximum output length is an infrastructure decision, and setting one that bounds runaway output without truncating valid completions means knowing the response-length distribution per task. Routing simple tasks to a smaller model is a cost lever classical ML never had. And an agent stuck in a retry loop shows up as token volume, not as an error rate, so no infrastructure alarm catches it. For self-hosted models on vLLM the cost shape returns to GPU provisioning, and prefix caching, which reuses the KV cache for a shared system prompt across requests, is the lever that changes the arithmetic.

## What this looks like in what I build now

The posture from both incidents, that the model may propose and a program must verify, is the design of [Striff](https://striff.io), the tool I work on today. It reads a repository's architecture documents, has a language model translate their sentences into rules, and then checks those rules against the parsed code with no model involved. The model never decides whether a rule holds, a silent "could not check" is never recorded as "checked, nothing found", and every finding names the sentence and the line it came from. That is the parser-error-rate lesson and the green-dashboard lesson, built into the shape of a product rather than added to it afterwards. I wrote up how it works in [a post on the pipeline]({% post_url 2026-04-28-striff-io-ml-infrastructure %}).

## Code referenced in this post

<div class="post-badges">
<a href="https://github.com/hadi-technology/vllm-mlops"><img src="https://img.shields.io/badge/vLLM%20MLOps-vllm--mlops-blue?logo=github" alt="vllm-mlops"></a>
<a href="https://github.com/hadi-technology/mlops-blueprint"><img src="https://img.shields.io/badge/MLOps%20Pipeline-mlops--blueprint-teal?logo=github" alt="mlops-blueprint"></a>
</div>
