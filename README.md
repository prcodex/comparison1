# M3xA: Current Model vs. Contextual v5 vs. Single-Vector Plus — Comparison

> **Read this first.** There are now **three candidate architectures** in play. The first is
> *live and measured* — what M3xA actually runs today. The second is *spec and unbuilt* —
> the v5 target (universal hierarchical chunking + per-chunk contextual prefixes), explicitly
> "design spec, not yet implemented." The third is *proposed alternative, unbuilt* —
> Single-Vector Plus (single-vector-per-document as the default + retrieval-time
> compensations + narrow opt-ins where chunking is mandatory), the recommendation that
> crystallized after a deeper June 2026 evidence review (see
> [single_vector_plus/](single_vector_plus/)). Every number in the two right columns is a
> *claim to be proven*, not an observed result. Phase 0's golden set is the thing that turns
> both right columns from claims into measurements.

---

## 0 · Where we are

| Architecture | Status | Effort to ship | Steady-state cost | Conceptual shape |
|---|---|---|---|---|
| **Current M3xA** | ✅ LIVE in production | n/a — running | Voyage vendor pricing | One vector per document, no BM25, no rerank, no contextual layer |
| **Contextual v5** (`chunking_vector`) | 📋 SPEC, not built | ~10–12 weeks across 7 phases | ~$95/mo + ~$360 one-time backfill | Universal hierarchical chunking + per-chunk Haiku contextual prefixes + multimodal layers |
| **Single-Vector Plus** ([single_vector_plus/](single_vector_plus/)) | 📋 PROPOSAL, not built | ~3–4 weeks across 6 phases | ~$53/mo + ~$50 one-time backfill | One vector per document by default + hybrid (BM25+dense+tags) + cross-encoder rerank + doc-level Haiku contextual prefix; narrow chunking opt-ins (late chunking for docs >4K tok, page-image embeddings for PDFs) |

**Current recommendation:** ship Single-Vector Plus. It's ~45% cheaper to operate than v5,
~3× faster to deliver, and avoids the rule-based-chunking-introduces-noise risk that v5
takes on across the bulk of the corpus. The two opt-ins handle the specific document classes
(long-form > 4K tok, PDFs with charts) where single-vector demonstrably underperforms,
without imposing chunking universally. The v5 spec remains a valid option if Phase 0 eval
shows it materially beating Single-Vector Plus on the M3xA query mix — but the burden of
proof has shifted onto v5 to demonstrate the extra cost and complexity pay off.

**Critical caveat:** none of this is settled without measurement. See
[single_vector_plus/phase0_eval.md](single_vector_plus/phase0_eval.md) for the 200-query
golden-set evaluation that compares all four configurations
(baseline / Single-Vector Plus / Contextual v5 / late chunking) on the same queries.

---

## 1 · At a glance

| Dimension | Current M3xA (live) | Contextual v5 (spec) | Single-Vector Plus (proposed) |
|---|---|---|---|
| **Text embedder** | Voyage-3-large @ 2048d, sole embedder, vendor key | Cohere Embed v4 @ 1536d (Matryoshka), Bedrock-native | Voyage-3-large @ 2048d or Cohere v4 — no embedder change required |
| **Multilingual** | EN + PT handled separately | One unified EN/PT/AR space — cross-lingual by default | Vector multilingual depends on embedder choice; BM25 stays language-split (en_stem + pt_stem) regardless |
| **Chunking** | ~1 vector per document for everything | Per-source-class matrix; hierarchical parent 2000 / child 400–512 | **One vector per document by default;** late-chunking opt-in for docs > 4K tok; page-image vectors opt-in for PDFs |
| **Podcasts** | 1 vector per *whole episode* (a retrieval black hole) | Text chunks (~600 tok, timestamped) **+** parallel Nova audio @ 3072d | Text chunks (~600 tok, speaker-boundary aware) + 1 episode-level vector; **no audio layer** (deferred) |
| **Bank PDFs** | 1 vector per whole PDF | Triple-layer: doc vector + text children + page-image vector/page | **Two-layer:** 1 doc-vector + page-image vectors per page; **no text-children layer** (the doc-vector handles attribution, page-images handle charts) |
| **Contextual captions** | None | Universal Haiku prefix on **every chunk**, before embed *and* BM25 | **Doc-level Haiku prefix** (once per document, same call as tag extraction) — same mechanism applied at doc granularity instead of chunk granularity |
| **Tag / entity extraction** | None | Implicit in per-chunk contextual prefixes | **Explicit:** structured JSON of entities + topic tags + themes per document, indexed as separate BM25 field with 1.5–2× boost |
| **Keyword search (BM25)** | Planned, not built | en_stem + pt_stem Tantivy over caption + chunk text | **Two indexes per language:** body text + tags-and-entities field (with boost) |
| **Reranking** | Not wired | Cohere Rerank 3.5, top-150 → top-30 | Cohere Rerank 3.5 (or Voyage rerank-2.5), top-150 → top-40 |
| **Multimodal** | None (text extraction only) | PDF page-images (ViDoRe pattern) + podcast audio | PDF page-images (ViDoRe pattern) only; no audio Nova layer |
| **Storage** | Single `unified_feed` table | `unified_v5_text` + `unified_v5_pdf_images` + `unified_v5_audio` | Single `unified_v_doc` table (+ sibling `unified_v_doc_late_chunks` and `unified_v_pdf_pages` for opt-ins) |
| **Retrieval** | Vector search + metadata prefilter, top-k | Rewrite → cache → parallel multi-index → RRF → rerank → parent expansion | 3 parallel legs (dense + BM25 body + BM25 tags) → RRF (k=60) → rerank → editorial multipliers → top-30 |
| **Steady-state cost** | Voyage vendor pricing | ~$95/mo (+ ~$360 one-time backfill) | **~$53/mo** (+ ~$50 one-time backfill) — driven by eliminating the per-chunk Haiku prefix |
| **Expected retrieval lift** | baseline | −35% failures (caption) → −49% (+BM25) → −67% (+rerank), *claimed* | Parity with v5 on attribution + recency + synthesis (the dominant query classes); narrower gap on multi-hop and needle-in-PDF (which the opt-ins address), *claimed* |
| **Engineering risk** | n/a — running | High: 7 phases, chunker complexity, per-chunk prefix consistency, parent/child mapping bugs | Moderate: 6 phases, smaller surface area, no chunk-boundary policy decisions on bulk corpus, easier rollback |

