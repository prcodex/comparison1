# Evidence Review — June 2026

> Per-compensation literature review for the architecture described in
> [README.md](README.md). For each of the five core compensations and the two opt-ins, this
> file documents: the mechanism, the evidence quality and citations, the magnitude of lift
> over plain single-vector dense retrieval, and the honest gaps in the evidence.
>
> **Honesty flag.** Where a specific number or paper is recalled from memory rather than
> live-fetched from arXiv/MTEB at write time, this is marked **[recall]**. Where the
> magnitude is approximate, this is marked **[approximate]**. Where only the direction is
> well-supported (not the specific magnitude), this is marked **[direction only]**.

---

## Verdict

The 2025–2026 evidence supports single-vector-per-document plus a well-tuned compensation
stack as a **viable, sometimes superior** alternative to naive chunking when documents have
unstable structure, when attribution / source-class matters, and when documents stay roughly
within the embedder's effective context (≤8–16K tokens for current top encoders). The single
biggest recoverable losses come from **hybrid retrieval + a strong cross-encoder reranker
over a wide candidate pool** — these together typically reclaim the majority of the gap to
chunk-based pipelines in attribution-heavy domains. **Doc-level contextual prefixes and
Haiku-extracted tag fields** are cheaper add-ons that help specifically with the proper-noun
blurring and centroid problems. Where single-vector still loses badly is **needle-in-
haystack span retrieval inside long documents** (NoLiMa-style) and **multi-hop questions
that require evidence from disjoint passages of one long doc** — the two opt-ins in §6 and
§7 address these.

---

## 1 · Compensation: Hybrid retrieval (BM25 + dense + RRF)

### Mechanism

Run BM25 (lexical, exact-match-driven) and dense vector retrieval (semantic, embedding-
similarity-driven) as separate legs against the same corpus. Combine the two ranked lists
with Reciprocal Rank Fusion. BM25 catches the proper nouns, tickers, jargon, and rare-token
matches that the averaged dense vector blurs. Dense catches the semantic-similarity matches
where the query and document use different words for the same concept.

### Evidence — strong

- **BEIR benchmark** (Thakur et al., 2021, arXiv:2104.08663) established that BM25 remains
  competitive with dense retrievers on out-of-domain corpora, particularly in finance, legal,
  and biomedical domains. The 2024 BEIR updates kept reinforcing this. **[recall]**
- **Anthropic's "Contextual Retrieval"** engineering post (Sept 2024) reported a ~49%
  reduction in retrieval failures going from dense-only to dense+BM25, and ~67% combined
  with their contextual prefix trick. The headline numbers are from a single internal eval
  but the *direction* has been reproduced widely. **[recall, single source]**
- Multiple production reports through 2024–2025 (Vespa, Qdrant, LlamaIndex, Weaviate blog
  posts) consistently report **+15 to +30% nDCG@10** lift from hybrid over dense-only on
  news/finance corpora. **[approximate]**

### Magnitude vs single-vector dense alone

**+15 to +30% nDCG@10** on news/finance corpora; the lift is bigger on entity-rich corpora
than on Wikipedia-style benchmarks. **[approximate]**

### Cost

Effectively free. BM25 indexes are CPU-bound, ~100–500ms to build per 100K docs, ~1–5ms to
query. LanceDB / Tantivy / Vespa / Elasticsearch all ship with production-ready BM25.

### RRF variants worth knowing (2024–2026)

- **Weighted RRF** with learned per-source weights (your existing editorial weights apply).
- **RRF with score normalization** (z-score or min-max before fusion) — outperforms vanilla
  RRF when the two lists have very different score distributions. **[direction only;
  multiple Vespa/Qdrant blog posts in 2025]**
- **Distribution-based score fusion (DBSF)** — Weaviate documented this in 2024; modest lift
  over RRF in some cases. **[direction only]**

### What hybrid does NOT fix

It doesn't fix the "blurry centroid" problem for multi-topic documents (a multi-topic
document is still represented by one vector). It surfaces the right document more often,
but the document is still the wrong granularity if the user needs a specific span.

### URL

https://arxiv.org/abs/2104.08663 (BEIR) ·
https://www.anthropic.com/news/contextual-retrieval (Anthropic Contextual Retrieval)

---

## 2 · Compensation: Strong cross-encoder reranker over top-100/200

### Mechanism

