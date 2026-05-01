---
title: "Building a Production Neurosymbolic Pipeline for Scientific Discourse Graphs"
date: 2026-04-26 10:00:00
og_image: /images/fylo-pipeline-overview.svg
tags: [nlp, knowledge graphs, neurosymbolic, scientific discourse, llm engineering, shex]
toc: true
description: "Lessons from building [Fylo](https://fylo.io/)'s ingestion pipeline that turns scientific papers into typed discourse graphs. Covers ShEx schema as executable contract, a phased LLM extraction loop that closes the validator-LLM feedback gap, and a three-tier cross-document merge policy that keeps the graph converging as more papers are ingested."
excerpt_separator: <!--more-->
---

LLMs are competent at reading scientific papers. They are not yet competent at producing knowledge graphs that downstream systems can trust. The output is plausible, well-formatted, and frequently wrong in ways that only become visible after the graph is queried, joined to other data, or used to train another model.

Through HADI Technology, I worked with [Fylo](https://fylo.io/) on the ingestion pipeline that turns scientific papers into a typed discourse graph of claims, evidence, hypotheses, and research questions. The interesting engineering was not the LLM extraction. It was the symbolic scaffolding around it that turned a noisy generative process into a graph reliable enough to ship.

<div id="fylo-graph-intro" style="margin:24px auto;max-width:700px;border:1px solid rgba(15,23,42,0.12);border-radius:12px;overflow:hidden;background:#0f172a;"></div>
<script src="https://d3js.org/d3.v7.min.js"></script>
<script>
(function(){
var nodes=[
{id:"RQ1",type:"ResearchQuestion",title:"How do bioelectric circuits control anatomical outcomes?"},
{id:"A1",type:"Association",title:"Gap junction proteins and ion channel expression regulate bioelectrical signals"},
{id:"A2",type:"Association",title:"Turing-like patterns arise from bioelectric mechanisms"},
{id:"A3",type:"Association",title:"Epithelial tissue geometry directs bioelectric field"},
{id:"A4",type:"Association",title:"Connexin43 regulates joint location in zebrafish fins"},
{id:"A5",type:"Association",title:"Calcium oscillations coordinate feather mesenchymal cell movement"},
{id:"F1",type:"Factor",title:"Morphogenetic outcomes"},
{id:"F2",type:"Factor",title:"TRPC6"},
{id:"F3",type:"Factor",title:"Membrane potential in mammalian cancer cells"},
{id:"F4",type:"Factor",title:"Gene expression domains"},
{id:"F5",type:"Factor",title:"Endogenous bioelectric gradients"},
{id:"F6",type:"Factor",title:"H+/K+-ATPase activity and membrane potential"},
{id:"F7",type:"Factor",title:"PI3K-mediated electrotaxis of neural progenitor cells"},
{id:"F8",type:"Factor",title:"Osteogenic and Chondrogenic Master Genes expression"},
{id:"F9",type:"Factor",title:"Size of zebrafish fins"},
{id:"EL1",type:"EvidenceLine",title:"hERG1 channel conformation impacts cancer progression"}
];
var links=[
{source:"RQ1",target:"A1",type:"motivates",conf:.9},
{source:"RQ1",target:"A2",type:"motivates",conf:.8},
{source:"RQ1",target:"A3",type:"motivates",conf:.9},
{source:"RQ1",target:"A4",type:"motivates",conf:.9},
{source:"RQ1",target:"A5",type:"motivates",conf:.7},
{source:"RQ1",target:"F1",type:"addresses",conf:.8},
{source:"RQ1",target:"F2",type:"addresses",conf:.9},
{source:"RQ1",target:"F4",type:"addresses",conf:.85},
{source:"RQ1",target:"F5",type:"addresses",conf:.8},
{source:"RQ1",target:"F6",type:"addresses",conf:.6},
{source:"RQ1",target:"F7",type:"addresses",conf:.8},
{source:"RQ1",target:"F8",type:"addresses",conf:.9},
{source:"RQ1",target:"F9",type:"addresses",conf:.9},
{source:"A1",target:"F2",type:"arg0",conf:1},
{source:"A1",target:"F3",type:"arg1",conf:1},
{source:"A1",target:"F1",type:"arg1",conf:1},
{source:"A1",target:"F4",type:"arg1",conf:1},
{source:"A1",target:"F6",type:"arg0",conf:.8},
{source:"A1",target:"F8",type:"arg1",conf:.8},
{source:"A2",target:"F2",type:"arg1",conf:1},
{source:"A3",target:"F2",type:"arg0",conf:.9},
{source:"A3",target:"F5",type:"arg1",conf:1},
{source:"A4",target:"F7",type:"arg0",conf:.8},
{source:"A4",target:"F9",type:"arg1",conf:1},
{source:"EL1",target:"A1",type:"supports",conf:.9}
];
var colors={ResearchQuestion:"#fbbf24",Association:"#a78bfa",Factor:"#34d399",EvidenceLine:"#fb7185"};
var edgeColors={motivates:"#60a5fa",addresses:"#60a5fa",arg0:"#fbbf24",arg1:"#fbbf24",supports:"#34d399"};
var sizes={ResearchQuestion:15,Association:10,Factor:8,EvidenceLine:8};
var W=700,H=500;
var svg=d3.select("#fylo-graph-intro").append("svg").attr("viewBox","0 0 "+W+" "+H).attr("width","100%").attr("height","100%");
var g=svg.append("g");
svg.call(d3.zoom().scaleExtent([.5,4]).on("zoom",function(e){g.attr("transform",e.transform)}));
var link=g.append("g").selectAll("line").data(links).join("line").attr("stroke",function(d){return edgeColors[d.type]||"#64748b"}).attr("stroke-width",function(d){return d.conf*2}).attr("stroke-opacity",.5);
var node=g.append("g").selectAll("g").data(nodes).join("g").call(d3.drag().on("start",dragstart).on("drag",dragged).on("end",dragend));
node.append("circle").attr("r",function(d){return sizes[d.type]||8}).attr("fill",function(d){return colors[d.type]||"#64748b"}).attr("stroke","rgba(255,255,255,0.2)").attr("stroke-width",1.5);
node.append("text").text(function(d){return d.title.length>30?d.title.substring(0,30)+"...":d.title}).attr("dy",function(d){return(sizes[d.type]||8)+12}).attr("text-anchor","middle").attr("fill","#94a3b8").attr("font-size","7.5px").attr("font-family","IBM Plex Sans, system-ui, sans-serif");
var tooltip=d3.select("#fylo-graph-intro").append("div").style("position","absolute").style("display","none").style("background","rgba(30,41,59,.95)").style("border","1px solid rgba(255,255,255,.1)").style("border-radius","6px").style("padding","8px 12px").style("font-size","11px").style("color","#e2e8f0").style("font-family","IBM Plex Sans, system-ui, sans-serif").style("pointer-events","none").style("max-width","260px").style("z-index","10");
node.on("mouseover",function(e,d){tooltip.style("display","block").html("<strong style='color:"+colors[d.type]+"'>"+d.type+"</strong><br/>"+d.title)}).on("mousemove",function(e){var rect=document.getElementById("fylo-graph-intro").getBoundingClientRect();tooltip.style("left",(e.clientX-rect.left+12)+"px").style("top",(e.clientY-rect.top-20)+"px")}).on("mouseout",function(){tooltip.style("display","none")});
var sim=d3.forceSimulation(nodes).force("link",d3.forceLink(links).id(function(d){return d.id}).distance(55)).force("charge",d3.forceManyBody().strength(-180)).force("center",d3.forceCenter(W/2,H/2)).force("collision",d3.forceCollide().radius(28));
sim.on("tick",function(){link.attr("x1",function(d){return d.source.x}).attr("y1",function(d){return d.source.y}).attr("x2",function(d){return d.target.x}).attr("y2",function(d){return d.target.y});node.attr("transform",function(d){return"translate("+d.x+","+d.y+")"})});
function dragstart(e,d){if(!e.active)sim.alphaTarget(.3).restart();d.fx=d.x;d.fy=d.y}
function dragged(e,d){d.fx=e.x;d.fy=e.y}
function dragend(e,d){if(!e.active)sim.alphaTarget(0);d.fx=null;d.fy=null}
var legend=d3.select("#fylo-graph-intro").style("position","relative").append("div").style("position","absolute").style("bottom","8px").style("left","8px").style("display","flex").style("gap","10px").style("font-size","10px").style("font-family","IBM Plex Sans, system-ui, sans-serif");
Object.entries(colors).forEach(function(kv){legend.append("span").style("display","flex").style("align-items","center").style("gap","4px").style("color","#94a3b8").html("<span style='display:inline-block;width:8px;height:8px;border-radius:50%;background:"+kv[1]+"'></span>"+kv[0])})
})();
</script>

This post covers the three design decisions that mattered most: a ShEx schema as executable contract, a phased extraction pipeline that closes the validator-LLM loop, and a three-tier merge policy that handles cross-document identity correctly. The graph neural network and infrastructure architecture behind the related [striff.io code review system]({% post_url 2026-04-28-detecting-architectural-anomalies-gnn %}) follows a similar neurosymbolic staging pattern, described in a [companion infrastructure post]({% post_url 2026-04-28-striff-io-ml-infrastructure %}).

<!--more-->

---

## Schema as Executable Contract

Discourse graphs need a stricter schema than most domain models. Nodes carry semantic role (claim, evidence, hypothesis), strict cardinality on outgoing edges, and IRI references that must resolve within the same graph. An Association claim missing its `arg0` and `arg1` factors is structurally meaningless. A schema that lets these through is documentation, not validation.

We chose ShEx (Shape Expressions) over JSON Schema or pydantic because ShEx natively expresses graph-level constraints: cardinality across edges, IRI references, shape disjunctions. The schema lives in a single `.shex` file, is machine-checkable through `pyshex`, and is the single source of truth that everything else derives from.

<img src="/images/fylo-shex-association.svg" style="margin-left:auto; margin-right:auto; display: block;"/>

The schema participates in two places. A curated excerpt is injected into the LLM system prompt so the model sees the constraints before generating. After extraction, `pyshex`'s `ShExEvaluator` runs against the produced subgraph and rejects nodes that violate their shape definitions. The prompt-time injection biases the model toward valid output; the post-extraction validation catches the cases where the bias was insufficient.

Edge confidence scores are clamped to [0, 1] and rounded at the validator boundary. Different LLM calls produce different confidence ranges, and without normalization at this point downstream analytics build on numerically unstable signals.

ShEx handles structural constraints well but struggles with logical constraints across the graph, the kind that say "a hypothesis cannot be confirmed unless at least two non-overlapping evidence lines support it." The architecture I would build today layers ShEx as the cheap structural gate that runs every pass, with Z3 as the more expensive semantic gate that runs at canonicalization time.

---

## Closing the Validator-LLM Loop

<img src="/images/fylo-three-phase-extraction.svg" style="margin-left:auto; margin-right:auto; display: block;"/>

The standard approach has the LLM extract once and the validator either pass or fail the whole batch. The problem is that LLMs produce confident but malformed output, and re-running the same prompt with the same document reproduces the same errors.

The pipeline I shipped at Fylo decomposes a single chunk into three sequential extraction phases, each with a specialized prompt and scope. Phase 1 extracts entities only, with the prompt explicitly forbidding edge calls. This produces a clean inventory that can be validated against node-level shapes before any relational reasoning happens. Phase 2 extracts relations only, with the Phase 1 entities passed back as context. New nodes are permitted only when an edge would otherwise be dangling. Phase 3 repairs orphans: any node that ended up disconnected gets a final pass where the LLM is shown the dangling nodes specifically and asked to either form connecting associations or accept that they should drop.

This decomposition matters more than the prompts themselves. Phase 1 outputs are validated before Phase 2 starts, so Phase 2 produces edges guaranteed to reference real, validated nodes. Phase 3 addresses the failure mode where graphs become a pile of disconnected entities, by giving the model an explicit second chance with focused context.

<img src="/images/fylo-pipeline-overview.svg" style="margin-left:auto; margin-right:auto; display: block;"/>

The regex-based LLM output parser had to go. Regex cannot reliably handle nested parentheses inside string literals, and the old parser silently dropped well-formed extractions whenever a string contained an unescaped quote. The replacement uses `ast.literal_eval` rather than `eval` for safety, and ships with full test coverage.

The other change that mattered was tightening the schema prompt with explicit cardinality language. The Phase 1 prompt for Association nodes makes it unambiguous: every Association must include both an `arg0` and an `arg1` factor edge, or it will fail validation. This is the prompt-engineering equivalent of writing good error messages -- tell the model exactly what will go wrong, before it goes wrong.

---

## Cross-Document Canonicalization

The obvious approach to cross-document deduplication is one similarity threshold and one merge action. This breaks immediately on a real schema. The right behaviour for a Factor node like "Protein X" is aggressive merging across documents because the same protein in two papers is the same protein. The right behaviour for an Association claim is the opposite. Two papers asserting that "Protein X reduces inflammation" might be expressing the same association or two subtly different claims with different evidence. Merging them collapses signal you cannot recover.

<img src="/images/fylo-merge-policies.svg" style="margin-left:auto; margin-right:auto; display: block;"/>

The pipeline classifies node types into three merge policies. Aggressive types (factors, variables of interest, study designs, statistical methods) go through direct deduplication. When a new node arrives, its embedding is compared against role-filtered ANN candidates from the existing graph, and if the top candidate clears the threshold, the two nodes merge. Provenance is unioned, embeddings are weight-averaged based on accumulated confidence, and the lower-confidence node's ID is rewritten across all incident edges.

Cautious types (associations, hypotheses, research questions) use canonicalization rather than direct merging. Once enough cautious nodes of a given type accumulate, a synthetic canonical node is created with a `CANON_` prefix. Subsequent nodes either attach to the canonical via a `mentions` edge, or trigger creation of a new canonical when they represent a distinct cluster. The originals stay in the graph with their citations and confidence scores intact. The canonical becomes the queryable hub; the originals remain the evidence behind it. A small set of structural types are never merged at all, because doing so would lose per-paper provenance.

Role-filtered ANN search matters as much as the policy itself. A FAISS HNSW index returns candidates fast, but the candidate list will mix node types unless you filter at the search boundary. We restrict candidates to the same type as the query node before similarity scoring runs. Embedding maintenance under merge is the other non-obvious piece: the survivor's embedding is a weight-averaged combination of the source vectors, upserted back into the ANN index so subsequent comparisons see the consolidated state.

The test of cross-document canonicalization is whether the graph converges. A pipeline that adds papers and produces a graph that grows without becoming more connected is not building a knowledge graph.

---

## Making the Symbolic Layer Visible

Symbolic system improvements are abstract until you can see the graph. Telling a stakeholder that edge confidence scores are now properly normalized, or that cautious-type canonicalization is reducing duplicate hypotheses, communicates nothing. The same change rendered as a side-by-side visual diff communicates immediately.

<div id="fylo-graph-viz" style="margin:24px auto;max-width:700px;border:1px solid rgba(15,23,42,0.12);border-radius:12px;overflow:hidden;background:#0f172a;"></div>
<script>
(function(){
var nodes=[
{id:"RQ1",type:"ResearchQuestion",title:"How do bioelectric circuits control anatomical outcomes?"},
{id:"A1",type:"Association",title:"Gap junction proteins and ion channel expression regulate bioelectrical signals"},
{id:"A2",type:"Association",title:"Turing-like patterns arise from bioelectric mechanisms"},
{id:"A3",type:"Association",title:"Epithelial tissue geometry directs bioelectric field"},
{id:"A4",type:"Association",title:"Connexin43 regulates joint location in zebrafish fins"},
{id:"A5",type:"Association",title:"Calcium oscillations coordinate feather mesenchymal cell movement"},
{id:"F1",type:"Factor",title:"Morphogenetic outcomes"},
{id:"F2",type:"Factor",title:"TRPC6"},
{id:"F3",type:"Factor",title:"Membrane potential in mammalian cancer cells"},
{id:"F4",type:"Factor",title:"Gene expression domains"},
{id:"F5",type:"Factor",title:"Endogenous bioelectric gradients"},
{id:"F6",type:"Factor",title:"H+/K+-ATPase activity and membrane potential"},
{id:"F7",type:"Factor",title:"PI3K-mediated electrotaxis of neural progenitor cells"},
{id:"F8",type:"Factor",title:"Osteogenic and Chondrogenic Master Genes expression"},
{id:"F9",type:"Factor",title:"Size of zebrafish fins"},
{id:"EL1",type:"EvidenceLine",title:"hERG1 channel conformation impacts cancer progression"}
];
var links=[
{source:"RQ1",target:"A1",type:"motivates",conf:.9},
{source:"RQ1",target:"A2",type:"motivates",conf:.8},
{source:"RQ1",target:"A3",type:"motivates",conf:.9},
{source:"RQ1",target:"A4",type:"motivates",conf:.9},
{source:"RQ1",target:"A5",type:"motivates",conf:.7},
{source:"RQ1",target:"F1",type:"addresses",conf:.8},
{source:"RQ1",target:"F2",type:"addresses",conf:.9},
{source:"RQ1",target:"F4",type:"addresses",conf:.85},
{source:"RQ1",target:"F5",type:"addresses",conf:.8},
{source:"RQ1",target:"F6",type:"addresses",conf:.6},
{source:"RQ1",target:"F7",type:"addresses",conf:.8},
{source:"RQ1",target:"F8",type:"addresses",conf:.9},
{source:"RQ1",target:"F9",type:"addresses",conf:.9},
{source:"A1",target:"F2",type:"arg0",conf:1},
{source:"A1",target:"F3",type:"arg1",conf:1},
{source:"A1",target:"F1",type:"arg1",conf:1},
{source:"A1",target:"F4",type:"arg1",conf:1},
{source:"A1",target:"F6",type:"arg0",conf:.8},
{source:"A1",target:"F8",type:"arg1",conf:.8},
{source:"A2",target:"F2",type:"arg1",conf:1},
{source:"A3",target:"F2",type:"arg0",conf:.9},
{source:"A3",target:"F5",type:"arg1",conf:1},
{source:"A4",target:"F7",type:"arg0",conf:.8},
{source:"A4",target:"F9",type:"arg1",conf:1},
{source:"EL1",target:"A1",type:"supports",conf:.9}
];
var colors={ResearchQuestion:"#fbbf24",Association:"#a78bfa",Factor:"#34d399",EvidenceLine:"#fb7185"};
var edgeColors={motivates:"#60a5fa",addresses:"#60a5fa",arg0:"#fbbf24",arg1:"#fbbf24",supports:"#34d399"};
var sizes={ResearchQuestion:15,Association:10,Factor:8,EvidenceLine:8};
var W=700,H=500;
var svg=d3.select("#fylo-graph-viz").append("svg").attr("viewBox","0 0 "+W+" "+H).attr("width","100%").attr("height","100%");
var g=svg.append("g");
svg.call(d3.zoom().scaleExtent([.5,4]).on("zoom",function(e){g.attr("transform",e.transform)}));
var link=g.append("g").selectAll("line").data(links).join("line").attr("stroke",function(d){return edgeColors[d.type]||"#64748b"}).attr("stroke-width",function(d){return d.conf*2}).attr("stroke-opacity",.5);
var node=g.append("g").selectAll("g").data(nodes).join("g").call(d3.drag().on("start",dragstart).on("drag",dragged).on("end",dragend));
node.append("circle").attr("r",function(d){return sizes[d.type]||8}).attr("fill",function(d){return colors[d.type]||"#64748b"}).attr("stroke","rgba(255,255,255,0.2)").attr("stroke-width",1.5);
node.append("text").text(function(d){return d.title.length>30?d.title.substring(0,30)+"...":d.title}).attr("dy",function(d){return(sizes[d.type]||8)+12}).attr("text-anchor","middle").attr("fill","#94a3b8").attr("font-size","7.5px").attr("font-family","IBM Plex Sans, system-ui, sans-serif");
var tooltip=d3.select("#fylo-graph-viz").append("div").style("position","absolute").style("display","none").style("background","rgba(30,41,59,.95)").style("border","1px solid rgba(255,255,255,.1)").style("border-radius","6px").style("padding","8px 12px").style("font-size","11px").style("color","#e2e8f0").style("font-family","IBM Plex Sans, system-ui, sans-serif").style("pointer-events","none").style("max-width","260px").style("z-index","10");
node.on("mouseover",function(e,d){tooltip.style("display","block").html("<strong style='color:"+colors[d.type]+"'>"+d.type+"</strong><br/>"+d.title)}).on("mousemove",function(e){var rect=document.getElementById("fylo-graph-viz").getBoundingClientRect();tooltip.style("left",(e.clientX-rect.left+12)+"px").style("top",(e.clientY-rect.top-20)+"px")}).on("mouseout",function(){tooltip.style("display","none")});
var sim=d3.forceSimulation(nodes).force("link",d3.forceLink(links).id(function(d){return d.id}).distance(55)).force("charge",d3.forceManyBody().strength(-180)).force("center",d3.forceCenter(W/2,H/2)).force("collision",d3.forceCollide().radius(28));
sim.on("tick",function(){link.attr("x1",function(d){return d.source.x}).attr("y1",function(d){return d.source.y}).attr("x2",function(d){return d.target.x}).attr("y2",function(d){return d.target.y});node.attr("transform",function(d){return"translate("+d.x+","+d.y+")"})});
function dragstart(e,d){if(!e.active)sim.alphaTarget(.3).restart();d.fx=d.x;d.fy=d.y}
function dragged(e,d){d.fx=e.x;d.fy=e.y}
function dragend(e,d){if(!e.active)sim.alphaTarget(0);d.fx=null;d.fy=null}
var legend=d3.select("#fylo-graph-viz").style("position","relative").append("div").style("position","absolute").style("bottom","8px").style("left","8px").style("display","flex").style("gap","10px").style("font-size","10px").style("font-family","IBM Plex Sans, system-ui, sans-serif");
Object.entries(colors).forEach(function(kv){legend.append("span").style("display","flex").style("align-items","center").style("gap","4px").style("color","#94a3b8").html("<span style='display:inline-block;width:8px;height:8px;border-radius:50%;background:"+kv[1]+"'></span>"+kv[0])})
})();
</script>

This is a cropped neighbourhood from FyloVisualizer, a D3.js force-directed graph viewer I built to close that loop. The full viewer renders the complete discourse graph with 167 nodes across 4 types, edges weighted by confidence, hover for metadata and ontology tags, and filtering by semantic role. It loads graph JSON directly from the extraction pipeline output, no manual curation.

This pattern recurs across my work. The enriched SVG diagrams [striff.io](https://striff.io) renders for code architecture diffs serve the same purpose: they translate backend ML quality changes into something a non-technical user can immediately judge. In a neurosymbolic system where most of the engineering is invisible, the visualization layer is part of the production debugging surface.