# M3xA: Current Model vs. Contextual v5 — Comparison

> **Read this first.** The left column is *live and measured* — what M3xA actually runs
> today. The right column is *spec and unbuilt* — the v5 target from `chunking_vector`,
> explicitly "design spec, not yet implemented." So this is **current reality vs. designed
> intent**, not two measured systems. Every right-column number (35–67% lift, $95/mo,
> retrieval shape) is a *claim to be proven*, not an observed result. Phase 0's golden set
> is the thing that turns the right column from claims into numbers.

---

## 1 · At a glance

| Dimension | Current M3xA | Contextual v5 (proposed) |
|---|---|---|
| **Text embedder** | Voyage-3-large @ 2048d, sole embedder, vendor key | Cohere Embed v4 @ 1536d (Matryoshka), Bedrock-native |
| **Multilingual** | EN + PT handled separately | One unified EN/PT/AR space — cross-lingual by default |
| **Chunking** | ~1 vector per document for everything | Per-source-class matrix; hierarchical parent 2000 / child 400–512 |
| **Podcasts** | 1 vector per *whole episode* (a retrieval black hole) | Text chunks (~600 tok, timestamped) **+** parallel Nova audio @ 3072d |
| **Bank PDFs** | 1 vector per whole PDF | Triple-layer: doc vector + text children + page-image vector/page |
| **Contextual captions** | None | Universal Haiku prefix on every chunk, before embed *and* BM25 |
| **Keyword search (BM25)** | Planned, not built | en_stem + pt_stem Tantivy over caption + chunk text |
| **Reranking** | Not wired | Cohere Rerank 3.5, top-150 → top-30 |
| **Multimodal** | None (text extraction only) | PDF page-images (ViDoRe pattern) + podcast audio |
| **Storage** | Single `unified_feed` table | `unified_v5_text` + `unified_v5_pdf_images` + `unified_v5_audio` |
| **Retrieval** | Vector search + metadata prefilter, top-k | Rewrite → cache → parallel multi-index → RRF → rerank → parent expansion |
| **Steady-state cost** | Voyage vendor pricing | ~$95/mo (+ ~$360 one-time backfill) |
| **Expected retrieval lift** | baseline | −35% failures (caption) → −49% (+BM25) → −67% (+rerank), *claimed* |

The single conceptual shift: **today every document collapses to one averaged point;
v5 searches on small, self-describing units and returns big context to the answer model.**

---

## 2 · Per-source impact inside M3xA macro

The gain is deliberately *uneven* — it concentrates where macro is heaviest (bank research,
podcasts) and barely touches what was already small and findable (tweets).

| Macro source | Current | Contextual v5 | Size of change |
|---|---|---|---|
| Macro tweets (~130 accts, 30–80 tok) | 1 Voyage vector | 1 vector **+ caption** | Small — caption + better embedder only |
| Wire articles (400–2000 tok) | 1 vector | 1 vector if short; children if >1500 tok | Moderate |
| Long-form journalism (3K–8K) | 1 averaged vector | Hierarchical, section-aware | **Large** |
| Email research notes (1.5K–6K) | 1 vector | Section-split into 3–15 captioned chunks | **Large** |
| Bank / Drive PDFs (5K–25K) | 1 vector per PDF | **Triple-layer** doc + text + page-image | **Largest** |
| Podcast transcripts (5K–25K) | 1 vector per episode | Text chunks + parallel Nova audio | **Largest** |
| Geo OSINT structured | 1 vector / article-style | Short = 1 vec; long = article-style | Small–moderate |
| Calendar / markets / Polymarket | DB rows, time-joined | **Unchanged** — never enters the vector world | None |

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

**Proposed v5**
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

---

## 4 · What actually drives the gain (ranked by leverage)

1. **Contextual captions** — highest ROI in the whole design, applied universally. Moves each
   chunk's vector into a findable neighborhood and feeds the keyword index too. ~$75/mo.
2. **Hierarchical parent-child** — kills the "one averaged point" problem for long docs while
   still handing full context to the answer model. Search small, answer big.
3. **Reranking** — re-reads the top 150 carefully; the step that compounds 49% → 67%.
4. **Multimodal layers** — PDF page-images catch charts/tables text extraction destroys;
   podcast audio fixes the per-episode black hole. High value, narrow source set.
5. **Cohere v4 embedder swap** — enables all of the above on Bedrock + unified multilingual
   space. Necessary, but on its own (Phase 1, no chunking change) the smallest lift.

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

The fastest path from this doc to evidence:

1. Build the Phase 0 golden set (200 queries, ~30% PT, ~10% cross-lingual, PDF-chart and
   podcast-timestamp subsets).
2. Score the **current** Voyage stack on it → relevance@10, MRR, right-source-in-top-30.
   That fills the left column with real numbers.
3. Stand up Phase 1 (Cohere v4 swap, no chunking change), shadow-eval → first honest delta.
4. Layer Phases 2–4 (hierarchical → captions → rerank), re-scoring at each gate.

Only after step 2 does this table stop being "reality vs. intent" and become "before vs. after."

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
