--- 
title:  "Building a Modern MLOps Pipeline: Lessons Learned from Categorizing Pull Requests"
date:  2024-01-09 15:04:23
og_image: /images/microservices.png
tags: [mlops, gitops]
description: Think back to the last time you worked in a distributed system, did you consider using something other than RESTful HTTP calls as the method of communication between components in this system?
excerpt_separator: <!--more-->
---

My team recently embarked on a quest to **categorize and analyze GitHub pull requests** using a deep-learning model. This might sound straightforward—fetch PR data, preprocess it, train a model, then serve predictions. But like many ML projects, it quickly became evident that **infrastructure** and **operational concerns** could dwarf the modeling tasks if not handled carefully.<!--more-->


My team recently embarked on a quest to **categorize and analyze GitHub pull requests** using a deep-learning model. This might sound straightforward—fetch PR data, preprocess it, train a model, then serve predictions. But like many ML projects, it quickly became evident that **infrastructure** and **operational concerns** could dwarf the modeling tasks if not handled carefully.

In this post, I’ll share how we designed our MLOps pipeline to address **common pitfalls**—secret handling, environment drift, clumsy CI/CD, downtime risks for model updates, and large data transformations. Along the way, I’ll reference the [**GitHub repository**](#) we created (insert link to your repo) so you can see **implementation details** for each stage.

---

## Our Experiment: Categorizing Pull Requests

We set out to **label** or **categorize** GitHub pull requests (PRs) based on metadata like additions, deletions, changed files, labels, author association, and more. The idea was to detect anomalies or patterns in PRs—think: “Is this PR likely to be high risk, or is it fairly standard?”

The **pipeline** ended up looking like this:

1. **Data Fetch**: Gathers raw PR metadata (additions, deletions, changed files, etc.) from GitHub, storing them in MongoDB.  
2. **Spark Preprocess**: Runs a Spark job to transform and fill missing data, then writes the resulting features to DigitalOcean Spaces.  
3. **ML Training**: Trains an autoencoder (or other deep-learning approach) on the preprocessed data, logs metrics to MLflow.  
4. **Model Serving**: Deploys a container that provides predictions (“What’s the anomaly score of this PR?”) behind a web endpoint.

As you can guess, each of these steps can get complicated. Below, I’ll highlight the **pain points** we encountered and the **design principles** we used to tame them.

---

## Pain Point #1: Secure Secrets for Multi-Stage Pipelines

We first realized that each stage needed different credentials—GitHub tokens for data fetch, DO Spaces keys for Spark, Mongo URIs for storing/fetching data, MLflow credentials for training logs, etc. Hard-coding them in environment variables (or the repo) would be a security nightmare. People typically do this until they inadvertently leak a password on GitHub, then vow never again.

**What We Did**  
- Adopted **Vault** to store and manage all secrets.  
- Each pipeline stage references only a few environment variables like `VAULT_ADDR`, `VAULT_ROLE`, and `VAULT_SECRET_PATH`.  
- The actual tokens/URIs remain inside Vault. Our containers pull them **at runtime** using the [**hvac** Python library](#).  
- In the [**GitHub repo**](#), you’ll see that every Dockerfile-based stage uses those `VAULT_*` environment variables, letting the code authenticate securely.

**Why It Helped**  
- We never commit sensitive data.  
- We can rotate secrets in Vault without updating code or environment files.  
- Each stage is a separate container, so each one references the portion of Vault secrets it needs, nothing more.

---

## Pain Point #2: Environment Drift

A big headache is keeping “staging” (or any test environment) aligned with “production.” Often, staging is only partially reflective of production’s code, data, or config, so something that works in staging fails in production because the build steps differ or environment variables changed. The result is confusion and unpredictable deployments.

**Our Approach**  
- Maintain **two** separate Kubernetes clusters: staging and production.  
- Use **two** main Git branches: `staging` and `main`.  
- Each cluster’s Argo CD references the same “base” K8s manifests but from a **different** branch.

If you look at our [**`argo-apps/base/` folder**](#) in the repo, you’ll see the single set of YAML definitions (Jobs for data fetch/preprocessing/training and a Rollout for model-serving). But staging references that folder in the `staging` branch, while production references the same folder in the `main` branch. When staging is validated, we simply merge `staging → main`—the simplest form of “promotion.”

**Why This Helps**  
- Staging is always a near clone of production’s config.  
- No complicated folder structure for overlays. Merges handle environment transitions more simply.

---

## Pain Point #3: Inconsistent Artifacts Across Environments

It’s common to build containers separately for staging and production. But if you do that, you can run into “Works in staging but not in production” issues. Why? The production build might yield a slightly different artifact or dependencies.

**What We Did**  
- **Single-build** ephemeral approach: For each pipeline stage, we build the Docker image once, ephemeral-test it, then push the same image to production after it’s validated in staging.  
- This ensures staging runs the exact container that will eventually run in production—no second build.

You can see this logic in our [**GitHub Actions workflows**](#) for data-fetch, spark-preprocess, ml-training, or model-serving. Each workflow:

1. **Builds** the image if code changed.  
2. **Runs ephemeral container tests** (like a local run that checks logs or an endpoint).  
3. On success, we **update** the staging environment references.  
4. Later, we merge staging → main with that same tag to move it into production.

**Why It Helps**  
- Consistency across environments.  
- Less “it worked in staging, not in production” debugging.  
- Encourages ephemeral container testing as a gate to staging promotion.

---

## Pain Point #4: Downtime During Model Upgrades

For real-time inference, downtime is a no-go. If you deploy a new model, you can’t just kill the old pod. That may cause dropped requests.

**Our Fix**  
- Use **Argo Rollouts** with **blue/green** strategy for the model-serving stage.  
- We define a “preview” service and an “active” service. We spin up new pods as the “blue” version while the “green” version remains live. Once tested, we flip traffic to the new version.

In the [**model-serving rollout YAML**](#) in our repo, you’ll see how the code references the same container image but performs a controlled switch from old to new. We also run **Locust** performance tests to ensure it meets latency requirements before finalizing the switchover.

**Why It Helps**  
- Minimal or zero downtime.  
- If the new model is broken, we can revert to the old version seamlessly.

---

## Pain Point #5: Handling Big Data, Tracking Metrics, and Validating Performance

Large datasets can’t be processed in a single Python script, so we rely on **Spark** to scale transformations. We wanted to track training metrics historically, so we integrated **MLflow**. And once our model-serving sees real usage, load testing with **Locust** is vital to gauge stability.

**Our Setup**  
- **Spark** for heavy-lifting in `spark-preprocess.py`, writing features to DigitalOcean Spaces.  
- **MLflow** to log training metrics for each run (like F1, AUC, or reconstruction error).  
- **Locust** to load-test the inference endpoint for each new serving container.

**Why It Helps**  
- Spark can handle large volumes with minimal re-architecture.  
- MLflow provides a single place to compare training runs.  
- Locust ensures new model or container versions don’t cause regressions under load.

---

## Conclusion: A Cohesive MLOps Blueprint

All of these design decisions—**Vault** for secrets, **two-branch environment separation**, **single-build ephemeral container tests**, **blue/green** rollouts for model-serving, plus synergy with **Spark** and **MLflow**—combine to address the *biggest* pain points that MLOps teams typically face. 

By applying them to our **PR-categorization** experiment, we’ve built a reliable pipeline. If you want to see exactly how we did it, check out our [**GitHub repository**](#) where we have Dockerfiles, the CI/CD workflows, Argo manifests, and ephemeral container test scripts. We hope these principles inspire you to approach MLOps with a focus on **security**, **consistency**, **low downtime**, and **scalability**—making it easier to keep track of complex ML projects while shipping models confidently to production.
