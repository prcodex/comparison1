# Single-Vector + Compensations — A Deliberate Alternative to Universal Chunking

> **Where this fits.** The main repo [README](../README.md) compares the *current* M3xA
> retrieval model against the *proposed* contextual-v5 architecture (universal hierarchical
> chunking + per-chunk contextual prefixes). The companion [chunking_study.md](../chunking_study.md)
> argues that v5's universal chunking is over-applied. This directory proposes **the third
> option**: keep single-vector-per-document as the default and *compensate* for its known
> weaknesses with a deliberately-designed stack — hybrid retrieval, strong reranking,
> Haiku-extracted tags/entities, a doc-level contextual prefix, and editorial multipliers.
> Chunking is reserved for the two source classes where it's genuinely mandatory (long PDFs,
> podcast transcripts).
>
> **Why this is a real architectural option, not a stopgap.** Two strands of 2025–2026
> evidence have converged: (1) on attribution- and synthesis-heavy query mixes (which is
> what M3xA actually serves), hybrid + strong reranker recovers most of the lift that
> chunking provides; (2) rule-based chunking introduces noise on documents with inconsistent
> structure (no reliable markdown headers, variable cleaning quality, topics shifting
> mid-paragraph) — and most ingested news/research documents *are* like that. Together these
> tilt the cost/quality curve in single-vector's favor for the bulk of the corpus.

---

## TL;DR

- **Default chunking strategy: one vector per document.** Long-context embedders (Voyage-3-large
  32K, Cohere v4 128K) make this viable for documents up to ~8K tokens; the predictable
  "blurry centroid" failure mode is easier to debug and compensate for than the unpredictable
  noise that rule-based chunking introduces on unstructured documents.
- **Five compensations stacked at retrieval time:** (1) hybrid BM25+dense+RRF, (2) strong
  cross-encoder reranker, (3) Haiku-extracted tags/entities as a separate BM25 leg,
  (4) doc-level Haiku contextual prefix, (5) source-class / entity / recency multipliers at
  the rerank stage. Each compensation is independently supported by 2025–2026 evidence; the
  stack is the converging production pattern.
- **Two narrow opt-ins for the classes where single-vector demonstrably breaks:** late
  chunking (Jina-style) for documents > 4K tokens; ColPali / page-image embeddings for PDFs
  with charts/tables. These are the only places this architecture concedes chunking is
  needed.
- **What this avoids:** universal per-chunk Haiku contextual prefixing (the most expensive
  line item in the v5 spec), the chunk-boundary policy debates that come with rule-based
  chunking on unstructured text, and the debugging surface area of multi-chunk dedup.
- **What this concedes:** true needle-in-haystack span retrieval inside a 20K-token PDF (the
  ColPali / late-chunking opt-ins handle this for the classes where it matters), and
  span-level citation in the UX (single-vector returns the document, not the sentence).

---

## 1 · Why this architecture exists

Two observations drove the design:

### 1.1 The documents have unreliable structure

The "snap to `##` header or `\n\n` or `.`" rule that hierarchical chunking depends on works
beautifully on well-structured documents. It works **badly** on much of what the system
actually ingests:

- Email research that arrives as one long HTML blob with no semantic structure
- Podcast transcripts where topic shifts don't align with speaker turns
- PDFs where text extraction collapses sections into walls of text
- Anything where the upstream cleaning pass produces inconsistent markdown

When the rule fires in the middle of a thought, the resulting chunk starts with
*"...and that's why we expect rates to..."* — a fragment that embeds badly and pollutes
search results. The blurry-centroid problem of single-vector is at least a *predictable*
failure mode; bad chunking is an *unpredictable* one.

### 1.2 The query mix doesn't favor chunking

The dominant query patterns in production traffic are:

- **Named-source attribution** — "what does Bank N say about US economy"
- **Time-windowed multi-source synthesis** — "last 24h on Iran"
- **Thematic synthesis** — "top 10 macro themes this week"
- **Day-over-day comparison** — "what's new vs yesterday"

These query types want **diversity across documents**, not depth within them. Chunking
optimizes for the opposite — span-level retrieval inside long documents — which is a
minority of actual traffic. For the dominant patterns, single-vector with the right
retrieval-time compensations is competitive with or better than chunked approaches.

### 1.3 The 2025–2026 evidence has been moving in this direction

