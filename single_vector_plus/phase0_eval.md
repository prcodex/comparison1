# Phase 0 Evaluation — Settling the Architecture Question with Measurement

> Without measurement, every claim in [README.md](README.md), [implementation.md](implementation.md),
> and [evidence.md](evidence.md) is a hypothesis. This file specifies the golden-set
> evaluation that would convert those hypotheses into measured results — directly comparing
> single-vector + compensations against full hierarchical chunking and against late
> chunking, on the actual M3xA corpus and query distribution.
>
> **Why Phase 0 is a gate, not a formality.** Three of the highest-leverage claims in this
> directory have no published benchmark behind them: (1) doc-level contextual prefix vs
> chunk-level prefix, (2) tag-augmented single-vector vs chunked retrieval on news/finance
> corpora, (3) single-vector + everything-around-it vs full hierarchical chunking. The
> golden set is the only way to know which architecture is right *for this corpus and these
> queries*.

---

## 1 · What this eval produces

A spreadsheet with one row per (query, retrieval-configuration) pair and these columns:

| Column | What it measures |
|---|---|
| `query_id` | Stable ID for the query |
| `query_text` | The user's question |
| `query_class` | Attribution / recency / synthesis / fact-lookup / multi-hop / chart-or-table |
| `source_class_target` | The doc class the right answer should come from |
| `gold_doc_ids` | Set of document IDs that contain the right answer (human-labeled) |
| `retrieval_config` | One of: `single_vector_baseline`, `svp_v1` (this architecture), `hierarchical_v5`, `late_chunking_v1` |
| `retrieved_top_30_ids` | What the configuration actually surfaced |
| `relevance_at_10` | Was a gold doc in the top 10? |
| `relevance_at_30` | Was a gold doc in the top 30? |
| `mrr` | Mean reciprocal rank of the first gold doc |
| `attribution_accuracy` | For attribution queries: did the top result come from the named entity? |
| `latency_ms` | Wall-clock retrieval latency |
| `cost_usd` | Per-query cost of this configuration |

Three numbers per (config, query_class) cell:

- Mean **relevance@10**
- Mean **MRR**
- Mean **attribution accuracy** (where applicable)

Plus per-cell **cost** and **latency**. The decision matrix then writes itself.

---

## 2 · The 200-query golden set — composition

The set is the most labor-intensive part of Phase 0 and the single highest-value
engineering artifact this evaluation produces. It should reflect the *actual* query
distribution, not a synthetic balanced set.

### 2.1 Sampled from `telegram_conversations.db`

Pull 1,000 recent real user queries, stratified by:

