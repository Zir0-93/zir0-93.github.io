---
title: "Detecting Architectural Anomalies in Code with Graph Neural Networks"
date: 2026-04-28 10:00:00
og_image: /images/gnn-pipeline-diagram.svg
tags: [gnn, code review, graph neural networks, software architecture, ml engineering]
toc: true
description: "How [striff.io](https://striff.io) uses a neurosymbolic pipeline with typed dependency graphs, Chidamber-Kemerer features, and a distilled GCN to flag the parts of a pull request that actually carry architectural risk. The post covers the graph construction pipeline, the 404-dimensional feature vector design, why spectral GCN was chosen over attention-based architectures, and how symbolic facts are fused with learned anomaly scores to produce grounded LLM review annotations rendered directly on architecture diagrams."
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

## Why Code Is a Graph

Source code has a natural graph structure. Components, which are classes, interfaces, enums, and abstract classes, are nodes. The relationships between them are typed directed edges:

- **INHERITANCE**: a class extends another. Implies substitutability, tight coupling, and propagation of interface contracts up and down the hierarchy.
- **REALIZATION**: a class implements an interface. A formal contract. Breaking the source of this edge is a breaking change for every downstream consumer.
- **ASSOCIATION**: a class holds a reference to another, typically as a field. Implies ownership or composition.
- **DEPENDENCY**: a class uses another as a method parameter or local variable. Directional and weaker than association, but still structurally significant.

<img src="/images/gnn-pipeline-diagram.svg" style="margin-left:auto; margin-right:auto; display: block;"/>

These edge types are not interchangeable. INHERITANCE implies a different coupling regime than DEPENDENCY. A tool that flattens them into a single "connected" relationship is throwing away signal the model could use. More on that tradeoff shortly.

This is also not a new representation in program analysis. Dependency graphs, call graphs, and class hierarchy graphs appear throughout the software engineering literature. What has not gotten much attention is using GNNs for anomaly detection over these graphs specifically, detecting components whose structural neighbourhood deviates from what a well-structured codebase looks like, rather than checking for named rule violations.

That gap is what the GNN fills.

