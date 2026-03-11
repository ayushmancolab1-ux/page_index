# AGENTS.md

## Purpose

This repository should evolve toward a production-grade, 3-layer retrieval architecture instead of relying on PageIndex hierarchy alone as the full retrieval system.

The target design is:

1. Document routing
2. Structured retrieval inside shortlisted documents
3. Fine retrieval plus reranking for answer grounding

Use this file as the default implementation guide for agents working in this repo.

## Core Direction

Treat PageIndex as a retrieval feature, not the entire retrieval stack.

PageIndex remains valuable for:

- section-aware routing
- page-range narrowing
- section summaries
- citation span quality
- long-form structured documents such as policies, legal docs, manuals, and BRDs

PageIndex should not be the only mechanism for:

- first-pass multi-document search
- large-corpus top-k routing
- low-latency enterprise retrieval
- broad corpus filtering before LLM reasoning

## Target Retrieval Architecture

### Layer 1: Document Routing

Goal: reduce the full corpus to a small document shortlist before any section-level reasoning.

This layer should use a search index over:

- document title
- document summary
- document metadata
- key section titles
- embeddings
- ACL or authorization metadata when required

Preferred retrieval mode:

- hybrid retrieval: lexical plus vector plus metadata filters

Expected output:

- shortlist of roughly 10 to 30 candidate documents

Implementation guidance:

- do not run section-level retrieval across every document in the corpus
- do not let the answer LLM act as the primary document router
- preserve explainability for why a document was shortlisted

### Layer 2: Structured Retrieval

Goal: use the PageIndex hierarchy to narrow each shortlisted document from document level to section level.

Each indexed document should maintain:

- section tree
- section summaries
- page ranges
- parent-child hierarchy
- optional section embeddings

Expected behavior:

- route from document to chapter to section
- prefer leaf or specific sections when possible
- keep parent context available for coherence and citation quality

Implementation guidance:

- PageIndex should drive section-aware narrowing
- section selection should happen before chunk-level answer grounding
- section-level outputs must stay traceable to page ranges and node IDs

### Layer 3: Fine Retrieval and Reranking

Goal: retrieve only the best snippets from the shortlisted sections and rerank them before answer generation.

This layer should use:

- chunk or snippet retrieval inside selected sections
- vector, lexical, or hybrid matching at chunk level
- a reranker for final relevance ordering

Expected output:

- small set of highly relevant grounded snippets for the answer prompt

Implementation guidance:

- do not always send full sections to the answer model
- prefer exact snippets with page-scoped citations
- reranking happens after retrieval narrowing, not before

## End-to-End Runtime Pattern

The desired runtime flow is:

User Query
-> Query understanding or decomposition
-> Document shortlist
-> Section shortlist inside shortlisted documents
-> Chunk or snippet retrieval inside shortlisted sections
-> Reranker
-> Answer generation with citations
-> Evaluation, tracing, and feedback loop

For harder questions, agentic retrieval is allowed:

- decompose the question into focused subqueries
- run subqueries in parallel when useful
- synthesize across retrieved evidence only after retrieval converges

## Production Rule of Thumb

Use:

- metadata and lexical search for precision filters
- vector search for semantic matching
- hierarchical structure for coherence
- reranking for final relevance
- LLM reasoning only after retrieval has been narrowed

This architecture is preferred because it improves:

- latency
- cost
- recall
- groundedness
- context control
- enterprise operability

## Azure-First Recommendation

For this repo, the preferred production stack is:

- Azure AI Search for hybrid retrieval and document routing
- Blob Storage for source files and raw artifacts
- Postgres, SQL, Cosmos DB, or equivalent for metadata and lineage
- PageIndex for section hierarchy and citation-aware narrowing
- LLMs for query decomposition, answer synthesis, and fallback reasoning
- LangGraph only when multi-step orchestration or human-in-the-loop flow is justified

If the corpus is still modest, start with:

- document summaries
- section tree
- chunk embeddings
- hybrid retrieval
- reranking
- answer LLM

Add multi-agent orchestration only when it solves a real retrieval problem.

## Guardrails

All implementations should preserve these controls:

- access control before retrieval
- grounding checks after generation
- explicit low-confidence or insufficient-evidence fallback
- traceability from answer claim to source snippet to page to document
- stage-level latency and token accounting

If retrieval confidence is low, prefer an insufficient-evidence answer over speculative synthesis.

## Observability Requirements

Every meaningful retrieval change should make these easier to measure:

- document routing hit rate
- section routing accuracy
- reranker impact
- citation coverage
- groundedness
- answer completeness
- latency per stage
- token cost per stage

Expose or preserve trace artifacts wherever practical.

## Phased Rollout Plan

### Phase 1: Hybrid RAG Foundation

Build:

- document metadata index
- chunking and embeddings
- hybrid retrieval
- reranking
- citations
- offline evaluation set

### Phase 2: Section-Aware Retrieval

Add:

- PageIndex-style section hierarchy
- retrieve by section first, chunk second
- better answer coherence
- better citation quality

### Phase 3: Agentic Retrieval

Add:

- query decomposition
- parallel subqueries
- synthesis across multiple documents or sources
- durable orchestration only where needed

## How This Maps to the Current Repo

Current repo strengths:

- background ingestion and tree generation already exist in `pageindex_backend/main.py`
- PageIndex section hierarchy already exists in `pageindex/`
- hierarchical parent-child retrieval already exists in `streamlit_app/rag/pipeline.py`
- answer tracing and citations already exist in the Streamlit RAG flow

Current gap relative to target architecture:

- document routing is still too thin
- retrieval is still centered around already selected documents
- no dedicated hybrid search layer exists yet for corpus-scale shortlist generation
- reranking is limited compared with a dedicated retrieval stack

## Preferred Next Implementation Steps

Agents making architectural improvements should prefer this sequence:

1. Introduce a document-level index and shortlist stage before section retrieval.
2. Separate document, section, and chunk retrieval data models explicitly.
3. Add hybrid retrieval interfaces instead of binding retrieval directly to one tree walk.
4. Add reranking over shortlisted chunk candidates before answer generation.
5. Preserve PageIndex as the section-structure layer, not the only retrieval layer.
6. Extend observability so each retrieval stage has measurable quality and latency.

## Coding Guidance for Agents

When modifying this repo:

- prefer retrieval narrowing over larger prompts
- preserve citations and page-level evidence
- avoid designs that require the LLM to scan entire documents unnecessarily
- keep retrieval stages separable and inspectable
- make routing decisions debuggable in the UI and traces
- preserve existing background-processing behavior unless the task explicitly changes it
- do not remove the current PageIndex hierarchy; reposition it within a broader retrieval stack

## Blunt Recommendation

The preferred production direction for this repo is:

Hybrid RAG plus Section Tree plus Reranker plus Evaluation Loop

Not:

PageIndex-only RAG

That is the default architectural assumption future agents should follow unless the user explicitly requests a different tradeoff.
