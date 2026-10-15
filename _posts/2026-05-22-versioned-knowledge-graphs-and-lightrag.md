---
layout: post
title: Content-addressed document sets and LightRAG in production
description: Building versioned knowledge graphs, cutting LightRAG query latency in half, and visualizing retrieval heat maps.
summary:
tags: [RAG, Knowledge-Graphs, AI]
---
Most RAG pipelines rot the moment someone edits a document.

In robotics operations, teams rely on Standard Operating Procedures (SOPs), hardware manuals, and incident playbooks. Those documents change constantly. If an AI agent's answer changes from Tuesday to Thursday, or an evaluation score drops by 15%, you need to know whether the model drifted, the prompt changed, or someone quietly rewrote `battery-fault-triage.md`.

When I built the knowledge graph and RAG stack in Cortex, the goal was to make knowledge retrieval reproducible, observable, and fast enough for live agent loops.

### Versioned, content-addressed document sets

Instead of pointing a vector store at a mutable folder of files, I built `DocumentSet` as a versioned primitive backed by content-addressed blobs in MinIO.

Every file is hashed on upload and stored by its content digest. A `DocumentSetVersion` is an immutable manifest mapping `(path → blob_digest)` with composite `(org_id, id)` foreign keys enforcing tenant isolation at the database layer.

<figure class="diagram-figure">
  <a href="/assets/diagrams/lightrag-pipeline.html" target="_blank">
    <img class="diagram-light" src="/assets/diagrams/lightrag-pipeline.light.png" alt="Versioned Document Sets and LightRAG Pipeline">
    <img class="diagram-dark" src="/assets/diagrams/lightrag-pipeline.dark.png" alt="Versioned Document Sets and LightRAG Pipeline">
  </a>
  <figcaption>
    <span>Data flow from versioned DocumentSet manifests and content-addressed MinIO blobs through LightRAG streaming.</span>
    <a href="/assets/diagrams/lightrag-pipeline.html" target="_blank">Open interactive diagram ↗</a>
  </figcaption>
</figure>

Because identical files share the same blob across versions, storage cost scales linearly with *changed* files rather than version count. More importantly, downstream consumers — LightRAG knowledge graph builds, workflow nodes, and evaluation datasets — pin to a specific `(documentSetId, versionId)`. When a document set updates, we run a delta sync against the previous manifest instead of rebuilding the entire knowledge graph from scratch.

### Cutting uncached LightRAG latency in half

We integrated [LightRAG](https://github.com/HKUDS/LightRAG) to combine graph entity traversal with vector retrieval (`local`, `global`, `hybrid`, and `mix` modes), exposing it as both an MCP tool (`rag_query`) and a native workflow node (`rag-retrieve`).

In production, uncached semantic queries on the Knowledge Graph console were taking **36 to 45 seconds**. Digging into the spans and logs showed two things:

1. Context-only retrieval in LightRAG took `~0.4–0.8s`, and cached hybrid queries took `~0.95s`, but an uncached `hybrid` synthesis spent `~36s` between keyword extraction and LLM answer generation.
2. Worse, our proxy service was making **two** separate calls to LightRAG for every query: one to generate the answer, and a second context-only call just to extract the participating graph entities so the UI could highlight them.

Routing the query through LightRAG's `/query/stream` endpoint with `includeReferences: true` returned the streamed answer and the referenced entities in a single round-trip. That eliminated the duplicate call (and a fragile JSON-parsing step on the second payload) and cut end-to-end query latency nearly in half.

### Visualizing retrieval with a decay heat-map

Once a knowledge graph has a few hundred entities and edges, a static node-link diagram doesn't tell you whether your agents are actually using it well. Some regions of the graph get hit on every incident; others are dead weight from a manual nobody reads.

I added an opt-in heat-map overlay to the knowledge graph canvas. Every query updates a heat score on both the retrieved nodes *and* the edges connecting two participating entities, with a soft exponential decay over time.

Because retrieval distributions are heavily skewed — a handful of hub entities get pulled into almost every context window — a linear color scale makes 95% of the graph look cold. Adding a gamma sensitivity curve (`display = Math.pow(heat, gamma)`, defaulting to `gamma = 0.5`) compressed the dynamic range so mid-traffic subgraphs glow clearly without washing out the hot spots.

![Knowledge graph retrieval heat-map overlay with sensitivity slider](https://github.com/user-attachments/assets/f5038da5-7159-4838-b858-6e740b682553)