After the hybrid retrieval surfaces 100–200 candidates, a separate cross-encoder model
re-reads each *(query, document)* pair and scores them directly. Cross-encoders are slow
per-pair (you can't pre-compute them like embeddings) but consistently more accurate than
bi-encoder cosine similarity. Used as a *re-ranking* step over a small candidate pool,
they're the highest-precision retrieval signal generally available.

### Production-grade options as of June 2026

- **Cohere Rerank 3.5** (Dec 2024) — finance-tuned, multilingual, 4K context per pair,
  +23.4% NDCG@10 vs hybrid search reported on internal finance benchmark. **[recall]**
- **Voyage rerank-2.5** (mid-2025) — instruction-following (you can pass per-query rerank
  instructions), 32K context per pair, +7.94% NDCG@10 over Cohere Rerank 3.5 in Voyage's
  benchmarks. **[recall]**
- **BGE-reranker-v2-m3** (open weights) — best-in-class open option, ~50ms per top-150
  batch on a single GPU.
- **Mixedbread mxbai-rerank-v2** — entered leaderboard in 2025, competitive with the hosted
  options.

### Evidence — strong

Cross-encoders have dominated MTEB reranking and BEIR reranking subsets continuously.
Across published evals (Cohere's own, Voyage's, plus independent reproductions on
FinanceBench and the LegalBench retrieval subset), **reranking a top-100 dense pool down to
top-10 yields +8 to +20 nDCG@10 points over the dense ranking alone**. **[approximate;
varies by domain]**

### Specifically for single-vector retrieval

The reranker is doing *more* work than usual because the initial vector match is coarser.
Empirically (per 2025 production blog posts from Vespa, Qdrant, and LlamaIndex), **the
reranker recovers most of the per-chunk granularity lost at retrieval** — it can read the
full doc (or a windowed slice) and score query–doc relevance directly, which is exactly
the operation chunk-level dense retrieval was approximating.

### Cost

~$1–2 per 1K reranks for hosted (Cohere/Voyage), near-zero for self-hosted BGE on a T4/A10
GPU. At top-100 candidates × 100 queries/day, hosted reranking costs ~$9/month at production
volumes — trivial.

### Strong interaction with hybrid

Hybrid + rerank is *the* canonical 2025 stack. The rerank step also cleans up the score-
distribution mismatch between BM25 and dense — you almost stop caring about RRF tuning
once a good reranker sits behind it.

### URL

https://cohere.com/blog/rerank-3pt5 · https://blog.voyageai.com/voyage-rerank-2.5

---

## 3 · Compensation: Haiku-extracted tags / entities as separate BM25 leg

### Mechanism

At ingest time, one Haiku call extracts a structured JSON of `entities`, `topics`,
`themes`, `geographies`, and `data_points` per document. These get stored in dedicated
columns and indexed by a separate BM25 index (with a field-level boost). At query time,
this becomes a third retrieval leg alongside dense and body-BM25.

The mechanism is essentially **automated metadata augmentation**: a query for "Brazil
rates" runs against (a) the doc-vector, (b) the body text via BM25, and (c) the tags field
via BM25. The tags field contains `brazil_rates` and `Banco Central do Brasil` regardless
of whether the document body says "Brazil" explicitly — it might say "the central bank,"
and the tags bridge the vocabulary gap.

### Evidence — medium

Weaker formal evidence than (1) and (2). I'm not aware of a clean academic eval titled
"tag-augmented single-vector retrieval vs chunked retrieval" in 2025–2026. What exists:

- **Anthropic's Contextual Retrieval** (Sept 2024) is the closest formal data point —
  adding ~50–100 tokens of generated context per *chunk* improved retrieval by ~35% (dense
  alone) / ~49% (combined hybrid). Applied at *doc* level via a tags+entities field, the
  direction should hold; the magnitude is unclear. **[extrapolation]**
- **Microsoft GraphRAG** (2024–2025) papers show entity-graph augmentation helps multi-hop
  questions specifically; less so for simple retrieval. **[recall]**
- Production posts from LlamaIndex, Pinecone, and Qdrant in 2025 consistently report
  **+5 to +15% recall@10** from metadata field expansion when the corpus has many rare
  entities. **[direction strong, magnitude approximate]**

### Three distinct uses

- **(i) Metadata filters** — high-ROI when queries are naturally filterable (`only sell-side,
  last 7 days, Brazil-tagged`). Independent of the embedding strategy.
- **(ii) BM25 field expansion** — the most underrated trick. Indexing Haiku-extracted tags
  into a separate BM25 field with a 1.5–2× field boost recovers exactly the "central bank
  → Brazil rates" kind of bridge that the blurry-centroid problem creates. Direction
  strongly positive; magnitude depends on tag-extractor quality.
- **(iii) Entity boosting at rerank** — see compensation (5).

### Interaction with the reranker

Tags overlap significantly with what a strong reranker would surface on its own. If you
have Voyage rerank-2.5 or Cohere Rerank 3.5 in the loop, the *marginal* lift from tags is
smaller (perhaps +3–5% recall@10) — but tags still help at the *retrieval* stage *before*
the reranker sees the candidates. The reranker can only rerank what hybrid surfaces; tags
make sure the right candidates get into the pool in the first place.

### Cost

~$0.001/doc with Haiku 4.5 when folded into the same call as the doc-level prefix
(compensation 4). ~$15/month at production volumes. Trivial.

### URL

https://www.anthropic.com/news/contextual-retrieval (the per-chunk pattern this extrapolates
from)