Independent reproductions through 2025 found that **hybrid retrieval + a strong cross-
encoder reranker** captures the majority of the lift attributed to chunking + contextual
retrieval in headline benchmarks — at a fraction of the ingest complexity and cost. The
"chunk everything" default that dominated 2023–2024 RAG advice has softened as long-context
embedders and stronger rerankers entered general availability.

See [evidence.md](evidence.md) for the per-compensation literature review.

---

## 2 · The architecture in one diagram

```
INGEST (per document, runs at scrape time)
══════════════════════════════════════════

Raw source
  │
  ▼
(1) Haiku CLEAN
    (existing pattern — see ../haiku_cleaning.md)
  │
  ▼
(2) Haiku TAGS + DOC-LEVEL PREFIX   ──── one call, two outputs
    Output A: structured JSON {entities, topics, themes, geographies}
    Output B: 100-200 tok prose prefix "this document is..."
  │
  ▼
(3) Embed (prefix + body) → ONE doc vector  (Voyage / Cohere v4)
  │
  ▼
Store in vector DB:
  - vector
  - body text (cleaned)
  - contextual_prefix
  - tags JSON
  - entities array
  - source metadata (date, source_id, domain, priority, etc.)

[OPT-IN for docs > 4K tokens]
(4) Late chunking via Jina-style windows → N additional sub-doc vectors
    (only if the embedder exposes token-level outputs)

[OPT-IN for PDFs only]
(5) Render each page as image → embed via Cohere v4 image mode
    → page-image vectors


QUERY (per question)
════════════════════

User query
  │
  ▼
3 parallel retrieval legs:
  ├─► dense vector on doc vectors            → top 100
  ├─► BM25 on body text                      → top 100
  └─► BM25 on tags + entities (boost 1.5-2x) → top 100   ◄── the key new leg
  │
  ▼
RRF fusion (k=60) → top 150
  │
  ▼
Cross-encoder rerank (Cohere Rerank 3.5 or Voyage rerank-2.5) → top 40
  │
  ▼
Editorial multipliers
  (source weight × recency decay × entity floor)
  → top 30
  │
  ▼
Synthesizer LLM (reads doc bodies; for opt-in cases, reads the
                 winning sub-doc / page-image with the parent doc)
```

---

## 3 · The five compensations

A summary table; see [evidence.md](evidence.md) for the literature review and per-item lift
magnitudes, and [implementation.md](implementation.md) for code-level detail.

| Compensation | What it fixes | Cost | Evidence strength |
|---|---|---|---|
| **(1) Hybrid (BM25 + dense + RRF)** | Proper nouns, tickers, person names that the averaged dense vector blurs | ~$0 (BM25 is free) | **Strong** — multiple reproductions, ~+15–30% nDCG@10 lift on news/finance corpora |
| **(2) Strong cross-encoder reranker over top-100** | Imprecise dense match → reranker re-reads candidates carefully | ~$0.0003/query | **Strong** — +8–20 nDCG@10 lift on news/finance reranking subsets |
| **(3) Haiku-extracted tags / entities as separate BM25 leg** | Provides a high-precision lexical-match channel; bridges queries that use different vocabulary than the document body | ~$0.001/doc (folded into (4)) | **Medium** — production reports + extrapolation from Anthropic's per-chunk contextual retrieval |
| **(4) Doc-level Haiku contextual prefix** | Pulls the centroid toward the document's named topics rather than averaging noise | ~$0.001/doc (same Haiku call as (3)) | **Medium** — extrapolation from Anthropic's per-chunk pattern; direction strong, magnitude unbenchmarked at doc level |
| **(5) Source-class / entity / recency multipliers at rerank** | Forces the right *kind* of source to surface; the only thing that gets the right doc to rank 1 for attribution queries | ~$0 (multipliers) | **Strong in production, weak in academic benchmarks** — domain-specific; existing scoring v3 in M3xA already implements this |

