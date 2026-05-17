---
title: "Detecting Architectural Anomalies in Code with Graph Neural Networks"
date: 2026-04-28 10:00:00
og_image: /images/gnn-pipeline-diagram.svg
tags: [gnn, code review, graph neural networks, software architecture, ml engineering]
toc: true
description: "How [striff.io](https://striff.io) uses a neurosymbolic pipeline with typed dependency graphs, Chidamber-Kemerer features, and an edge-prediction GNN to flag the dependencies in a pull request that carry architectural risk. The post covers the graph construction pipeline, the 405-dimensional feature vector spanning four languages, why we switched from node-level anomaly scoring to edge-prediction, six deterministic architectural detectors, and how symbolic facts are fused with learned anomaly scores to produce grounded LLM review annotations rendered directly on architecture diagrams."
excerpt_separator: <!--more-->
---



Code review has a specific information problem that most tooling ignores. When you open a pull request on a large codebase, the diff shows you *lines*. It does not show you that the class you just modified now has fourteen things depending on it when it had three last week. It does not show you that a new import three files away quietly created a dependency cycle between two packages that were previously clean. It does not show you that the abstraction Claude just extended sits at depth six in an inheritance tree that has been growing for two years.

These are not edge cases. They are the class of change that produces architectural debt, the kind that compounds quietly and becomes expensive to unwind.