---

## 4 · Compensation: Doc-level Haiku contextual prefix

### Mechanism

A 100–200 token prose paragraph, generated by Haiku at ingest, that describes what the
document is and what it contains (using the entities and topics from compensation 3 in
natural sentences). The prefix is prepended to the document body before computing the
single doc-vector. The combined `(prefix + body)` gets embedded.

This is **the per-document analogue of Anthropic's per-chunk contextual retrieval pattern**.
Same mechanism (LLM-generated context injection before embedding), different granularity
(once per document instead of once per chunk).

### Evidence — weak

I do not know of a published benchmark that isolates *doc-level* prefix from *chunk-level*
prefix in 2025–2026. The extrapolation rests on:

- The mechanism Anthropic identified — generated context makes the embedding "self-
  describing" so the embedder doesn't have to guess what the surrounding context is —
  applies at *any* granularity.
- For single-vector-per-doc, doc-level prefix should help most with the blurry-centroid
  problem: a 150-token "this is a Bank N note covering Brazil fiscal, Selic path, and
  energy-sector dividend policy" preamble pulls the centroid toward those named topics
  rather than averaging the noise.
- It will help much less with the long-doc / needle-style problems addressed by the late-
  chunking opt-in.

### Estimated lift

**+10 to +20% recall@10 over plain single-vector** **[approximate, direction only]**, with
most of the lift concentrated on multi-topic documents and almost none on single-topic
documents. Worth measuring on the Phase 0 golden set — this is one of the experiments most
worth running and least covered in the public literature.

### Cost

~$0.001–0.005/doc with Haiku 4.5 when folded into the same call as the tag extraction
(compensation 3). Negligible.

### Key implementation detail

Include the tags from compensation 3 *inside* the prose prefix. This collapses (3) and (4)
into a single Haiku call per document with two outputs: structured JSON for indexing, and
prose prefix for embedding. The prose prefix uses the entities and topic slugs from the
JSON in natural sentences, ensuring the embedding picks them up at high weight.

### URL

https://www.anthropic.com/news/contextual-retrieval (the per-chunk pattern)

---

## 5 · Compensation: Source-class / entity / recency multipliers at rerank

### Mechanism

After the cross-encoder reranker produces top-40 candidates, apply multiplicative
adjustments to the rerank score:

- **Source-class multiplier** — sell-side research > op-ed > tweet for "what does X say"
  queries; news wire > op-ed for breaking-news queries; etc.
- **Recency decay** — `exp(-Δt / τ)` with `τ` between 3–14 days for news.
- **Entity floor** — if the query mentions an entity (e.g., "Bank N"), guarantee at least N
  documents from that entity in the top-30 regardless of pure rerank score.

### Evidence — weak in academic benchmarks, strong in production reports

Academic IR has been slow to evaluate metadata-augmented reranking rigorously because it's
domain-specific. **RAGBench** (Friel et al., 2024, arXiv:2407.11005) and **CRAG**
(Comprehensive RAG Benchmark, Meta, 2024) provide some scaffolding but not a clean ablation
of editorial weights. **[recall]**

The mechanism is uncontroversial and the M3xA domain (attribution-heavy macro/
geopolitics) is exactly where it pays off most. Empirically what teams report:

- Source-class multipliers at rerank time (not at retrieval) consistently improve human-
  rated answer quality, even when nDCG against a content-only ground truth is flat or
  slightly worse.
- For "what does X think about Y" queries, source weighting is the *only* thing that gets
  the right doc to rank 1 — semantic similarity to wire copy will often beat semantic
  similarity to the author's actual piece.