striff-lib is the open-source parsing and diagram core that [striff.io](https://striff.io) is built on. It wraps Clarpse, a multi-language static analysis library, to extract this component and relation model from Java, Python, and TypeScript source trees. Every node and typed edge in the GNN's input graph comes out of striff-lib.

---

## Graph Construction: Seeds, Neighbourhood Expansion, and the Heterogeneous Edge Problem

Given a pull request, the first question is which subgraph to analyse. You cannot run GNN inference over the full repository on every PR. A large codebase might have tens of thousands of components, and most of the graph is unchanged and irrelevant to the review.

The right unit is the PR neighbourhood. Changed files are parsed to extract the modified components. These become the seed nodes. From there, we expand the subgraph using bidirectional BFS over the full repo's relation graph, following edges in both directions for N hops (currently 3, configurable). The result captures not just what changed, but what depends on it (fan-in) and what it depends on (fan-out) -- the structural blast radius of the PR.

Node capping is enforced deterministically: if expansion exceeds 500 nodes, non-seed nodes are retained in BFS hop-distance order, closest to seeds first, with lexicographic tie-breaking. The 500-node cap felt arbitrary when we picked it. It still does. We chose it because larger subgraphs blew the inference budget, not because of any principled analysis.

The model must also generalise to subgraphs it has never seen before. Every PR brings a different neighbourhood from a different repository. This rules out transductive approaches like Node2Vec that embed specific nodes at training time and cannot generalise beyond the training graph. The model needs to describe every node purely through its feature vector. Feature engineering carries a lot of weight here.

### Heterogeneous Relation Types

The graph we extract is heterogeneous, and the model treats it that way. Each relation type (INHERITANCE, REALIZATION, ASSOCIATION, DEPENDENCY) gets its own typed edge index. Rather than collapsing everything into a single adjacency matrix, the scorer maintains separate weight matrices per relation type, following the R-GCN approach. This lets the model learn that INHERITANCE edges should be aggregated differently from DEPENDENCY edges, which they should: inheriting from a class is not the same coupling contract as depending on one, and flattening them loses that signal.

---

## Node Feature Engineering: The 404-Dimensional Feature Vector

Every node is represented by a 404-dimensional feature vector. The breakdown is intentional across four groups:

<img src="/images/gnn-feature-vector-dimensions.svg" style="margin-left:auto; margin-right:auto; display: block;"/>

### Text Embeddings (dims 0-383)

Each node is described by concatenating its display name and docstring. This string is encoded using all-MiniLM-L6-v2, a sentence transformer that produces 384-dimensional dense embeddings.

Sentence embeddings over bag-of-words matters here. Components with similar architectural roles, like UserRepository, OrderRepository, and ProductRepository, should be close in embedding space. Bag-of-words would place them near each other only on the shared token "Repository". Sentence embeddings capture the broader semantic pattern, which is what we want: the model should learn that repository-like components have a characteristic structural neighbourhood, and deviation from that is a signal.

MiniLM was chosen over larger sentence transformers for inference latency. The text encoder runs synchronously during review generation, so embedding time compounds with subgraph size. MiniLM produces embeddings of comparable quality for short technical strings at roughly half the cost of larger models.

### OOP Metrics (dims 384-392)

The next nine dimensions are the Chidamber-Kemerer metric suite plus three additional structural metrics:

- **WMC** (Weighted Methods per Class): sum of cyclomatic complexity across methods. High WMC correlates with low cohesion.
- **DIT** (Depth of Inheritance Tree): depth in the hierarchy. High DIT increases the surface area of inherited behaviour.
- **NOC** (Number of Children): direct subclasses. In combination with high DIT, signals fragile base class patterns.
- **AC** (Afferent Coupling, fan-in): classes outside the package that depend on this class. High AC means wide blast radius.
- **EC** (Efferent Coupling, fan-out): classes this component depends on. High EC means many reasons to change.
- **Encapsulation score**: derived from the ratio of public to total members.
- **Cyclomatic complexity**, **refs count**, and **children count** round out the vector.

The Chidamber-Kemerer metrics date back to 1994 and have held up in empirical software quality work (Basili, Briand and Melo 1996; Zhou and Leung 2005). They are not perfect defect predictors individually, but as a multivariate signal they correlate well with maintenance burden and architectural fragility.

<img src="/images/striff-oopmetrics.png" style="margin-left:auto; margin-right:auto; display: block; max-width: 600px;"/>

Z-score normalisation per language is applied to the first six metric dimensions. This is non-negotiable. A Java class with a WMC of 20 is unremarkable. A Python module with WMC of 20 is an outlier. Without per-language normalisation using pre-computed training statistics, the model learns spurious correlations between language choice and anomaly score. Normalisation statistics are stored in metric_normalizer.json and loaded at scorer initialisation.

### Component Type and Language (dims 393-403)

Seven dimensions encode component type (CLASS, INTERFACE, ENUM, METHOD, FIELD, ANNOTATION, OTHER). This matters because metric values alone are insufficient. An interface with high efferent coupling is normal by design. A concrete class with the same coupling profile is a different story. The type encoding lets the model learn type-conditional structural norms. Three language dimensions (Java, Python, TypeScript) and one synthetic node flag complete the vector.

---

## The GCN Architecture: Why Spectral Over Attention

### Message Passing

A Graph Convolutional Network updates each node's representation by aggregating its neighbourhood:

$$
H^{(l+1)} = \sigma\!\left(\tilde{D}^{-\frac{1}{2}}\,\tilde{A}\,\tilde{D}^{-\frac{1}{2}}\,H^{(l)}\,W^{(l)}\right)
$$

Where $\tilde{A} = A + I$ (adjacency with self-loops added), $\tilde{D}$ is the corresponding degree matrix, $H^{(l)}$ is the node representation matrix at layer $l$, and $W^{(l)}$ is a learnable weight matrix.

![GCN message passing](/images/gcn-message-passing.svg){: .light-border }

### Why Symmetric Normalisation Matters

The $\tilde{D}^{-\frac{1}{2}}\,A\,\tilde{D}^{-\frac{1}{2}}$ normalisation scales messages by the degree of both the source and the target node. Without it, high-degree nodes (god classes, utility packages with dozens of dependents) flood the neighbourhood aggregation of everything connected to them. In code graphs this is a real problem. A base exception class or a logging utility might have 50+ DEPENDENCY edges. Under unnormalised aggregation, every component in its neighbourhood starts to look structurally similar to a high-centrality hub, making anomalies harder to detect.

Self-loops deserve a note. Without them, a node's own features disappear from its representation after one round of message passing. Including self-loops in $\tilde{A}$ ensures every node aggregates itself alongside its neighbours.

### Why GCN Over GAT

Graph Attention Networks learn per-edge attention weights, allowing differential weighting of neighbour contributions. For an anomaly scorer this sounds appealing: attention weights could tell you which neighbours the model considered most suspicious. The problem in practice is score stability.

Attention weights are learned and vary per input graph. Two structurally similar components in different PRs can receive different attention allocations depending on local neighbourhood context, which produces score variance that is hard to reason about operationally. For a product like [striff.io](https://striff.io) where users see review notes tied directly to anomaly scores on their diagram, unexplained score variance erodes trust quickly.

We experimented with GAT during development. Score distributions were broader and less calibrated against our threshold values, requiring more frequent recalibration as training data grew. We spent a good two weeks convinced the variance was a training bug before accepting it was structural. GCN produced tighter, more consistent distributions. Spectral normalisation gives up per-neighbour interpretability in exchange for stability.

### Distillation for Deployment

The deployed scorer is a distilled version of a larger model. Training used an ArchGraphMAE -- a masked autoencoder with a 3-layer Heterogeneous Graph Transformer (HGT) encoder (128 hidden dimensions, 4 attention heads per layer). The encoder learns type-specific key and value transforms per relation type, plus edge-type attention priors, giving it the capacity to model different coupling regimes for different edge types. The decoder uses five independent MLP heads (one per edge type) for masked edge reconstruction. Training masks roughly 50% of outgoing edges from focal nodes and uses hard negative sampling from 2-hop neighbours. The distilled GCN matches the larger model's output distribution on held-out graphs while being compact enough for synchronous inference during a live review request.

The reason to train-then-distill rather than train small directly: the larger model generalises better and produces better-calibrated anomaly scores. Knowledge distillation transfers this into an inference-efficient form. The deployed model runs on ONNX Runtime.

> **The training pipeline is open-sourced.**
> The Python GNN training code, covering dataset preparation, model architecture, distillation, and ONNX export, is available at [github.com/hadi-technology/striff-gnn](https://github.com/hadi-technology/striff-gnn).

---

## Anomaly Scoring: Training Objective and What "Anomalous" Means

Anomaly in a code graph means a component whose structural neighbourhood deviates from patterns seen in well-structured codebases. High unexpected coupling density, atypical fan-in/out for its component type, metric combinations that fall outside the training distribution, or a neighbourhood that looks nothing like similarly-typed components in healthy code.

Training graphs were built from 60 open-source repositories across Java, Python, and TypeScript (20 per language), including large enterprise frameworks, middleware, and web applications. These form the healthy class. Anomalous examples come from two sources: synthetically generated architectural violations (introduced cycles, deliberate god class construction, boundary crossings) and real historical debt that was later refactored out, extracted from commit history. Class imbalance is addressed with weighted sampling. Architectural anomalies are rare in maintained codebases, and a naive training setup would learn to predict healthy for everything.

Graph sizes in training data ranged from roughly 15 to 480 nodes with a median around 60. Edge density varied considerably between tightly-coupled Java enterprise codebases and functional Python repositories.

The model outputs a per-node anomaly score in [0, 1], mapped back to seed nodes only. Thresholds are configurable: medium anomaly at 0.4, high at 0.7. These reflect a product-level decision about sensitivity, not a fixed model property. A team on a greenfield microservice might tune them tighter than a team maintaining a legacy monolith where some debt is accepted.

The value of the continuous score is in ranking. A dependency cycle is binary: it either exists or it does not. But consider a component that introduced no cycle, crossed no package boundary, and tripped no individual metric threshold. Its WMC increased by 4, EC by 3, and it gained two new inheritance dependents. None of these changes triggers a rule. The combination, in a neighbourhood where similar components have WMC under 10 and EC under 5, is a *distributional outlier* the GNN catches. Rules don't catch distributional outliers.

---

## Neurosymbolic Fusion: Combining Symbolic and Learned Signals

With both symbolic facts and GNN scores available, the question is how to combine them. The answer is deliberately simple.

<img src="/images/gnn-pipeline-diagram.svg" style="margin-left:auto; margin-right:auto; display: block;"/>


The symbolic layer computes deterministic facts from the expanded subgraph:

- **Dependency cycles**: Kosaraju SCC on JGraphT, at both class and package level, filtered to cycles involving seed nodes.
- **Package boundary crossings**: new cross-package edges introduced by changed components.
- **Fan-in blast radius**: upstream dependents outside the changed set.
- **OOP metric deltas**: per-component changes in WMC, coupling, fan-in, and encapsulation across the PR.

These facts are binary and explainable. A cycle either exists or it does not. There is no probability distribution, no threshold sensitivity, no hallucination risk.

The GNN score is continuous and distributional. It captures structural deviance that no named rule fires on. Useful for ranking but weaker as standalone evidence.

The fusion strategy: symbolic facts are hard signals, anomaly scores are soft ranking signals. Both are serialised into a structured JSON payload alongside the compact diff and fed to the LLM agent. The agent annotates a pre-structured set of signals with architectural judgment and natural language. It does not reason freely over a raw diff.

This constraint on the LLM does more work than it looks like. An LLM given a raw diff and asked "what are the architectural risks?" produces confident-sounding but structurally ungrounded output. An LLM told "*there is a new dependency cycle between these two packages, this component has an anomaly score of 0.71, its EC increased from 3 to 9 this PR, and it now has 4 incoming dependents it did not have before*" produces output grounded in actual structural evidence.

The output in [striff.io](https://striff.io) looks like this, rendered as sticky notes directly on the architecture diagram:

<img src="/images/striff-hero-diagram-zoomed.svg" style="margin-left:auto; margin-right:auto; display: block; max-width: 100%; border-radius: 8px;"/>

Examples of review annotations the system produces:

> **HIGH: Contract drift**
> `AbstractJavaCodegenTest` changes (EC increased) indicate test expectations have shifted alongside `AbstractJavaCodegen` behavior. When tests that compose with core classes change, it often signals a behavioral contract change rather than a pure  refactor. Consequence: downstream language implementations and template consumers are likely to need updates within the next release window, increasing short-term integration friction. Tradeoff: evolving the core contract can unlock improved behavior but raises immediate upgrade work and client compatibility pressure.

> **REVIEW: Fan-in gravity**
> `ModelUtils` is a very large utility (WMC ~679) referenced by core code (`AbstractJavaCodegen`) and tests. That size makes it a coupling magnet: small changes to `ModelUtils` will ripple across many generators/tests in the short term (next release) and create substantial coordination cost over the medium term (few releases) as callers evolve. Tradeoff: centralizing parsing/validation logic reduces duplication today but increases the risk that future API tweaks become high-impact change events that block parallel work and pressure teams to copy-or-extend portions (copying will accelerate boundary erosion). Evidence: the changeset shows heavy `ModelUtils` surface area and explicit deps from `AbstractJavaCodegen` and tests.


These are generated from structured signals, not from the diff text.

---

## Where to Go From Here

[striff.io](https://striff.io) is live. You can run it on any public GitHub pull request and get a visual architecture diff with AI review annotations.

striff-lib is open source. The parsing and diagram generation core is available if you want to explore the graph extraction layer or build on top of it.

The Python GNN training pipeline is available at [github.com/hadi-technology/striff-gnn](https://github.com/hadi-technology/striff-gnn).

The staging pattern described here, symbolic analysis followed by learned scoring followed by grounded LLM annotation, is not specific to code review. Anywhere you have a domain representable as a structured graph where some properties are deterministically computable and others are distributional, this approach applies. Database schema evolution, API contract drift, infrastructure dependency analysis, security vulnerability propagation.