**The merge insight.** Compensations (3) and (4) are the same Haiku call at different
granularities — one returns structured JSON tags, the other returns a prose prefix using
those tags. Implement as a single Haiku call per document with two outputs. See
[implementation.md §2.2](implementation.md#22-haiku-tag-extraction--doc-level-prefix-one-call).

---

## 4 · Two narrow opt-ins

This architecture concedes that **two specific document classes genuinely need chunking**.
Both are opt-ins layered on top of the single-vector default — they do not replace it.

### 4.1 Late chunking for documents > 4K tokens

For the small fraction of documents long enough that single-vector demonstrably degrades
(long-form journalism over 4K tokens, long email research notes, op-eds), add a **late-
chunking** pass: encode the whole document once with the long-context embedder, then mean-
pool token embeddings over fixed-size windows (no smart-boundary detection needed). Each
sub-document vector carries cross-window context for free. Store these as additional rows
keyed to the same `doc_id`.

This is opt-in by document length, not by source class. ~10–20% of the corpus enters this
path. See [implementation.md §4](implementation.md#4-late-chunking-opt-in-for-docs--4k-tokens).

### 4.2 ColPali page-image vectors for PDFs

PDFs are the one class where single-vector + text-only retrieval fundamentally fails on
chart/table queries — text extraction destroys the visual content. The fix is to render
each page as an image and embed via the multimodal embedder (Cohere v4 image mode), storing
page-image vectors alongside the text vectors.

This is opt-in by source class (PDFs only), not by length. See
[implementation.md §5](implementation.md#5-colpali-opt-in-for-pdfs).

---

## 5 · When this architecture is the right choice

Use this architecture when:

- Documents have inconsistent structure (no reliable markdown headers, variable upstream
  cleaning quality).
- Queries are dominated by attribution, recency, synthesis, or thematic retrieval — not
  span-level fact lookup.
- The system can absorb the cost of a strong cross-encoder reranker on every query.
- A long-context embedder (Voyage-3-large, Cohere v4) is available.
- Engineering effort is constrained — debugging single-vector + compensations is materially
  cheaper than debugging hierarchical chunking + parent/child mapping bugs.

Don't use this architecture when:

- Documents have reliable structure (legal contracts with numbered clauses, structured
  reports with consistent sections) — hierarchical chunking is cleaner there.
- Queries are dominated by needle-in-haystack span retrieval inside long documents.
- Span-level citation in the UX is required (highlight the supporting sentence, not the
  document).

For M3xA's actual corpus and query mix, the "use this" conditions all hold; the "don't use"
conditions are concentrated in the two classes the opt-ins (§4) address.

---

## 6 · Honest losing scenarios

Three places this architecture still underperforms full hierarchical chunking, called out
directly so the trade-off is explicit:

1. **True needle-in-haystack inside a 20K-token PDF.** The single document-level vector
   cannot surface *"the one sentence on page 14 about Argentine wheat exports."* Late
   chunking (the §4.1 opt-in) helps; nothing else does cleanly.
2. **Multi-hop within a single long document.** Retrieving the document isn't enough; the
   reranker can't simultaneously surface two disjoint spans from one document for the LLM.
   Needs either long-context synthesis or in-document span extraction as a second stage.
3. **Span-level citation in the UX.** If you want to highlight the supporting sentence in
   the answer rather than cite the document, single-vector forces span extraction at
   generation time, which is a separate engineering cost.

For attribution-shaped queries (which dominate M3xA traffic), these are minority cases. For
PDFs specifically they bite hard enough to justify the page-image opt-in (§4.2) and the
late-chunking opt-in (§4.1).

---

## 7 · How this relates to the other repo files

- [../README.md](../README.md) — the high-level "current vs proposed v5" comparison.
- [../chunking_study.md](../chunking_study.md) — the document-and-query-specific study of
  single-vector vs multi-chunk. §11 originally recommended a two-tier index hedged toward
  chunking; this directory is the cleaner statement of the architecture after further
  evaluation and the 2026 evidence review.
- [../haiku_cleaning.md](../haiku_cleaning.md) — the per-document Haiku cleaning pass
  already in production. This architecture extends that pattern with a second Haiku call per
  document for tags + prefix.
- [../for_beginners/hierarchical_and_contextual.md](../for_beginners/hierarchical_and_contextual.md)
  — beginner walkthrough of the *chunking* alternative. Read that first if the terms in this
  document are unfamiliar.

---

## 8 · Files in this directory

- [README.md](README.md) (this file) — overview, motivation, architecture, when to use, what
  it concedes.
- [implementation.md](implementation.md) — concrete how-to: schema, Haiku prompts, code
  skeletons for the ingest pipeline, the 3-leg retrieval, the reranker integration, the
  opt-ins, and the migration path from the existing system.
- [evidence.md](evidence.md) — June 2026 literature review per compensation: mechanism,
  evidence quality, lift magnitudes, citations, and the honest gaps in the evidence.
- [phase0_eval.md](phase0_eval.md) — proposed golden-set evaluation that would settle the
  question definitively: single-vector + compensations vs full hierarchical chunking vs late
  chunking on the same query set, scored on relevance@10, MRR, attribution accuracy, and
  per-source-class breakdowns.