**Three conceptual shapes, side by side:**

- **Current M3xA**: every document collapses to one averaged point; the averaged point IS the search target. No compensation. Misses on proper nouns and specific spans.
- **Contextual v5**: every long document is broken into small self-describing chunks; search hits small, returns big. Universal application — every chunk pays for an LLM prefix call.
- **Single-Vector Plus**: every document collapses to one averaged point *plus a doc-level LLM-generated context prefix*, *plus a separate BM25-indexable tag field*; the search compensates via hybrid + reranker instead of finer-granularity vectors. Chunking opt-ins only where single-vector demonstrably breaks.

---

## 2 · Per-source impact inside M3xA macro

The gain is deliberately *uneven* — it concentrates where macro is heaviest (bank research,
podcasts) and barely touches what was already small and findable (tweets).

| Macro source | Current | Contextual v5 | Single-Vector Plus |
|---|---|---|---|
| Macro tweets (~130 accts, 30–80 tok) | 1 Voyage vector | 1 vector **+ caption** | 1 vector + doc-level prefix + tags |
| Wire articles (400–2000 tok) | 1 vector | 1 vector if short; children if >1500 tok | 1 vector + doc-level prefix + tags |
| Long-form journalism (3K–8K) | 1 averaged vector | Hierarchical, section-aware | 1 vector + doc-level prefix + tags **+ late chunking (opt-in) if >4K tok** |
| Email research notes (1.5K–6K) | 1 vector | Section-split into 3–15 captioned chunks | 1 vector + doc-level prefix + tags |
| Bank / Drive PDFs (5K–25K) | 1 vector per PDF | **Triple-layer** doc + text + page-image | **Two-layer:** 1 doc-vector + page-image vectors (no text-children) |
| Podcast transcripts (5K–25K) | 1 vector per episode | Text chunks + parallel Nova audio | Text chunks at speaker boundaries + 1 episode vector (no audio layer) |
| Geo OSINT structured | 1 vector / article-style | Short = 1 vec; long = article-style | 1 vector + doc-level prefix + tags |
| Calendar / markets / Polymarket | DB rows, time-joined | **Unchanged** — never enters the vector world | **Unchanged** |

**Where the three architectures actually differ in cost:** v5 multiplies its Haiku spend by
the number of chunks (~2,500–7,500 per day at typical volumes). Single-Vector Plus
multiplies by the number of documents (~500 per day). The compute and storage scale
similarly. The ~$40/mo gap between the two unbuilt options is essentially this per-chunk
vs. per-doc Haiku-call asymmetry.

