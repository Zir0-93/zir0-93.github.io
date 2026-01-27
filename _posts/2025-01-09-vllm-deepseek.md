---
title: "Deploying DeepSeek-R1 with vLLM on KubeAI"
date: 2025-01-09 15:04:23
og_image: /images/microservices.png
tags: [mlops, LLMOps, gitops, kubernetes, vLLM, kubeai]
description: Deploying large language models in production requires efficient GPU utilization, predictable latency, and a clean operational model. This post shows how to serve DeepSeek-R1 using vLLM through KubeAI on Kubernetes.
excerpt_separator: <!--more-->
---

Large language models (LLMs) place heavy demands on infrastructure. Beyond raw GPU capacity, production deployments need controlled startup behavior, efficient request batching, and a stable API surface that clients can depend on.
<!--more-->

In this post, I’ll walk through deploying DeepSeek-R1 using vLLM as the inference engine, with KubeAI managing model lifecycle, scaling, and API exposure. Rather than deploying vLLM pods directly, we rely on KubeAI’s Model abstraction to keep the setup declarative and operationally simple.

A reference repository is available here: https://github.com/hadii-tech/vllm-mlops

## Why vLLM on KubeAI?

vLLM is a high-performance inference engine designed specifically for large language models. It focuses on GPU efficiency and predictable latency under load, which makes it a strong backend choice for models like DeepSeek-R1.

KubeAI sits one layer above the inference engine and provides:

- A Kubernetes-native Model API for declaring models  
- Automated backend orchestration (vLLM, TGI, etc.)  
- Scale-from-zero with request buffering  
- Centralized, OpenAI-compatible API routing  
- Smarter load balancing for stateful LLM backends  

The result is a clean separation of concerns: vLLM handles inference, while KubeAI handles operations.

## Installing KubeAI

KubeAI is installed using Helm.

Add the Helm repository:
```
$ helm repo add kubeai https://www.kubeai.org  
$ helm repo update  
```
Install KubeAI into the cluster:
```
$ helm install kubeai kubeai/kubeai --wait --timeout 10m  
```
After installation, the `kubeai` Service becomes the main entry point for OpenAI-compatible requests.

## Defining the DeepSeek-R1 Model

In KubeAI, models are defined using a Model custom resource. This describes what to run, not how to deploy it.

Create a file named `deepseek-r1-model.yaml` with the following contents:
```
apiVersion: kubeai.org/v1  
kind: Model  
metadata:  
  name: deepseek-r1  
spec:  
  features:  
    - TextGeneration  
  url: hf://deepseek-ai/DeepSeek-R1  
  engine: VLLM  
  args:  
    - --dtype=float16  
    - --max-model-len=32768  
    - --max-num-batched-tokens=8192  
    - --gpu-memory-utilization=0.90  
    - --disable-log-requests  
  resourceProfile: nvidia-gpu-h100:1  
  # minReplicas: 1  
```
Apply the model:
```
$ kubectl apply -f deepseek-r1-model.yaml  
$ kubectl get models  
$ kubectl get pods --watch  
```
KubeAI will pull the model, cache it, and launch vLLM backend pods as needed. The model name `deepseek-r1` becomes the identifier used in OpenAI-compatible requests.

## How Requests Flow Through KubeAI

When a request arrives:

1. Traffic hits the KubeAI OpenAI-compatible proxy  
2. KubeAI selects or spins up a backend for the target model  
3. Requests are queued while the backend initializes  
4. The request is forwarded to vLLM  
5. Responses are streamed back through the proxy  

This avoids sending traffic to cold pods and helps preserve KV cache locality.

## Exposing the OpenAI-Compatible API

Inside the cluster, the base API URL is:

`http://kubeai/openai/v1`

To expose it externally, create an Ingress that routes traffic to the `kubeai` Service:
```
apiVersion: networking.k8s.io/v1  
kind: Ingress  
metadata:  
  name: kubeai-ingress  
spec:  
  rules:  
    - host: llm.example.com  
      http:  
        paths:  
          - path: /openai  
            pathType: Prefix  
            backend:  
              service:  
                name: kubeai  
                port:  
                  number: 80  
```
After DNS is configured, the external base URL becomes:

https://llm.example.com/openai/v1  

## Testing the Deployment

List models:
```
$ curl https://llm.example.com/openai/v1/models  
```
Send a chat completion request:
```
$ curl https://llm.example.com/openai/v1/chat/completions  
  -H "Content-Type: application/json"  
  -d '{"model":"deepseek-r1","messages":[{"role":"system","content":"You are a helpful assistant."},{"role":"user","content":"Write a haiku about Kubernetes autoscaling."}]}'  
```
## Precision and Resource Profiles

DeepSeek-R1 is large enough that GPU choice and precision settings have a noticeable impact.

FP16 is a solid default on A100 and H100-class GPUs.  
BF16 can be preferable on some newer hardware.  
8-bit and 4-bit quantization can significantly reduce VRAM usage but should be validated carefully.

KubeAI’s resourceProfile abstraction keeps GPU sizing explicit and decoupled from model configuration.

## Final Thoughts

Using vLLM directly gives you raw performance. Using vLLM through KubeAI gives you that performance with a cleaner operational model. You declare the model once, and KubeAI handles backend orchestration, scaling behavior, and OpenAI-compatible routing.

For teams serving large language models in production, this combination provides a practical balance between control and simplicity—without building an entire inference platform from scratch.