That is the problem [striff.io](https://striff.io) was built to address. The product generates visual architecture diffs from GitHub pull requests: instead of a line diff, you get an SVG class diagram showing which components changed, how their relationships shifted, and which of those changes carry structural risk, annotated directly on the diagram.

The interesting engineering question is not the diagram rendering. It is *how you decide what to flag*. The approach we landed on is a staged neurosymbolic pipeline: extract a structural graph, compute deterministic symbolic facts, run a learned GNN anomaly scorer, and *only then* ask an LLM to annotate. The order is deliberate. The infrastructure that runs this pipeline at scale -- Kafka staging, Triton inference, degradation modes -- is covered in a [companion post]({% post_url 2026-04-28-striff-io-ml-infrastructure %}).

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

## What the Output Looks Like

Before explaining how the system works, here is what it produces. When you open a pull request on striff.io, you get an interactive architecture diagram with review annotations pinned to the components that carry structural risk:

<div id="striff-diagram" style="margin:24px auto;max-width:800px;border:1px solid rgba(15,23,42,0.12);border-radius:12px;overflow:hidden;background:#f8fafc;"></div>
<script src="https://d3js.org/d3.v7.min.js"></script>
<script>
(function(){
  var svg = d3.select("#striff-diagram").append("div").style("position","relative").style("width","100%").style("height","500px");
  d3.xml("/images/striff-hero-diagram.svg").then(function(data){
    var importedNode = document.importNode(data.documentElement, true);
    var g = svg.append("svg").attr("width","100%").attr("height","100%").attr("viewBox","0 0 800 500").append("g");
    g.node().appendChild(importedNode);
    svg.call(d3.zoom().scaleExtent([0.5,4]).on("zoom",function(e){
      g.attr("transform",e.transform);
    }));
  });
})();
</script>

The annotations look like this:

> **HIGH: Contract drift**
> `AbstractJavaCodegenTest` changes (EC increased) indicate test expectations have shifted alongside `AbstractJavaCodegen` behavior. When tests that compose with core classes change, it often signals a behavioral contract change rather than a pure refactor. Consequence: downstream language implementations and template consumers are likely to need updates within the next release window, increasing short-term integration friction. Tradeoff: evolving the core contract can unlock improved behavior but raises immediate upgrade work and client compatibility pressure.

> **REVIEW: Fan-in gravity**
> `ModelUtils` is a very large utility (WMC ~679) referenced by core code (`AbstractJavaCodegen`) and tests. That size makes it a coupling magnet: small changes to `ModelUtils` will ripple across many generators/tests in the short term (next release) and create substantial coordination cost over the medium term (few releases) as callers evolve. Tradeoff: centralizing parsing/validation logic reduces duplication today but increases the risk that future API tweaks become high-impact change events that block parallel work and pressure teams to copy-or-extend portions (copying will accelerate boundary erosion).

These are generated from structured signals, not from the diff text. The rest of this post explains how those signals are produced.

---

## The Pipeline in Brief

The system has three stages that run in order. Each one catches something the others cannot:

1. **Deterministic detectors** scan for hard rule violations -- dependency cycles, boundary crossings, hub formation. These are always wrong regardless of context.
2. **An edge-prediction GNN** scores every dependency in the diff for structural surprise -- "given patterns across thousands of codebases, should this edge exist?" This catches soft distributional patterns that no named rule covers.
3. **A grounded LLM** takes the findings from both stages alongside metric deltas and produces the review annotations you see above. It does not reason freely over a raw diff; it annotates a pre-structured set of signals.

The order matters. Deterministic detectors run first because they are cheap and binary. The GNN runs second because it is more expensive but catches what rules miss. The LLM runs last because its job is annotation, not detection -- and it needs structured evidence to produce grounded output rather than confident-sounding speculation.

---

## Code as a Graph

Source code has a natural graph structure. Components (classes, interfaces, enums, abstract classes) are nodes. The relationships between them are typed directed edges: inheritance (`extends`), realization (`implements`), association (holds a reference), and dependency (uses as a parameter). These edge types are not interchangeable -- inheriting from a class implies a tighter coupling contract than depending on one, and the model needs to know the difference.

<img src="/images/gnn-pipeline-diagram.svg" style="margin-left:auto; margin-right:auto; display: block;"/>

This representation is not new. Dependency graphs and call graphs appear throughout the software engineering literature. What has not gotten much attention is using GNNs for anomaly detection over these graphs -- detecting components whose structural neighbourhood deviates from what well-structured code looks like, rather than checking for named rule violations.

striff-lib is the open-source parsing and diagram core that striff.io is built on. It wraps Clarpse, a multi-language static analysis library, to extract this component and relation model from Java, Python, TypeScript, and C# source trees. Every node and typed edge in the GNN's input graph comes out of striff-lib.

### Extracting the Right Subgraph

Given a pull request, the first question is which subgraph to analyse. You cannot run GNN inference over the full repository on every PR. Changed files are parsed to extract the modified components (seed nodes), then the subgraph is expanded using bidirectional BFS for 3 hops over the full repo's relation graph. The result captures not just what changed, but what depends on it and what it depends on -- the structural blast radius of the PR.

Node capping is enforced at 500 nodes. Beyond that, inference latency grows faster than signal quality. The cap felt arbitrary when we picked it. It still does. We chose it because larger subgraphs blew the inference budget, not because of any principled analysis.

The graph maintains separate typed edges per relation type (following the R-GCN approach), rather than collapsing everything into a single adjacency matrix. INHERITANCE edges are aggregated differently from DEPENDENCY edges, which matters because the coupling regimes are different.

---

## Stage 1: Deterministic Detectors

Six detectors scan the PR diff for specific architectural violations before any ML runs. These encode hard rules -- patterns that are *always* problematic regardless of context:

| Detector | What It Catches |
|----------|----------------|
| **New Package Cycle** | New circular dependencies between packages (Tarjan SCC diffed against baseline) |
| **Boundary Crossing** | New cross-package edges not in baseline |
| **Stable Contract Change** | Modified component with high afferent coupling + signature change (AC >= 10) |
| **Hub Formation** | Fan-in growing from <5 to >=10 in a single PR |
| **Layer Skip** | New edge skipping >=2 architectural layers |
| **Cyclic Seed** | New edge A->B where path B->A already exists |

Each detector is a pure function. They run in parallel via a `DetectorRegistry`, and one detector failure does not block others. The output is a list of `Finding` records with severity, affected components, and a human-readable explanation.

These catch what is *always* wrong. A dependency cycle is a cycle regardless of what the training distribution says. But they miss something important.

---

## What the Rules Miss

Consider a component where no detector fires. Its WMC increased by 4, its efferent coupling by 3, and it gained two new inheritance dependents. No cycle. No boundary crossing. No hub threshold breached. Every individual metric is within normal range.

But in a neighbourhood where similar components have WMC under 10 and EC under 5, the *combination* is a distributional outlier. No single rule fires because no single signal is extreme. The pattern is unusual only when you see all of it together.

This is the gap the GNN fills. It catches distributional outliers where no hard rule applies but the overall structural pattern deviates from what well-structured code looks like.

---

## Stage 2: Edge-Prediction Anomaly Detection

### From Node Norms to Edge Prediction

The original version of this system scored anomalies at the node level: it computed per-node embedding norms, z-scored them, and flagged nodes whose norms deviated from the mean. After running this in production and auditing the results, we realised this was essentially degree-centrality detection wearing a neural network costume. High-degree nodes got large embedding norms, low-degree nodes got small ones, and the "anomaly" signal correlated almost perfectly with node connectivity. Useful, but not worth the overhead of a GNN.

The redesign uses what the model was actually trained to do: **edge prediction**.

### How It Works

The model is trained with a masked edge reconstruction objective. During training, roughly 50% of outgoing edges from focal nodes are masked, and the model must predict whether each masked edge should exist. It learns structural patterns like "service classes rarely depend on controller classes" and "interfaces are typically implemented by classes in the same package."

At inference time on a PR, we flip this around. For every dependency edge in the PR subgraph:

1. Feed the graph to the model **without** that edge
2. Ask: "given what you know about typical code structure, should this edge exist?"
3. If the model says "no" (low probability) -- that is an **anomalous dependency**

This is a fundamentally different question from "is this node unusual?" It asks "is this *specific dependency* structurally surprising given patterns across thousands of codebases?"

### What It Catches

- **Cross-layer violations**: a `service` depending on a `controller` is structurally rare across the corpus
- **Unusual coupling**: two components in unrelated packages suddenly wired together
- **God-class signals**: a class accumulating dependencies that no similar class has
- **Missing abstractions**: a concrete class directly depending on another concrete class when the pattern usually goes through an interface

It does *not* catch project-specific conventions (your project might intentionally do something unusual), semantic issues (the dependency is technically fine but the reason is wrong), or rare-but-valid patterns (structurally unusual but architecturally correct). This is why the deterministic detectors remain essential -- they catch hard rules, the GNN catches soft distributional patterns, and together they are complementary.

### Calibration

Raw edge probabilities are relative, not absolute. We calibrate them using percentile thresholds computed on held-out positive edges from the training corpus:

| Level | Edge probability | Meaning |
|-------|-----------------|---------|
| Normal | Above 50th percentile | This dependency looks typical |
| Possibly unusual | 25th-50th percentile | Somewhat uncommon |
| Likely unusual | 10th-25th percentile | Structural pattern is unexpected |
| Very unusual | Below 5th percentile | Strongly anomalous |

Thresholds ship in `calibration.json` alongside the ONNX model, ensuring scores mean the same thing across model retraining runs.

Only edges introduced by the PR are scored and surfaced to users. Background edges from the surrounding code provide context for the model's encoder, but a reviewer cannot act on a pre-existing dependency they did not introduce.

---

## How the Model Works

### Feature Vector

Every node is represented by a 405-dimensional feature vector spanning four groups: text embeddings of the component name and docstring (384 dims, via all-MiniLM-L6-v2), OOP structural metrics from the Chidamber-Kemerer suite (9 dims including WMC, DIT, NOC, afferent and efferent coupling), component type encoding (7 dims for class/interface/enum/method/field/annotation/other), and language plus synthetic node flags (5 dims).

The text embeddings matter because components with similar architectural roles (UserRepository, OrderRepository, ProductRepository) should be close in embedding space, letting the model learn that repository-like components have a characteristic structural neighbourhood.

<img src="/images/gnn-feature-vector-dimensions.svg" style="margin-left:auto; margin-right:auto; display: block;"/>

<img src="/images/striff-oopmetrics.png" style="margin-left:auto; margin-right:auto; display: block; max-width: 600px;"/>

The OOP metrics are z-score normalised per language. A Java class with WMC of 20 is unremarkable; a Python module with WMC of 20 is an outlier. Without per-language normalisation, the model learns spurious correlations between language choice and anomaly score.

### GCN Architecture and Distillation

The deployed scorer is a Graph Convolutional Network with an edge-prediction head, distilled from a larger teacher model. The teacher is an ArchGraphMAE -- a masked autoencoder with a 3-layer Heterogeneous Graph Transformer (HGT) encoder that learns type-specific transforms per relation type. The distilled GCN matches the teacher's edge probability distribution (gated on Pearson correlation >= 0.85 on held-out data) while being compact enough for synchronous inference during a live review request.

We chose spectral GCN over Graph Attention Networks after experimentation. GAT's learned per-edge attention weights produce score variance across structurally similar components in different PRs. We spent a good two weeks convinced the variance was a training bug before accepting it was structural. GCN's spectral normalisation gives up per-neighbour interpretability in exchange for stable, consistent score distributions -- a worthwhile tradeoff for a product where users see scores directly on their diagram.

The model runs on ONNX Runtime with three inputs: node features (N x 405), adjacency matrix (N x N), and edge queries (M x 2). It outputs per-edge probabilities in a single forward pass. Training graphs were built from 105 open-source repositories across Java, Python, TypeScript, and C#, totalling 4.9 million structural nodes. The corpus includes enterprise frameworks (Quarkus, Kafka, Spring), middleware (Netty, Flink), web applications (Django, FastAPI, NestJS), and smaller focused libraries.

> **The training pipeline is open-sourced.**
> The Python GNN training code, covering dataset preparation, model architecture, distillation, and ONNX export, is available at [github.com/hadi-technology/striff-gnn](https://github.com/hadi-technology/striff-gnn).

---

## Stage 3: Putting It Together

With deterministic findings, edge anomaly scores, and structural metric deltas all available, the question is how to combine them. The answer is deliberately simple.

All three signal types are serialised into a structured JSON payload alongside the compact diff and fed to the LLM agent. The agent annotates a pre-structured set of signals with architectural judgment and natural language. It does not reason freely over a raw diff.

This constraint does more work than it looks like. An LLM given a raw diff and asked "what are the architectural risks?" produces confident-sounding but structurally ungrounded output. An LLM told "*there is a new dependency cycle between these two packages, the edge from UserService to PaymentController has an anomaly score of 0.88 (the model does not expect service classes to depend on controller classes), its EC increased from 3 to 9 this PR, and it now has 4 incoming dependents it did not have before*" produces output grounded in actual structural evidence.

The symbolic layer also computes deterministic facts from the expanded subgraph: dependency cycles via Kosaraju SCC (at both class and package level), package boundary crossings, fan-in blast radius, and OOP metric deltas. These facts are binary and explainable -- a cycle either exists or it does not.

---

## Where to Go From Here

[striff.io](https://striff.io) is live. You can run it on any public GitHub pull request and get a visual architecture diff with AI review annotations.

striff-lib is open source. The parsing and diagram generation core is available if you want to explore the graph extraction layer or build on top of it.

The Python GNN training pipeline is available at [github.com/hadi-technology/striff-gnn](https://github.com/hadi-technology/striff-gnn). The corpus spans 105 open-source repositories across Java, Python, TypeScript, and C#, totalling 4.9 million structural nodes.

The staging pattern described here -- deterministic detectors followed by edge-prediction anomaly scoring followed by grounded LLM annotation -- is not specific to code review. Anywhere you have a domain representable as a structured graph where some properties are deterministically computable and others are distributional, this approach applies. Database schema evolution, API contract drift, infrastructure dependency analysis, security vulnerability propagation.
