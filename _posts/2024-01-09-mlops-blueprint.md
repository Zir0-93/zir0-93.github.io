---
title: "Building a Modern MLOps Pipeline: Lessons Learned from Categorizing Pull Requests"
date: 2024-01-09 15:04:23
og_image: /images/microservices.png
tags: [mlops, gitops]
description: Lessons learned building a production MLOps pipeline for categorizing GitHub pull requests, covering secrets, CI/CD, model rollout, and scalable data processing.
excerpt_separator: <!--more-->
---

My team recently set out to categorize and analyze GitHub pull requests using a deep learning model. On paper, the task sounded straightforward: fetch PR data, preprocess it, train a model, and serve predictions.

In practice, the modeling itself was the easy part.

What quickly dominated the project were infrastructure and operational concerns—secret management, environment drift, brittle CI/CD workflows, rollout risk during model updates, and handling large data transformations. Left unchecked, these issues can easily overwhelm an otherwise simple ML use case.  
<!--more-->

In this post, I’ll walk through how we designed our MLOps pipeline to address those problems head-on. The full implementation—Dockerfiles, CI workflows, Argo manifests, and test scripts—is available in the GitHub repository:  
https://github.com/hadii-tech/mlops-blueprint

---

## The Experiment: Categorizing Pull Requests

The goal was to label GitHub pull requests based on metadata such as additions, deletions, changed files, labels, author association, and similar signals. From there, we could start asking higher-level questions: Is this PR unusually risky? Does it look like an outlier compared to the project’s normal change patterns?

We started with an autoencoder because labeled data was limited early on, and anomaly detection was a better fit than supervised classification for the initial problem.

At a high level, the pipeline looked like this:

1. Data fetch: pull raw PR metadata from GitHub and store it in MongoDB  
2. Preprocessing: use Spark to clean, normalize, and enrich the data, then write features to object storage  
3. Model training: train the model and log metrics to MLflow  
4. Model serving: deploy a container that exposes an endpoint for anomaly scoring  

Each step was manageable on its own. The real challenge was making the whole system reliable, repeatable, and safe to operate across environments.

---

## Managing Secrets Across Pipeline Stages

Every stage required different credentials: GitHub tokens, MongoDB URIs, object storage keys, and MLflow credentials. Hard-coding these values—whether in the repository or environment files—was never an option.

We standardized on Vault for all secrets.

Each container only receives a small set of Vault-related environment variables, such as VAULT_ADDR, VAULT_ROLE, and VAULT_SECRET_PATH. The actual credentials live entirely inside Vault and are fetched at runtime using the hvac Python client.

This approach gave us a few immediate benefits:

- Sensitive values never land in Git  
- Secrets can be rotated without code or image changes  
- Each stage only has access to what it actually needs  

If you browse the repository, you’ll see this pattern applied consistently across all Dockerfile-based stages.

---

## Keeping Staging and Production Aligned

One of the most common failure modes in MLOps is environment drift. Staging slowly diverges from production, and deployments become unpredictable as a result.

To avoid that, we made a few deliberate choices:

- Two Kubernetes clusters: one for staging, one for production  
- Two primary Git branches: staging and main  
- A single set of base Kubernetes manifests shared by both environments  

Argo CD in each cluster points to the same base manifests, but from different branches. Staging tracks the staging branch, production tracks main. Promotion is just a merge from staging into main.

This keeps the environments closely aligned without introducing a complex overlay structure or duplicated configuration.

---

## Keeping Artifacts Identical Across Environments

Building separate container images for staging and production is a subtle but common source of bugs. Even small differences in build context or dependency resolution can lead to failures that only show up after promotion.

We avoided this by using a single-build approach.

Each pipeline stage builds its container image once. That exact image is tested ephemerally, deployed to staging, and—after validation—promoted to production using the same image tag. There is no second build.

Our CI workflows all follow the same pattern:

1. Build the image when code changes  
2. Run ephemeral container tests  
3. Deploy to staging  
4. Promote the same image to production via a branch merge  

This keeps artifacts consistent and significantly reduces works-in-staging-fails-in-prod debugging.

---

## Shipping New Models Without Downtime

For real-time inference, even brief downtime is unacceptable. Simply replacing a running pod during a model update risks dropped requests and degraded user experience.

We addressed this by using Argo Rollouts with a blue/green strategy for the model-serving component.

New model versions are deployed alongside the existing one. Traffic continues flowing to the active version while the new version is validated behind a preview service. Once it passes basic checks and load tests, traffic is switched over. If anything looks off, rollback is immediate.

This gave us confidence to deploy new models without fear of impacting live traffic.

---

## Scaling Data Processing and Validating Performance

Pull request data adds up quickly, and processing it in a single Python process doesn’t scale. We leaned on Spark for preprocessing and MLflow for experiment tracking.

The setup looks like this:

- Spark handles large-scale feature transformations and writes results to object storage  
- MLflow tracks training runs and metrics over time  
- Locust load-tests the inference endpoint before rollout  

Together, these tools give visibility into both offline model quality and online system behavior.

---

## Final Thoughts: A Practical MLOps Blueprint

None of these ideas are novel on their own. What mattered was combining them into a system that was simple to reason about and safe to operate:

- Vault for secrets  
- Branch-based environment promotion  
- Single-build container artifacts  
- Blue/green rollouts for model serving  
- Spark, MLflow, and load testing where they actually add value  

Applied to a relatively small experiment like PR categorization, this setup gave us a pipeline we could trust. Promotion to production is a single merge, secret rotation doesn’t require rebuilds, and new models can be shipped without downtime.

If you want to dig into the details, everything—from Dockerfiles to CI workflows and Argo manifests—is available in the repository linked above.
