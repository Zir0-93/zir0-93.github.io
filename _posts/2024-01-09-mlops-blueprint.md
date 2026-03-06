---
title: "Designing a Production MLOps Pipeline: The Decisions That Actually Determine Reliability"
date: 2024-01-09 15:04:23
og_image: /images/microservices.png
tags: [mlops, gitops]
description: This post distills a reference architecture we built to address that. The context is a pipeline for categorizing and analyzing GitHub pull requests using a PyTorch autoencoder, developed as a reusable blueprint based on what we have seen fail repeatedly in production ML systems.
excerpt_separator: <!--more-->
---

Across client engagements at HADI Technology, one pattern shows up consistently: teams invest heavily in model development and then ship the surrounding infrastructure as an afterthought. The ML code works. The pipeline that runs it in production slowly becomes a liability.

This post distills a reference architecture we built to address that. The context is a pipeline for categorizing and analyzing GitHub pull requests using a PyTorch autoencoder, developed as a reusable blueprint based on what we have seen fail repeatedly in production ML systems. The intent was not to solve a difficult modeling problem but to demonstrate the infrastructure patterns that determine whether a system holds up over time. The full implementation is available at [github.com/hadi-technology/mlops-blueprint](https://github.com/hadi-technology/mlops-blueprint).
<!--more-->

![MLOps Pipeline Architecture](/images/mlops-pipeline-diagram.svg)

---

## Why Anomaly Detection Before Classification

When labeled data is sparse, and it almost always is early in a project, reaching for a supervised classifier tends to produce a model that looks better in evaluation than it actually is. With pull request data, there is no reliable ground truth for "risky" or "anomalous" changes, so supervised classification would have required either expensive labeling or synthetic labels that introduce their own assumptions.

An autoencoder trained to reconstruct normal patterns lets the anomaly threshold emerge from reconstruction error rather than from an arbitrarily defined label. This framing also defers the harder modeling question (what exactly constitutes a problematic PR?) until there is enough operational data to answer it meaningfully. It is a more defensible starting point when the problem is still being understood.

The pipeline feeds the model PR metadata: additions, deletions, changed file counts, labels, author association, and related signals. From there, the question of whether a given PR is an outlier relative to normal project patterns becomes tractable without labeled training data.

---

## Secret Management: Why Vault and Not Environment Files

Every pipeline stage needs credentials: GitHub tokens for data ingestion, MongoDB URIs for storage, object storage keys for features, and MLflow credentials for experiment tracking. The temptation in these situations is to reach for `.env` files or inject secrets as CI environment variables.

The problem with that approach is not just security. It is operational fragility. Secrets baked into environment files or injected through CI variables are painful to rotate, difficult to audit, and tend to proliferate across container images and build logs in ways that are hard to fully trace. In client environments like those at SkanAI, where infrastructure serves major financial institutions, this kind of loose secret handling is simply not acceptable.

We standardized on Vault for all secrets. Each container receives only three environment variables at startup: `VAULT_ADDR`, `VAULT_ROLE`, and `VAULT_SECRET_PATH`. Actual credentials are fetched at runtime using the `hvac` Python client. Nothing sensitive ever lands in Git or in a container layer.

The architectural consequence is that secret rotation requires no code changes and no image rebuilds. Rotating a GitHub token or MongoDB credential is a Vault operation, not a deployment event. Over time, this matters more than it seems upfront.

---

## Preventing Environment Drift: The Structural Choice

Staging and production diverge in subtle ways when there is no deliberate architectural constraint preventing it. A one-off config change made directly to the staging cluster, a dependency pinned differently, a Kubernetes resource limit adjusted during debugging: over weeks and months, these accumulate into a staging environment that does not meaningfully predict production behavior.

We address this structurally rather than procedurally across most client engagements. The pattern is two separate Kubernetes clusters (one for staging, one for production), sharing a single set of base Kubernetes manifests. Argo CD in each cluster points to the same base manifests from different Git branches: `staging` tracks the staging branch, production tracks `main`. Promotion is a merge, not a redeployment.

The key property this creates is that you cannot accidentally promote without going through version control. There is no manual `kubectl apply` path to production. Drift has to be introduced deliberately, through code, which means it is visible and reviewable.

This is worth distinguishing from overlay-heavy setups where staging and production have separate Kustomize overlays that diverge over time. Overlays are useful for environment-specific configuration, but they can become a source of drift if the base is not kept genuinely shared. The goal is that staging and production are running the same manifests with minimal parameterized differences, not parallel configurations that happen to look similar.

---

## Single-Build Artifact Strategy

Building separate container images for staging and production is one of those practices that feels harmless until it causes a failure that takes two days to trace. The issue is not that the builds are intentionally different. It is that dependency resolution, base image layers, and build context can vary in ways that are difficult to detect until something breaks in production that worked in staging.

The pipeline uses a single-build approach across all stages. An image is built once, tested ephemerally against the same image that was just built, deployed to staging, and after validation promoted to production using the same digest. There is no second build.

The CI workflow pattern is consistent across all stages:

1. Build the image on code change
2. Run ephemeral container tests against the freshly built image
3. Deploy to staging
4. Promote to production via branch merge, using the same image tag

The practical effect is that if something passes staging, it will not fail in production due to a build artifact difference. This sounds obvious, but the number of production incidents that trace back to "worked in staging" is high enough that it warrants an architectural constraint rather than a best-effort practice.

---

## Blue/Green Rollouts for Model Serving

Model updates are different from regular service deployments in one important way: the rollout risk is asymmetric. A new model that scores poorly under production traffic has already affected real users by the time it is caught. Rolling back after the fact is costlier than never having routed traffic to it in the first place.

We use Argo Rollouts with a blue/green strategy for the inference serving component. New model versions are deployed alongside the active version. Traffic continues flowing to the current version while the new version handles requests through a preview service. Load tests run against the preview, and only after those pass does traffic cut over. Rollback is instantaneous because the previous version is still running.

The cost of this approach is resource usage: you are briefly running two full serving stacks. For a production system handling real traffic at scale, this is an acceptable trade. For a smaller experiment, it might feel like over-engineering. Our position, shaped by what we have seen happen when teams skip this pattern, is that it is better to practice these rollout patterns on small systems than to discover their importance for the first time on a system that cannot afford downtime.

The alternative, a rolling update that replaces pods incrementally, is fine for stateless services where any request can be handled by any pod. It is less appropriate for model serving, where the inference behavior changes discretely between versions and you want a clear before/after boundary rather than a window where both old and new model versions are simultaneously handling traffic.

---

## Data Processing and Experiment Tracking

Spark is often positioned as a big data tool, which leads teams to dismiss it for smaller workloads. The more useful framing is that Spark handles large-scale feature transformations reliably and without requiring you to manage memory carefully in a single Python process. For preprocessing PR data at any meaningful scale, a single-process approach will eventually hit a wall.

MLflow's value in this context is not so much in the model training itself. It is in the operational discipline it enforces. Every training run is logged. Metrics are tracked. Model artifacts are versioned. When a new run underperforms, there is a clear comparison point rather than a vague memory of what parameters were used last time.

Locust load-tests the inference endpoint before rollout, which bridges the gap between offline model evaluation and online system behavior. A model that scores well on reconstruction error but causes the serving container to OOM under concurrent requests is still a broken deployment. Load testing before promotion catches this class of failure.

---

## Cost and Operational Risk as Design Inputs

Something absent from most MLOps writing is an honest discussion of how cost and operational risk should shape architectural decisions from the start, not as afterthoughts.

The single-build approach reduces rollout risk. The shared manifest base reduces the cost of maintaining two environments. The Vault integration reduces the operational overhead of secret rotation. Blue/green rollouts reduce the cost of a bad model deployment. These are not independent technical choices. They are a coordinated set of decisions that collectively reduce the total cost of operating the system over time.

In client work through HADI Technology, the framing we bring to these decisions is: what is the ongoing cost of operating this the wrong way? The answer usually justifies investing in the right patterns upfront. A secret management scheme that requires image rebuilds to rotate credentials looks cheap initially and expensive at the worst possible moment.

---

## Closing Thoughts

The PR categorization experiment is a relatively small ML use case. The infrastructure around it is not. That gap is intentional: the value of working through these design decisions on a contained problem is that the patterns are ready when the problem gets larger.

Promotion to production is a single merge. Secret rotation does not require rebuilds. New models can be shipped without downtime. These properties did not emerge from the tools. They came from the decisions made before the tools were chosen.

The full implementation is at [github.com/hadi-technology/mlops-blueprint](https://github.com/hadi-technology/mlops-blueprint).