Macro-specific note: macro is English-dominant, so the unified-space win is less about
"now Portuguese works" (that's the M3xBr story) and more about *the door to Portuguese
content opening by default* — a macro query can now reach a relevant PT passage it
previously couldn't see. Only BM25 stays language-split (en_stem for macro).

---

## 3 · Retrieval flow: today vs. proposed

**Today**
```
query → embed (Voyage) → vector search in unified_feed
      → metadata prefilter (domain = '…' AND has_vector = 1.0)
      → top_k → synthesis
```
One pass. No rewrite, no keyword leg, no rerank, no parent expansion. A query whose words
live in a document's *title* but not its *body* tends to miss.

**Proposed Contextual v5**
```
query → ① rewrite to 2–4 sub-queries (Haiku)
      → ② semantic cache check
      → ③ PARALLEL per sub-query:
            ③a BM25 (en_stem + pt_stem) over caption+text
            ③b vector search × 3 tables (text / pdf-image / audio)
            ③c entity-KG 2-hop traversal
      → ④ RRF fusion (k=60) → top 150
      → ⑤ Cohere Rerank 3.5 → top 40
      → ⑥ editorial multipliers + entity floor → top 30
      → child→parent expansion (PDFs return parent; podcast ±1 window)
      → synthesis (Sonnet)
```
Five extra stages, each one a separate accuracy lever. The captions make stages ③a and ③b
both sharper at once.

**Proposed Single-Vector Plus**
```
query → (single pass — no per-query rewrite needed by default)
      → PARALLEL:
            ① dense vector search on doc vectors        → top 100
            ② BM25 on body text                         → top 100
            ③ BM25 on tags+entities field (boost 1.5–2×)→ top 100
      → ④ RRF fusion (k=60) → top 150
      → ⑤ Cohere Rerank 3.5 → top 40
      → ⑥ editorial multipliers (source × recency × entity floor) → top 30
      → (parent expansion only for late-chunk and page-image winners)
      → synthesis (Sonnet)
```
Four stages instead of v5's six. No per-query rewrite, no semantic cache (defer to Phase 2),
no entity-KG traversal, no parent expansion on the bulk of the corpus. The retrieval shape
is simpler because the per-doc richness (prefix + tags) shifted *from query time to ingest
time*: more work done once, on ingestion, in exchange for less work done per query.

---

## 4 · What actually drives the gain (ranked by leverage)

**Contextual v5 leverage ranking**

1. **Contextual captions** — highest ROI in the design, applied universally. Moves each
   chunk's vector into a findable neighborhood and feeds the keyword index too. ~$75/mo.
2. **Hierarchical parent-child** — kills the "one averaged point" problem for long docs while
   still handing full context to the answer model. Search small, answer big.
3. **Reranking** — re-reads the top 150 carefully; the step that compounds 49% → 67%.
4. **Multimodal layers** — PDF page-images catch charts/tables text extraction destroys;
   podcast audio fixes the per-episode black hole. High value, narrow source set.
5. **Cohere v4 embedder swap** — enables all of the above on Bedrock + unified multilingual
   space. Necessary, but on its own (Phase 1, no chunking change) the smallest lift.

**Single-Vector Plus leverage ranking**

1. **Hybrid retrieval (BM25 + dense + RRF)** — single biggest single-vector booster.
   Recovers ~50–55% of the chunking-related lift on its own across news/finance corpora.
   Effectively free.
2. **Cross-encoder reranker** (Cohere Rerank 3.5 or Voyage rerank-2.5) — the second biggest
   lever. Stacks with hybrid; together they capture most of what chunking would recover.
   ~$9/mo.
3. **Haiku-extracted tags + entities as separate BM25 leg** — the proper-noun bridge.
   Provides a high-precision lexical-match channel that fixes the "query says 'Brazil rates'
   but document says 'the central bank'" miss. ~$15/mo (folded into the same Haiku call as
   doc-level prefix).
4. **Doc-level Haiku contextual prefix** — pulls the centroid toward named topics rather
   than averaging noise. Same mechanism as v5's per-chunk prefix, applied at doc granularity
   (~20× cheaper). ~$15/mo (folded with tag extraction).
5. **Source-class / recency / entity-floor multipliers at rerank** — the only thing that
   gets the right doc to rank 1 for attribution-shaped queries. Existing M3xA scoring v3 is
   already this. ~$0.
6. **Page-image vectors for PDFs (ColPali opt-in)** — fixes chart/table queries that text
   extraction destroys. Narrow but high-value. ~$3/mo.
7. **Late chunking for docs > 4K tokens (opt-in)** — fixes long-doc needle queries without
   imposing universal chunking. Gated on whether the chosen embedder exposes token-level
   outputs. ~$3/mo if available.

---

## 5 · The honest caveat

- **Nothing on the right is built.** The comparison is baseline-vs-target. Treat every v5
  number as a hypothesis.
- **Phase 3 (captions) can't be judged without Phase 0.** The 200-query golden set is itself
  still a checklist. Turn captions on without it and you've spent ~$75/mo you can't defend.
- **The gain is concentrated.** Don't expect a uniform lift — tweets barely move, bank PDFs
  and podcasts move enormously. A blended "average improvement" number will mislead; score
  per source class.