- Recency decay as a multiplicative term is well-established; `τ` between 3–14 days is
  typical for news.

### Caveat

This is post-hoc score manipulation, not a retrieval algorithm. Tune `τ` and source weights
against human-labeled relevance judgments (the Phase 0 golden set) or you'll overfit to
priors. The existing scoring v3 with Haiku-classified `macro_pct` / `geo_pct` /
`source_importance` is essentially this, done well, and should carry over.

### Cost

Effectively zero. Multipliers are a 5-line scoring pass in pandas.

### URL

https://arxiv.org/abs/2407.11005 (RAGBench)

---

## 6 · Opt-in: Late chunking (Jina, 2024) for documents > 4K tokens

### Mechanism

Encode the whole document once with a long-context bi-encoder that exposes token-level
outputs. Then mean-pool token embeddings over fixed-size windows (e.g., 600-token windows
with 500-token stride). Each window's vector carries cross-window context for free because
every token saw the whole document in attention.

### Evidence — medium-strong

- **Günther et al., "Late Chunking: Contextual Chunk Embeddings Using Long-Context Embedding
  Models"** (arXiv:2409.04701, Sept 2024; updated July 2025) — the original paper. Reported
  **+5 to +10% nDCG@10** on BEIR subsets vs naive chunking. **[recall]**
- Independently reproduced by LlamaIndex (late 2024) and Weaviate (Q1 2025). Direction
  robust; magnitude varies by corpus.

### Practical answers to common questions

- **Does it require smart chunk boundaries?** No — that's the point. Late chunking is more
  tolerant of naive fixed-window splits than traditional chunking is, because each token's
  contextual embedding already carries doc-level info before pooling. You can use 512- or
  600-token windows with no overlap and still get decent results. **This is the property
  that makes it fit the "rule-based chunking introduces noise" concern that motivated
  single-vector in the first place.**
- **Does it work past the embedder's context limit?** Only up to the embedder's max
  sequence length (8K for jina-embeddings-v3, 32K for the long-context variants of BGE-M3
  and similar). Past that, it degrades like any other transformer attention mechanism.
- **What tooling supports it?** Jina v3 API supports it natively. For self-hosted, you need
  an embedder that exposes token-level embeddings (jina-v3, nomic-embed-text-v1.5, BGE-M3
  in multi-vector mode). **Voyage-3-large and Cohere v4 currently expose only pooled
  outputs via their hosted APIs** — if you stay on those, late chunking requires either
  self-hosting a compatible model or skipping this opt-in entirely. This is a real
  architectural decision; see [phase0_eval.md](phase0_eval.md).

### Where it fits in the architecture

Opt-in by document length (>4K tokens), not by source class. ~10–20% of the corpus enters
this path. The doc-level vector from compensation 4 still gets stored — late-chunk vectors
*supplement* it, they don't replace it.

### Honest gap

I do not know of a 2025–2026 paper that directly compares **late chunking vs single-vector
+ doc-level contextual prefix** on a finance/news corpus. That's the most directly relevant
experiment for the decision and the one most worth running on the Phase 0 golden set.

### URL

https://arxiv.org/abs/2409.04701

---

## 7 · Opt-in: ColPali / page-image vectors for PDFs

### Mechanism

Render each PDF page as an image (~150 DPI, ≤2MM pixels). Embed each image via a
multimodal embedder (Cohere v4 image mode, or ColPali itself if self-hosting). Store the
page-image vectors keyed back to the parent `doc_id`.

### Evidence — strong (for visual document retrieval specifically)

- **Faysse et al., "ColPali: Efficient Document Retrieval with Vision Language Models"**
  (arXiv:2407.01449, ECIR 2025) — the seminal paper. Established the page-image embedding
  pattern. On the ViDoRe benchmark, ColPali and ColPali-style approaches beat OCR + text
  pipelines by **15–25 percentage points nDCG@10** on chart-and-table-heavy documents.
  **[recall]**
- Cohere Embed v4 (April 2025 release) implements the same pattern natively via image
  mode, in the same vector space as its text embeddings — enabling cross-modal retrieval.

### Where it fits

Opt-in by source class (PDFs only), not by length. Adds a fourth retrieval leg (or extends
Leg A) at query time. Hits return their parent `doc_id` for assembly.

### Cost

~$3/month at production PDF rates. The line item is dominated by image embedding cost,
which is small at ~50 PDFs/week × ~25 pages = ~5,000 page embeds/month.

### What this catches that text-only single-vector cannot