- Bot (macro vs brazil vs mxai)
- Time of day (morning brief vs midday vs evening)
- User (the operator's queries vs guest-user queries)

Filter to 200 by:

- Coverage of all 6 `query_class` values
- Coverage of all 8 `source_class_target` values (tweet, wire, longform, email, PDF,
  podcast, geo, calendar)
- Avoidance of duplicate semantic queries (different phrasings of "what's the latest on
  Iran" all map to one entry, with the others dropped)
- Inclusion of the failure modes documented in `mind_lessons.md`:
  - #21 (Tony P / sell-side attribution)
  - #22 (high-volume wire under-retrieval)
  - #44 (institution-named-in-podcast undetected)
  - #15 (scope keyword leak)

### 2.2 Target distribution

| Query class | Target count | Why this weight |
|---|---|---|
| Named-source attribution | 50 | Dominant production query type ("what does Bank N say about X") |
| Time-windowed multi-source synthesis | 50 | Dominant production query type ("last 24h on Iran") |
| Thematic synthesis | 30 | Common premade-prompt usage ("top 10 macro themes") |
| Fact / number lookup | 30 | Includes chart/table queries (Layer C ColPali targets) |
| Multi-hop synthesis | 20 | Tests where chunking specifically wins |
| Day-over-day comparison | 10 | Niche but well-defined ("what's new vs yesterday") |
| Cross-lingual (EN query → PT corpus or vice versa) | 10 | Validates Cohere v4 unified-space claim |

### 2.3 Language distribution

- ~70% English
- ~25% Portuguese (Brazil-domain queries)
- ~5% mixed / cross-lingual

### 2.4 Source-class breakdown for the gold-doc target

| Source class target | Target count | Notes |
|---|---|---|
| Tweet | 20 | Wire/headline-style queries; chunking irrelevant |
| Wire article | 30 | Short-doc queries; chunking irrelevant |
| Email research | 30 | Borderline length; chunking may or may not pay |
| Long-form journalism | 25 | Where late chunking should pay |
| Sell-side PDF | 35 | Includes chart/table subset (ColPali target) |
| Podcast | 25 | Includes named-guest subset (chunking target) |
| Geo OSINT | 15 | Structured, generally short |
| Cross-source synthesis | 20 | Multiple gold docs per query |

### 2.5 Human labeling — who and how

Pedro and ~1 collaborator with domain knowledge. For each of 200 queries:

1. Run the query through the *current* production system + pull the top 50 retrieved.
2. Label each of the 50 as: gold (would answer the query), supporting (related), or
   irrelevant.
3. Manually search the corpus for any gold docs not in the top 50 and add them.

This produces, per query, a `gold_doc_ids` set of typically 1–5 documents. Time budget:
~3–5 minutes per query × 200 = ~10–17 hours of focused labeling. Spread over a week.

---

## 3 · The four retrieval configurations to compare

All four configurations are evaluated against the *same* 200-query golden set with the
*same* corpus snapshot. The only thing that varies is the retrieval pipeline.

### 3.1 Config A — `single_vector_baseline` (current production)

The current M3xA retrieval as it runs today:

- 1 vector per doc (Voyage-3-large 2048d, no prefix)
- Vector search + metadata prefilter only (no BM25 leg, no rerank)
- Editorial multipliers applied to raw cosine similarity
- Top 30 returned

This is the baseline every other configuration measures against.

### 3.2 Config B — `svp_v1` (this architecture)

The architecture described in [implementation.md](implementation.md):

- 1 vector per doc, computed on `(prefix + body)` via Voyage-3-large
- Haiku-extracted tags + entities stored as separate field
- 3-leg retrieval: dense + BM25(body) + BM25(tags) → RRF → top 150
- Cohere Rerank 3.5 → top 40
- Editorial multipliers → top 30

### 3.3 Config C — `hierarchical_v5` (the v5 spec)

The full hierarchical chunking architecture from the proposed v5 design:

- Hierarchical chunking (parent 2,000 / child 400–512) on all docs > 1500 tokens
- Per-chunk Haiku contextual prefix on every child
- Cohere v4 embeddings @ 1536d (or Voyage-3-large for fairness with B)
- Hybrid retrieval + Cohere Rerank 3.5
- Editorial multipliers → top 30 parents

### 3.4 Config D — `late_chunking_v1` (the middle path)

Late chunking (Jina-style) only:

- 1 doc-level vector via Voyage-3-large
- Additional sub-doc vectors via late chunking (600-tok windows, no smart boundaries) for
  docs > 4K tokens
- Hybrid retrieval + Cohere Rerank 3.5
- Editorial multipliers → top 30

**Note:** Config D is gated on whether the chosen embedder exposes token-level outputs.
If Voyage-3-large does not (as of writing), Config D requires either self-hosting a
compatible model (BGE-M3, jina-v3) or being dropped from the comparison.

---

## 4 · The decision matrix this produces

After running all four configs against all 200 queries, the result is a 4 × 6 matrix
(configs × query classes) of mean relevance@10:

|                          | Attribution | Recency synthesis | Thematic | Fact lookup | Multi-hop | Comparison |
|---                       |---          |---                |---       |---          |---        |---         |
| `single_vector_baseline` | (baseline)  | (baseline)        | (baseline)| (baseline) | (baseline)| (baseline) |
| `svp_v1`                 | ?           | ?                 | ?        | ?           | ?         | ?          |
| `hierarchical_v5`        | ?           | ?                 | ?        | ?           | ?         | ?          |
| `late_chunking_v1`       | ?           | ?                 | ?        | ?           | ?         | ?          |

The decision rules:

- **If `svp_v1` wins or ties on attribution + recency + thematic** (the dominant 3 classes
  = ~130 of 200 queries) **and is within 5 pp of `hierarchical_v5` on multi-hop + fact
  lookup** → ship `svp_v1`. The cost and complexity savings dominate.
- **If `hierarchical_v5` wins by > 10 pp on multi-hop + fact lookup AND `svp_v1` is no
  better than baseline on attribution** → ship `hierarchical_v5`. Single-vector is
  genuinely insufficient.
- **If `late_chunking_v1` is the best on long-doc queries (>4K tok) AND `svp_v1` wins
  short-doc queries** → ship `svp_v1` with the late-chunking opt-in (the recommended
  hybrid). Most likely outcome.
- **If results are within noise (<3 pp differences)** → ship `svp_v1` on cost grounds.
  This is the "no measurable difference" case where the cheaper architecture wins.

### 4.1 Separate sub-matrix for chart/table queries (PDFs only)

Run the 15 chart-or-table queries against `svp_v1` *with and without* the ColPali opt-in
enabled. This isolates the value of page-image embeddings on the queries they're designed
to serve.

Decision: if ColPali surfaces ≥ 30% more gold-doc-pages than text-only on this subset,
ship the ColPali opt-in. If not (e.g., text extraction is doing better than expected),
defer.

---

## 5 · Engineering effort

| Phase | Effort | Calendar time |
|---|---|---|
| **0.1** Sample 1,000 queries, filter to 200, label gold docs | ~15 hours | 1 week |
| **0.2** Implement `svp_v1` end-to-end (Haiku tag+prefix, 3-leg retrieval, reranker wired) | ~3–4 days | 1 week |
| **0.3** Implement `hierarchical_v5` end-to-end (chunker, per-chunk prefix, hybrid + rerank) | ~5–7 days | 1.5 weeks |
| **0.4** Implement `late_chunking_v1` (gated on embedder support) | ~2–4 days | 0.5–1 week |
| **0.5** Run all 4 configs against the 200-query set, score, populate the matrix | ~2 days | 0.5 week |
| **0.6** Analyze, write decision memo, share for review | ~2 days | 0.5 week |
| **Total** | ~3–4 weeks engineering | ~4–5 weeks calendar |

Roughly half the effort is in Phase 0.3 (full hierarchical-v5 implementation), which is
the most complex configuration. If Phase 0.3 risks slipping, an acceptable simplification
is to use a stock LangChain or LlamaIndex hierarchical-chunking implementation rather than
a from-scratch build — the goal is to *measure* the architecture, not optimize it.

---

## 6 · What this eval does NOT settle

Called out so the boundary of the experiment is explicit:

- **Production latency at scale.** The 200-query set runs offline; sustained query rates,
  cache behavior, and concurrency effects are out of scope. Expect a separate
  load-testing pass after the architecture decision is made.
- **Long-term cost drift.** The cost numbers in [implementation.md](implementation.md)
  assume current vendor pricing. Re-validate annually.
- **Tag-extraction quality drift.** If the controlled vocabulary in `topics_vocab.yml`
  ages badly (new sources cover new themes), the BM25-tag-leg lift will degrade. Schedule
  a quarterly vocab review.
- **Cross-corpus generalization.** Results are specific to M3xA's mix of macro + geo +
  brazil + AI content. A different corpus mix (legal contracts, biomedical papers) would
  shift the decision matrix meaningfully.
- **Subjective answer quality.** The eval measures retrieval — it does not measure
  whether the synthesizer LLM produces good answers given the retrieved context. A
  follow-up eval on answer quality (using the existing Actor 7 self-evaluator framework)
  is the natural next step after the retrieval question is settled.

---

## 7 · Why this eval is worth doing even if you've already decided

Two reasons it's worth running even if Pedro's prior is already pro-`svp_v1`:

1. **It establishes the per-source-class breakdown** that lets you tune the opt-ins. Knowing
   that `svp_v1` is at 0.78 relevance@10 on email research but only 0.61 on long-form
   journalism > 4K tokens tells you exactly where the late-chunking opt-in needs to fire.
2. **The 200-query golden set becomes a permanent regression-test corpus.** Every future
   retrieval change (new source, new prompt, new embedder) can be measured against the
   same set. Without it, every change is unmeasured and accumulating regressions go
   undetected.

The set is the artifact. The current eval is just its first use.