- **Bigger chunks are not the efficiency win.** ANN search scales ~logarithmically, so halving
  vector count buys almost nothing on speed while blurring every vector. Efficiency in RAG =
  *right passage in the top few*, which small captioned children deliver, not fewer/larger ones.

---

## 6 · How to make this comparison real

The fastest path from this doc to evidence is a single 200-query golden set scored against
all three (or four) candidate retrieval configurations on the same corpus snapshot. The full
spec is in [single_vector_plus/phase0_eval.md](single_vector_plus/phase0_eval.md); the
short version:

1. Build the **200-query golden set** (~30% PT, ~10% cross-lingual, PDF-chart and
   podcast-timestamp subsets, drawn from the actual production query log). ~15 hours of
   focused labeling, one week of calendar time.
2. Score the **current Voyage stack** on it → relevance@10, MRR, attribution accuracy per
   query class. This is the baseline and fills the left column with real numbers.
3. Implement **Single-Vector Plus** end-to-end (~3–4 days). Shadow-eval against the golden
   set → honest delta vs baseline.
4. Implement **Contextual v5** end-to-end (~5–7 days). Shadow-eval against the same golden
   set → honest delta vs baseline.
5. **(Gated on embedder support)** implement **late chunking** as a fourth configuration to
   test as a middle path.
6. Compare the decision matrix (per query class, per source class). The architecture that
   wins the dominant query classes at acceptable cost wins overall.

Only after step 2 does this whole comparison stop being "reality vs. intent" and become
"before vs. after." Only after steps 3–4 are both run does the architecture decision
become a measurement instead of a debate.

---

## 7 · Further reading

For a deeper, document- and query-specific study of the single-vector vs. multi-chunk
question — including a direct answer to "do my PDFs actually need to be chunked?", a per-
document-class matrix, an honest pros/cons table, and where the 2026 literature pushes back
on the universal-chunking instinct — see **[chunking_study.md](chunking_study.md)**.

That companion file goes beyond the at-a-glance comparison above:

- Separates the three independent levers usually conflated as "chunking strategy"
  (granularity, contextualization, modality).
- Maps each query type in the system's actual traffic to the retrieval shape that wins for it.
- Argues the triple-layer PDF pattern is correct, while the universal Haiku contextual prefix
  is over-applied relative to where the evidence concentrates.
- Recommends a concrete two-tier index: single-vector default, multi-chunk selectively on
  PDFs / podcasts / long-form > 3K tokens, with late chunking as a cheaper substitute for
  hierarchical + LLM prefix on the middle of the doc-length distribution.

A third companion, **[single_vector_plus/](single_vector_plus/)**, proposes the architecture
that crystallized after the deeper evaluation: keep single-vector-per-document as the
default and compensate for its known weaknesses (hybrid BM25+dense, strong reranker,
Haiku-extracted tags+entities, doc-level contextual prefix, source-class multipliers), with
chunking reserved as a narrow opt-in for the two source classes where it's genuinely
mandatory (long-form > 4K tokens via late chunking; PDFs via page-image embeddings). It is
the deliberate alternative to the v5 spec's universal chunking, motivated by (a) the
observation that rule-based chunking introduces noise on documents with inconsistent
structure (most ingested news/research), and (b) the 2025–2026 evidence that hybrid +
strong reranker recovers most of the lift attributed to chunking on attribution- and
synthesis-heavy query mixes. The directory contains: a [README](single_vector_plus/README.md)
with motivation and architecture, an [implementation guide](single_vector_plus/implementation.md)
with schema and prompts and code, an [evidence review](single_vector_plus/evidence.md) of
June 2026 sources per compensation, and a [Phase 0 eval spec](single_vector_plus/phase0_eval.md)
for the golden-set comparison that would settle the question with measurement.

**New to RAG?** Start with the beginner walkthrough in
**[for_beginners/hierarchical_and_contextual.md](for_beginners/hierarchical_and_contextual.md)**
— it explains tokens, embeddings, the per-source-class matrix, hierarchical chunking
(parent 2,000 / child 400–512), Anthropic's contextual-retrieval prefix, and how the three
ingestion-time pieces stack together. The other companions assume this background.

A second companion, **[haiku_cleaning.md](haiku_cleaning.md)**, documents a *pre-embedding*
pattern already in production in the live system: one small-LLM (Claude Haiku) call per
ingested document at ingest time that strips source-specific boilerplate and standardizes
formatting before any embedding or chunking happens. It is independent of the chunking
decision, complementary to per-chunk contextual prefixing, and roughly 20× cheaper because
it operates per-document rather than per-chunk. Includes the actual prompts (preserve-all
and extractive variants, EN and PT), the code skeleton, the source-class routing table, and
the observed failure modes.