- Equity / yield / rate curves embedded as images
- Scatter plots
- Heatmaps
- Formatted tables with multi-row headers that OCR scrambles
- Hand-drawn analyst sketches and annotated charts

These are the queries that single-vector retrieval on text-only PDF extracts will
*systematically* miss. ColPali is the only known approach that handles them at production
quality.

### URL

https://arxiv.org/abs/2407.01449

---

## 8 · What this architecture does NOT include (and why)

Three patterns that come up in RAG discussions but are *not* part of this design, with the
reasoning:

### 8.1 Multi-vector-per-doc ("doc + 2–3 topic summary vectors")

Considered as a middle path. Direction: should help, magnitude unclear, no published
benchmark. The key risk is that topic-summary vectors drift from doc content in ways that
introduce false positives — you retrieve based on the summary, then the answer's not
actually in the document.

**Why excluded:** the late-chunking opt-in (compensation 6) is a stronger, better-supported
alternative for the same goal (multiple search angles per document). If late chunking turns
out to be unavailable (e.g., embedder doesn't expose token-level outputs), revisit this as
a fallback.

### 8.2 HyDE (Hypothetical Document Embeddings)

Generates a hypothetical answer document at *query* time and embeds *that* for retrieval.
Useful when the query is very short and the corpus uses very different vocabulary. Not
addressing the specific problems this architecture targets.

**Why excluded:** different problem (query-side augmentation, not doc-side). Worth
revisiting if attribution queries with short bot-style inputs become a measurable failure
mode.

### 8.3 RAPTOR-style summary trees

Recursive abstractive summarization tree built over the document collection. Strong on
multi-hop and aggregation queries.

**Why excluded:** ages badly with corpus churn — every leaf change invalidates summary
nodes, and the M3xA corpus is daily-updating. The doc-level contextual prefix
(compensation 4) captures most of the per-document benefit without the rebuild cost.

---

## 9 · The honest gaps in the evidence

Called out explicitly:

- **No published benchmark isolates doc-level contextual prefix from chunk-level
  contextual prefix.** The directional claim (it works similarly at any granularity) is
  reasonable but unproven. The Phase 0 golden set should measure this directly.
- **No published benchmark directly compares this stack against full hierarchical
  chunking on a finance/news corpus.** Most existing benchmarks are either pure dense
  vs pure BM25 vs pure chunking comparisons; the "single-vector + everything around it"
  configuration is a production pattern not a benchmarked configuration.
- **Tag/entity field BM25 evidence is mostly engineering reports, not formal evaluations.**
  Direction is strong; magnitude varies wildly with tag-extractor quality and controlled-
  vocabulary discipline.
- **The 1.5–2× field boost on the tags BM25 leg is a hand-tuned range, not an empirically
  derived constant.** Tune on the golden set.

These gaps are the strongest argument for treating Phase 0 as a *gate*, not a formality —
without measurement, every claim in this document is a hypothesis.

---

## 10 · References (consolidated)

**Strongest evidence — well-established techniques**

- Thakur et al., *BEIR: A Heterogeneous Benchmark for Zero-shot Evaluation of Information
  Retrieval Models*, arXiv:2104.08663 (2021) — https://arxiv.org/abs/2104.08663
- Faysse et al., *ColPali: Efficient Document Retrieval with Vision Language Models*,
  arXiv:2407.01449 (ECIR 2025) — https://arxiv.org/abs/2407.01449
- Günther et al., *Late Chunking: Contextual Chunk Embeddings Using Long-Context Embedding
  Models*, arXiv:2409.04701 (2024, updated 2025) — https://arxiv.org/abs/2409.04701

**Medium evidence — direction strong, magnitude reproduced**

- Anthropic, *Contextual Retrieval*, blog post (Sept 2024) —
  https://www.anthropic.com/news/contextual-retrieval
- Friel et al., *RAGBench: Explainable Benchmark for Retrieval-Augmented Generation
  Systems*, arXiv:2407.11005 (2024) — https://arxiv.org/abs/2407.11005

**Production references — biased but informative**

- Cohere Rerank 3.5 release (Dec 2024) — https://cohere.com/blog/rerank-3pt5
- Voyage Rerank 2.5 release (mid-2025) — https://blog.voyageai.com/voyage-rerank-2.5
- Cohere Embed v4 release (April 2025) — multimodal, multilingual, finance-tuned
- Various 2025 production blog posts from Vespa, Qdrant, LlamaIndex, Weaviate on hybrid +
  rerank stacks
