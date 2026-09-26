---
title: "Detecting Architectural Anomalies in Code with Graph Neural Networks"
date: 2026-04-28 10:00:00
og_image: /images/gnn-pipeline-diagram.svg
tags: [gnn, code review, graph neural networks, software architecture, ml engineering]
toc: true
description: "How [striff.io](https://striff.io)'s spring 2026 pipeline scored pull requests with an edge-prediction graph neural network over typed dependency graphs, and why the deterministic layer decided what got said: neither the model's scores nor the LLM's prose could originate a finding."
excerpt_separator: <!--more-->
---

> *Update, September 2026.* The model this post describes was retired in August 2026, and the deterministic detectors followed in September. Striff now reads the architecture documents already in a repository and checks each pull request against them. The principle in the last section, that a language model may phrase a finding but never originate one, survived into the current design; the GNN did not. This post stays up as a record of what was built and why it was withdrawn. The current system is described in [a post on the Striff blog](https://striff.io/blog/design-docs-are-enforceable-now).

Code review has a specific information problem that most tooling ignores. When you open a pull request on a large codebase, the diff shows you *lines*. It does not show you that the class you just modified now has fourteen things depending on it when it had three last week. It does not show you that a new import three files away quietly created a dependency cycle between two packages that were previously clean. It does not show you that the abstraction an LLM just extended sits at depth six in an inheritance tree that has been growing for two years.

These are not edge cases. They are the class of change that produces architectural debt, the kind that compounds quietly and becomes expensive to unwind.

That is the problem the spring 2026 version of [striff.io](https://striff.io) was built to address. It reviewed the architecture of a pull request: which components changed, how their relationships shifted, and which of those shifts carried structural risk, posted as a GitHub check and drawn on a class diagram.

The interesting engineering question is not the diagram rendering. It is *what earns the right to be said out loud*. This post is largely about the constraints we put on our own machine learning: a GNN whose scores users never saw, and an LLM that was structurally forbidden from telling you something a deterministic detector had not already found. Those sound like limitations. They are the reason the output is trustworthy, and getting there took removing capability, not adding it. The infrastructure that runs the pipeline (queueing, in-process inference, degradation modes) is covered in a [companion post]({% post_url 2026-04-28-striff-io-ml-infrastructure %}).

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

The findings look like this:

> **Structural Regression: Package cycle detected**
> This PR completes a dependency cycle among 3 packages (7 edges) by adding `manager.persistence -> manager.models`.

> **Structural Regression: Package structure violation**
> `DefaultCommandHandlerManager` was added to `core.engine`, but `ARCHITECTURE.md` requires command handlers to live in `core.handlers`.

Two properties of those findings matter more than their wording. Each one **cites something you can go and check**: a named edge that exists in the diff, or a verbatim line from a document in the repository. And each one is **reproducible** — run the same PR through twice and you get the same finding, because a detector computed it rather than a model deciding it was worth mentioning.

An earlier version of the product wrote in a different register. Here is a real annotation it produced:

> *`ModelUtils` is a very large utility (WMC ~679) referenced by core code and tests. That size makes it a coupling magnet: small changes will ripple across many generators/tests in the short term and create substantial coordination cost over the medium term as callers evolve. Tradeoff: centralizing parsing/validation logic reduces duplication today but increases the risk that future API tweaks become high-impact change events...*

It is fluent, it is plausible, and most of it is unfalsifiable. "Substantial coordination cost over the medium term" cannot be checked, cannot be wrong, and cannot be acted on. Worse, the same PR could produce a differently-worded version of it on the next run. The rest of this post is largely the story of how the pipeline was rebuilt so that output like that can no longer reach a user.

---

## The Pipeline in Brief

The system has three stages that run in order, and — this is the part that took us longest to get right — only the first one is allowed to originate a finding:

1. **Deterministic detectors** scan for structural violations: dependency cycles, directional boundary crossings, complexity growth, documented-rule violations. These are computed, not inferred.
2. **An edge-prediction GNN** scores dependencies for structural surprise: "given patterns across thousands of codebases, should this edge exist?" This catches soft distributional patterns no named rule covers. Its scores are **evidence attached to findings, never a finding by themselves.**
3. **An LLM** may rewrite the prose of a finding that already exists. It cannot add one. An annotation that does not match a fact is discarded.

The order matters, but the *permissions* matter more. Each stage downward is less deterministic and therefore trusted with less authority: the detectors decide what is true, the GNN decides what is interesting among things already known to be true, and the LLM decides only how it reads. A pipeline where every stage can add its own findings is a pipeline whose output quality is bounded by its least reliable component.

{% include striff-permission-demo.html %}

---

## Code as a Graph

Source code has a natural graph structure. Components (classes, interfaces, enums, abstract classes) are nodes. The relationships between them are typed directed edges: inheritance (`extends`), realization (`implements`), association (holds a reference), and dependency (uses as a parameter). These edge types are not interchangeable: inheriting from a class implies a tighter coupling contract than depending on one, and the model needs to know the difference.

<a class="post-figure__link" href="/images/gnn-pipeline-diagram.svg" target="_blank" rel="noopener"><img src="/images/gnn-pipeline-diagram.svg" class="post-figure"/></a>

This representation is not new. Dependency graphs and call graphs appear throughout the software engineering literature. What has not gotten much attention is using GNNs for anomaly detection over these graphs, detecting components whose structural neighbourhood deviates from what well-structured code looks like, rather than checking for named rule violations.

striff-lib is the open-source parsing and diagram core that striff.io is built on. It wraps Clarpse, a multi-language static analysis library, to extract this component and relation model from Java, Python, TypeScript, and C# source trees. Every node and typed edge in the GNN's input graph comes out of striff-lib.

### Extracting the Right Subgraph

Given a pull request, the first question is which subgraph to analyse. You cannot run GNN inference over the full repository on every PR. Changed files are parsed to extract the modified components (seed nodes), then the subgraph is expanded using bidirectional BFS for 3 hops over the full repo's relation graph. The result captures not just what changed, but what depends on it and what it depends on: the structural blast radius of the PR.

Node capping is enforced at 500 nodes. Beyond that, inference latency grows faster than signal quality. The cap felt arbitrary when we picked it. It still does. We chose it because larger subgraphs blew the inference budget, not because of any principled analysis.

The extracted graph keeps relation types distinct — INHERITANCE is not the same edge as DEPENDENCY, because the coupling regimes differ — and the HGT teacher learns type-specific transforms over them. The distilled student that actually runs in production collapses those into a single adjacency matrix. That is a real loss of fidelity, and it is the price of a model small enough to score synchronously inside a review request; the teacher's type-awareness survives only as far as it shaped the student's weights during distillation.

---

## Stage 1: Deterministic Detectors

Twelve detectors scan the PR diff for specific structural violations before any ML runs. Each is a pure function over the before/after graph, they run in parallel via a `DetectorRegistry` so one failure does not block the others, and each emits `Finding` records with severity, affected components, and evidence.

| Detector | What It Catches |
|----------|----------------|
| **New Package Cycle** | A cycle among packages that the baseline did not have (Kosaraju SCC, diffed against base) |
| **New Directional Boundary Crossing** | A new cross-package edge; severity is HIGH only when it *inverts* an existing dependency |
| **Stable Contract Change** | Signature change on a component many others depend on (fires at AC >= 5, HIGH at >= 10) |
| **WMC Growth** | Weighted-methods-per-class rising sharply on an existing class |
| **Hub Formation** | Fan-in crossing from <=2 to >=4 dependents in a single PR |
| **Layer Skip** | A new edge skipping >= 2 architectural layers |
| **Cyclic Dependency Seed** | New edge A->B where a path B->A of length 2-4 already exists |
| **Instability Spike** | A component's efferent/afferent balance shifting sharply toward instability |
| **Encapsulation Drop** | Internals becoming more exposed than they were |
| **Production Depends On Test** | Production code acquiring a dependency on test code |
| **Interface To Concrete Downgrade** | A dependency moving from an interface to a concrete implementation |
| **Module Boundary Violation** | A cross-module edge that the monorepo's own package boundaries forbid |

Two more findings come from documentation rather than graph shape: a **Doc Dependency Rule** violation (a dependency contradicting a rule written in the repo's own architecture docs) and a **Doc Architecture Advisory** (a documented intention the change may erode). Both must cite a verbatim line from a real file in the repository, which is the only reason they are allowed to exist alongside the graph-derived detectors.

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

The model is trained with a masked edge reconstruction objective. During training, half the focal nodes are sampled and *all* of their outgoing edges are masked, and the model must predict whether each masked edge should exist, with hard negatives drawn from 2-hop neighbours. It learns structural patterns like "service classes rarely depend on controller classes" and "interfaces are typically implemented by classes in the same package."

At inference time on a PR, we flip this around. For every dependency edge in the PR subgraph:

1. Feed the graph to the model **without** that edge
2. Ask: "given what you know about typical code structure, should this edge exist?"
3. If the model says "no" (low probability), that is an **anomalous dependency**

This is a fundamentally different question from "is this node unusual?" It asks "is this *specific dependency* structurally surprising given patterns across thousands of codebases?"

### What It Catches

- **Cross-layer violations**: a `service` depending on a `controller` is structurally rare across the corpus
- **Unusual coupling**: two components in unrelated packages suddenly wired together
- **God-class signals**: a class accumulating dependencies that no similar class has
- **Missing abstractions**: a concrete class directly depending on another concrete class when the pattern usually goes through an interface

It does *not* catch project-specific conventions (your project might intentionally do something unusual), semantic issues (the dependency is technically fine but the reason is wrong), or rare-but-valid patterns (structurally unusual but architecturally correct). This is why the deterministic detectors remain essential: they catch hard rules, the GNN catches soft distributional patterns, and together they are complementary.

### Calibration

Raw edge probabilities are relative, not absolute. A score of 0.4 means nothing on its own; it only means something against the distribution of scores the model assigns to edges it has seen. So thresholds are computed as percentiles over held-out positive edges from the training corpus and shipped in `calibration.json` alongside the ONNX model, which keeps a given score meaning the same thing across retraining runs. On the current corpus the 5th percentile sits at 0.31 and the median at 0.73; lower means more surprising.

Three bands come out of that file — anomalous, advisory, and normal — and the product layer is deliberately more conservative still, applying flat cutoffs at 0.30 and 0.60 rather than tracking the corpus percentiles. That gap between "what the calibration file supports" and "what we actually act on" is intentional: percentile bands move when the corpus changes, and we would rather the user-facing behaviour not shift under a retrain.

An edge is scored when **either endpoint is a component the PR changed**, not only when the edge itself is new. Pre-existing dependencies touching changed code are part of the blast radius and the model needs them. Downstream, only edges the diff actually added are reported, since a reviewer cannot act on a dependency they did not introduce.

---

## How the Model Works

### Feature Vector

Every node is represented by a 403-dimensional feature vector in a fixed layout: text embeddings of the component name and docstring (384 dims, via all-MiniLM-L6-v2), OOP structural metrics from the Chidamber-Kemerer suite (9 dims including WMC, DIT, NOC, afferent and efferent coupling), component type one-hot (5 dims: class, interface, enum, struct, other), language one-hot (4 dims), and a synthetic-node flag (1 dim).

The layout is worth stating precisely because it is load-bearing. The scorer pins the expected dimension in `ModelMetadata` and refuses to start if the loaded model disagrees — a retrained model that shuffled the metric block would otherwise produce plausible, wrong scores and throw nothing.

The text embeddings matter because components with similar architectural roles (UserRepository, OrderRepository, ProductRepository) should be close in embedding space, letting the model learn that repository-like components have a characteristic structural neighbourhood.

<a class="post-figure__link" href="/images/gnn-feature-vector-dimensions.svg" target="_blank" rel="noopener"><img src="/images/gnn-feature-vector-dimensions.svg" class="post-figure"/></a>

<a class="post-figure__link" href="/images/striff-oopmetrics.png" target="_blank" rel="noopener"><img src="/images/striff-oopmetrics.png" class="post-figure"/></a>

The OOP metrics are z-score normalised per language. A Java class with WMC of 20 is unremarkable; a Python module with WMC of 20 is an outlier. Without per-language normalisation, the model learns spurious correlations between language choice and anomaly score.

### GCN Architecture and Distillation

The deployed scorer is a Graph Convolutional Network with an edge-prediction head, distilled from a larger teacher. The teacher is an ArchGraphMAE — a masked autoencoder with a 3-layer Heterogeneous Graph Transformer (HGT) encoder, 4 heads, hidden dimension 128, learning type-specific transforms per relation type. The student is a homogeneous GCN over a collapsed adjacency, trained to match the teacher and compact enough for synchronous inference inside a live review request.

We chose spectral GCN over Graph Attention Networks after experimentation. GAT's learned per-edge attention weights produce score variance across structurally similar components in different PRs. We spent a good two weeks convinced the variance was a training bug before accepting it was structural. GCN's spectral normalisation gives up per-neighbour interpretability in exchange for stable, consistent score distributions.

Worth flagging honestly: the export path validates the HGT against a tight numerical tolerance, but **the distilled student ships without an automated agreement gate against its teacher.** Distillation quality is currently eyeballed from the training run rather than enforced at export time, which is a real gap — a student that silently drifted from the teacher would not be caught by CI. It is on the list precisely because the equivalent check already exists one layer up.

The model runs on ONNX Runtime with three inputs — node features (N x 403), adjacency matrix (N x N), and edge queries (M x 2) — emitting per-edge scores in a single forward pass. Training graphs were built from 60 open-source repositories, 20 each across Java, Python and TypeScript, totalling roughly 4.2 million structural nodes. The corpus includes enterprise frameworks (Quarkus, Spring), middleware, web applications (Django, FastAPI, NestJS), and smaller focused libraries. C# is parsed and scored in production but is not represented in the training corpus, so its scores lean on patterns learned from the other three languages — a known weakness rather than a designed behaviour.

> **The training pipeline is open-sourced.**
> The Python GNN training code, covering dataset preparation, model architecture, distillation, and ONNX export, is available at [github.com/hadi-technology/striff-gnn](https://github.com/hadi-technology/striff-gnn).

---

## Stage 3: Who Is Allowed to Say Something

With deterministic findings, edge anomaly scores, and structural metric deltas all available, the obvious move is to hand all of it to an LLM and let it write the review. That is what the first version did, and it is the part we had to undo.

The failure was not hallucination in the usual sense. The model rarely invented a class that did not exist. What it did was assign *significance* — deciding that a real edge was concerning, in fluent prose, with no way for anyone to check whether it was. Two runs over the same PR would surface different concerns. There was no way to measure precision, because there was no stable set of claims to measure.

So the pipeline was rebuilt around a single rule: **a deterministic fact is the only thing that can originate a user-visible finding.**

Concretely, the surfacing stage builds items from detector findings first. LLM annotations are then matched against those items by target — a component or an edge naming one of its endpoints. If an annotation matches, its prose *replaces the wording* of that finding while the finding's identity, priority, targets and evidence stay untouched. If an annotation matches nothing, it is dropped and counted. There is no tier at which an unmatched claim is surfaced with a lower confidence label, because a confidence label on an unverifiable claim is just a hedge.

The GNN sits under the same rule. Its scores enrich findings as evidence, and its anomalous-edge count appears as an aggregate in the check run, but **no anomalous edge becomes a finding on its own.** The reason is precision measurement, not distrust of the model: we have never had a clean way to tell a "structurally surprising" edge from a "structurally surprising and actually worth your time" edge, and until that number exists, promoting scores to findings would be raising volume, not value.

The same conservatism applies within the detectors themselves. Twelve detectors run; **five reach users.** New package cycles always surface. Directional boundary crossings, stable contract changes, WMC growth and documented-rule violations surface only at HIGH severity. The remaining detectors — hub formation, layer skip, instability spikes, production-depends-on-test, interface downgrades, module boundary violations, cyclic seeds, encapsulation drops — run on every PR and stay evidence-only, feeding the payload without ever speaking. Each stays quiet until its precision has been measured on real repositories, because a detector promoted on the strength of "this seems obviously bad" is how a review tool starts costing more attention than it saves.

That is the tradeoff worth being explicit about: this design is biased hard toward silence. It will miss real problems that an evidence-only detector spotted. We would rather have that than a reviewer who learns to scroll past us.

The symbolic layer underneath computes the facts these decisions rest on: dependency cycles via Kosaraju SCC at both class and package level, package boundary crossings, fan-in blast radius, and OOP metric deltas. These are binary and explainable. A cycle either exists or it does not, and if we say it does, you can go and find it.

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

[striff.io](https://striff.io) is live. You can run it on any public GitHub pull request and get a visual architecture diff with AI review annotations.

striff-lib is open source. The parsing and diagram generation core is available if you want to explore the graph extraction layer or build on top of it.

The Python GNN training pipeline is available at [github.com/hadi-technology/striff-gnn](https://github.com/hadi-technology/striff-gnn). The corpus spans 60 open-source repositories across Java, Python and TypeScript, totalling roughly 4.2 million structural nodes.

The pattern described here is not specific to code review: deterministic computation decides what is true, a learned model ranks what is interesting among things already known to be true, and a language model is allowed to phrase it and nothing else. Anywhere you have a domain representable as a structured graph where some properties are deterministically computable and others are distributional, the same permission ordering applies — database schema evolution, API contract drift, infrastructure dependency analysis, security vulnerability propagation.

The generalisable lesson is narrower than "use a neurosymbolic pipeline." It is this: **the hard part of shipping ML in a review product is not detection, it is deciding what earns the right to interrupt someone.** Every stage we constrained — the GNN that scores but cannot speak, the seven detectors that run but stay silent, the LLM that can only reword — made the product quieter and more trusted at the same time. That was not the intuition we started with.

---

*Muntazir Fadhel builds production AI and ML infrastructure. He is the founder of HADI Technology. [Technical Profile](/downloads/MFadhel_Engagement_Brief.pdf) · [Get in Touch](/contact/)*
