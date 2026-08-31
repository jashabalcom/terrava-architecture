# ADR-0007: RAG over Bedrock Knowledge Bases instead of fine-tuning

**Status:** Accepted
**Date:** 2025
**Scope:** BedrockKBStack

## Context

A meaningful share of user questions are regulatory: visa thresholds, freehold eligibility,
transfer fees, ownership rules for foreign buyers, free-zone requirements. These have exact,
checkable answers that live in published documents — and they change.

A general-purpose model answering these from parametric memory will be confidently wrong some
of the time. In a financial-services-adjacent product, a confidently wrong regulatory answer
is the worst possible output: worse than "I don't know," because the user acts on it.

Two requirements followed: answers must be **grounded in source documents**, and they must be
**citable**, so the user can verify and the platform can defend the answer.

## Options considered

### Option A — Rely on the base model's knowledge

Zero infrastructure. Rejected outright. Training cutoffs, no citations, no way to update when
a regulation changes, and hallucination risk on precisely the questions where being wrong
matters most.

### Option B — Fine-tune a model on the regulatory corpus

Rejected on three grounds:

1. **Freshness.** Regulations change. Fine-tuning bakes knowledge into weights, so every
   change needs a retraining cycle — expensive, slow, and easy to defer until it's stale.
2. **No citations.** A fine-tuned model still generates from weights. It can't point at the
   paragraph it used, which fails the verifiability requirement.
3. **It doesn't fix hallucination.** Fine-tuning teaches style and format far more reliably
   than it teaches facts. The failure mode stays, it just sounds more authoritative.

### Option C — Stuff all documents into the context window

Simple, no retrieval infrastructure. Rejected: the corpus exceeds practical context limits,
paying for the full corpus on every request is the opposite of ADR-0005 and ADR-0006, and
answer quality degrades as irrelevant context grows.

### Option D — RAG over Bedrock Knowledge Bases + OpenSearch Serverless

Documents chunked and embedded with Titan Embeddings v2, stored as vectors in OpenSearch
Serverless. A query is embedded, the top-k most relevant chunks retrieved, and the model
generates an answer grounded in those chunks — with citations back to the source documents.

Updating knowledge means re-syncing a document, not retraining a model.

## Decision

**RAG: Titan Embeddings v2 → OpenSearch Serverless vector store → top-5 retrieval → grounded
generation with citations**, managed through Bedrock Knowledge Bases.

Retrieval-backed answers are routed to Haiku (ADR-0005) — once the correct source text is in
context, the task is comprehension and formatting, not deep reasoning, so the expensive model
earns nothing here.

## Consequences

**What got better**

- Answers cite their sources. Users can verify, and the platform can show its work.
- Updating knowledge is a document sync, not a retraining run — a regulatory change is live in
  minutes.
- Hallucination on regulatory questions drops sharply, because the model is summarizing
  supplied text rather than recalling it.
- Bedrock Knowledge Bases handles chunking, embedding, and sync orchestration, so there's no
  bespoke ingestion pipeline to maintain.
- Cheap model + retrieved context beats expensive model + no context, on both cost and
  accuracy.

**What got worse**

- **OpenSearch Serverless has a minimum OCU cost.** It is the most expensive component
  relative to its traffic — a real fixed cost for a low-volume feature, and the single
  weakest line in the cost model.
- **Retrieval quality is now the bottleneck.** If the right chunk isn't retrieved, the model
  answers from a bad context and sounds just as confident. Chunk size, overlap, and k are
  tuning parameters with no obvious correct value.
- **Garbage in, cited garbage out.** An outdated document in the corpus produces a wrong
  answer *with a citation*, which is more persuasive and therefore more dangerous than an
  uncited one. Corpus curation is an ongoing obligation, not a setup step.
- **Added latency** — embed, retrieve, then generate.
- **Sync is a job that can fail silently**, leaving the index stale while everything looks
  healthy.

## Revisit when

- OpenSearch Serverless minimum cost outgrows the feature's value — a smaller vector store, or
  pgvector on the Aurora cluster that's already running, becomes the cheaper answer.
- Retrieval misses show up in sampled review often enough to need hybrid (keyword + vector)
  search or a reranking stage.
- The corpus grows past what naive top-k retrieval handles well and needs metadata filtering
  by market and document type.
