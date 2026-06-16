# Single-Vector vs Multi-Chunk Indexing — A Document- and Query-Specific Study

> **Companion to [README.md](README.md).** The main comparison frames *current vs. proposed* at a
> high level. This file goes deeper on one question the README defers: **given the documents
> M3xA actually ingests and the queries users actually ask, when does breaking a document into
> multiple chunks make the system measurably better — and when is one vector per document a
> valid (even preferable) alternative?** Particular attention is paid to the PDF case, because
> that is the document class where the answer matters most.

---

## TL;DR — directly answering "do my PDFs need to be chunked?"

**Yes for PDFs — multi-chunk is the strongest case for chunking in the entire corpus — but the
win is conditional on the query mix, and the right answer is not "chunk vs. don't chunk" but
rather the *triple layer* (doc-vector + text-children + page-image).** A single text vector per
PDF, paired with a strong reranker and an author/source filter, is a defensible *alternative*
for systems where queries are mostly attribution-shaped ("what does Bank N publish about X").
But for any query that needs (a) a specific passage buried in a 20K-token note, (b) a number
from a table, or (c) information from a chart that text extraction destroys, single-vector PDFs
fail and multi-chunk + multimodal wins decisively.

For the rest of the corpus — tweets, wires, most email notes, op-eds under ~3K tokens — the
2026 evidence does **not** support a universal "more chunks = better" rule. With a long-context
embedder (Cohere v4, Voyage-3-large) and a strong reranker, one vector per doc is competitive
on attribution, synthesis, and recency queries — which together are the majority of M3xA
traffic. Universal chunking + per-chunk LLM-generated contextual prefixes (the most expensive
component of the proposed v5 architecture) is **over-applied** relative to where the evidence
says the lift actually concentrates.

---

## 1 · The two questions, separated

Production discussions usually collapse three independent decisions into one "chunking
strategy." Separating them makes the cost/benefit much clearer:

1. **Granularity** — should this document become 1 vector, or N vectors?
2. **Contextualization** — if N, should each chunk get an LLM-generated prefix (Anthropic
   Contextual Retrieval) before embedding?
3. **Modality** — for visual documents (PDFs with charts/tables), should pages also be
   embedded as images (ColPali / ViDoRe pattern), independently of text chunking?

These are independent levers. The proposed v5 architecture pulls all three at once on
high-value docs, which is correct *if you've established that all three pay*. The evidence
synthesized below suggests pulling them selectively rather than universally.

---

## 2 · The document mix being indexed

Drawn from the source matrix in the proposed architecture. Token counts are typical, not
strict.

| Class | Typical tokens | Volume | Why this class behaves the way it does |
|---|---|---|---|
| **Tweets** (~130 tracked accounts) | 30–80 | High (thousands/day) | Already a "chunk" — no internal structure to recover. Author IS the unit. |
| **Wire articles** (Wire 1–7) | 400–2000 | High | Inverted-pyramid structure — the lede already summarizes; chunking adds noise. |
| **Long-form journalism / op-eds** (Pub 1–8) | 3000–8000 | Medium | Narrative; anaphora across paragraphs ("the policy", "the chairman") matters. |
| **Email research notes** (sell-side / indie) | 1500–6000 | Medium | Author voice dominates; readers ask "what does X think?" as much as "what does the note say?" |
| **Bank / Drive PDFs** (Bank 1–11, A–F) | **5000–25000** | Medium (~50/week) | Long, structurally rich (sections, charts, tables). A single 25K-token note covers 4–8 distinct topics. |
| **Podcast transcripts** | **5000–25000** | Medium (~10/day) | Multi-speaker, time-structured. The named guest IS the retrieval target half the time. |
| **Geo OSINT / structured monitors** | mixed | High | Mostly short events; occasional long reports. |
| **Calendar / markets / prediction-market data** | structured | High | Not chunked — DB rows, joined at query time. |

Two characteristics matter for the chunking decision:

- **Length distribution is bimodal.** ~80% of incoming documents are under 2000 tokens (tweets,
  wires, short emails). ~20% are 5000+ tokens (PDFs, podcasts, some long-form). Recent
  evaluations consistently put the single-vector / chunking break-even around 2500–3000 tokens
  for long-context embedders ([LongEmbed, NAACL 2024](https://arxiv.org/abs/2404.12096); SBERT
  "Does Chunking Matter?" 2024). So the corpus is mostly *below* the break-even — a universal
  chunking policy is overkill for the volume tail.
- **The 20% above break-even is where the cost lives.** PDFs and podcasts are exactly the
  classes where chunking pays. Concentrating chunking effort there is the high-leverage move.

---

## 3 · The query mix being served

Drawn from the premade prompts and from the post-hoc rubric log of real user queries. The
distribution is unusually attribution- and synthesis-heavy compared to standard RAG benchmarks
(which are dominated by single-span fact lookup).

| Query type | Example shape | Best retrieval shape | Why |
|---|---|---|---|
| **Named-source attribution** | "What does Bank 1 / Bank 4 / Indie Analyst N say about US economy" | **1-vector + author/source filter + reranker** | Doc gestalt and author voice carry the signal. Sub-doc chunks fragment authorship and force re-aggregation. |
| **Time-windowed multi-source synthesis** | "Last 24h on Iran / Brazil / oil" | **1-vector + time prefilter + reranker** | You want unique documents in the window. N chunks per doc = duplicate-doc noise. |
| **Thematic synthesis** | "Top 10 macro themes this week" | **1-vector, large k, strong synthesizer LLM** | Synthesis benefits from diversity *across* docs, not depth *within* docs. |
| **Day-over-day / weekly evolution** | "What's new vs yesterday" | **1-vector + time delta** | Pure set-difference problem. Chunking inflates the candidate pool. |
| **Comprehensive situation report** | "Where do we stand on X, prognosis?" | Mixed — 1-vector for surface, sub-doc for the long PDFs/podcasts that contain the deep arguments | The only common query that materially benefits from chunking. |
| **Fact / number extraction** | "Key numbers from today's data releases" | **Multi-chunk** when the numbers live in long PDFs/tables; otherwise 1-vector | Span-extraction. The classic chunking-wins case — but a small fraction of actual traffic. |
| **Direct quote retrieval** | "Find the quote where X said Y" | **Multi-chunk** (or BM25 alone) | Span-extraction. Same as above. |

The implication: **most M3xA queries do not look like the queries standard benchmarks
optimize for**. BEIR, MS MARCO, HotpotQA, and FinanceBench are dominated by needle-in-doc fact
lookup. M3xA traffic is mostly attribution + synthesis + recency. This shifts the cost/benefit
of chunking meaningfully — single-vector approaches are stronger on this distribution than the
"chunk everything" default would suggest.

Two real failure modes from the rubric log that *do* look like chunking problems:

- **A query about institutional views ("latest views from Bank 1 / Bank 4 on US economy")
  returned no results from a podcast that contained an extended segment from a Bank 4
  economist.** One vector per 25K-token episode is a black hole for named-guest queries inside
  long episodes — this is the textbook case for podcast text-chunking. Real chunking win.
- **A query mixed unrelated content ("Australia" appearing in an Iran-war report).** This looks
  more like a relevance/scoping problem than a chunking problem. Chunking might make it *worse*
  by inflating the candidate pool with marginal matches.

---

## 4 · The PDF question, directly

**Does breaking a PDF into more than one chunk make the system more efficient?**

For the M3xA PDF corpus (sell-side and buy-side research notes, 5K–25K tokens, structurally
rich, chart-and-table dense), the honest answer is:

- **Yes if your users ask passage-level questions.** "What did Bank 4 say about Brazil rates in
  Q3" — single-vector PDF cannot retrieve the Brazil-rates passage; the 25K-token average
  smears it into the dominant theme of the note. Multi-chunk is mandatory.
- **Yes if charts and tables carry signal.** A single text vector per PDF cannot retrieve a
  chart of "Bank 4's terminal-rate forecast across regions" because text extraction destroys
  the chart. ColPali / ViDoRe-pattern page-image embeddings ([Faysse et al., ECIR
  2025](https://arxiv.org/abs/2407.01449)) consistently beat OCR + text by 15–25 percentage
  points nDCG on chart/table benchmarks.
- **Defensible-as-alternative if your queries are mostly attribution.** "What did Bank 4
  publish last week" — single-vector PDF + an `author = Bank 4` filter + reranker handles this
  fine. The doc-level vector encodes "what this note is broadly about," and the metadata filter
  does the precision work.

The strongest design is therefore not "chunk vs. don't chunk" but the **triple layer**
proposed in the v5 architecture:

| Layer | What it stores | What query it serves |
|---|---|---|
| **A. Doc-level vector** (whole PDF text → 1 vector) | "This note is about X" | Attribution: "which Bank N notes mentioned X" |
| **B. Hierarchical text children** (parent 2000 tok / child 400–512 tok) | "On page 7, the note says specifically Y" | Passage retrieval: "what did Bank 4 say about Brazil rates" |
| **C. Page-image vectors** (each page as image → 1 vector via multimodal embedder) | The chart on page 12 | Visual retrieval: "show the chart of forecasts by region" |

The three layers are *complementary*, not competing. Each addresses a query class the other
two cannot serve. Storage cost is negligible at this corpus size (~50 PDFs/week × ~25 pages ×
~5 children + 1 doc-vec ≈ 6,500 vectors/week, ~2 GB/year at 1536d — well within budget). The
v5 spec gets this right.

**The one caveat:** the triple-layer pattern is high-ROI on PDFs specifically because PDFs are
where all three failure modes co-occur (long, dense, visual). It does **not** generalize. The
same triple-layer applied to wire articles or tweets would be expensive nonsense.

---

## 5 · Single vector per doc — pros and cons

**Pros**

- **Storage**: 1× baseline. With ~500 docs/day at 1536d, the table grows ~1 GB/year. Trivial.
- **Embed cost**: 1× baseline. One API call per doc on ingest.
- **Attribution queries are native**: filtering on `author = X` or `source = X` gives clean
  per-source result sets — no de-aggregation across chunks needed.
- **Recency queries are clean**: one document = one decision in the time-sorted result list. No
  duplicate-doc inflation.
- **Cross-doc dedup is trivial**: `id` is unique per document.
- **Eval interpretability**: a hit is unambiguous — "this doc was retrieved" rather than "chunk
  4 of 12 from this doc matched, was the doc actually relevant or was that one chunk a
  fluke?"
- **No chunk-boundary policy debates**: no parent/child mapping bugs, no recursive splitter
  tuning, no "did the section header land in chunk A or chunk B."
- **Aging gracefully**: corpus churn doesn't invalidate cross-chunk summaries (an issue for
  RAPTOR-style hierarchical summary trees).

**Cons**

- **Needle queries fail on long docs**: a specific number, name, or claim buried in a
  25K-token PDF will not surface — the doc-level vector averages it away.
- **Multi-hop synthesis underperforms**: queries that need to combine three passages from three
  different long docs are weaker because each long doc only contributes one (averaged) signal.
- **Hybrid BM25 becomes more important to compensate**: lexical retrieval picks up the proper
  nouns and numbers that the averaged dense vector misses. This is fine — BM25 is cheap — but
  the system depends on it more.
- **Degrades past the embedder's effective context**: even 128K-context embedders show
  needle-retrieval degradation past ~16K tokens
  ([NoLiMa, Modarressi et al., 2025](https://arxiv.org/abs/2502.05167)). For PDFs in the upper
  half of the size range, single-vector is genuinely weak.

---

## 6 · Multi-chunk (hierarchical) — pros and cons

**Pros**

- **Best retrieval quality on needle-in-doc and multi-hop tasks** — this is well-established
  across BEIR, LoTTE, FinanceBench, and the RAGBench paper
  ([Friel et al., 2024](https://arxiv.org/abs/2407.11005)).
- **Enables ColPali / page-image retrieval for visual content** — the single biggest known win
  for sell-side research PDFs.
- **Parent-doc retriever pattern is cheap and well-understood** — search on small children,
  return the parent window to the LLM. ~80% of the chunking benefit at ~20% of the complexity
  of RAPTOR-style hierarchies.
- **Allows chunk-level metadata** — timestamps on podcast chunks, section headings on PDF
  children, speaker tags. This enables retrieval-time filters that single-vector cannot
  support.

**Cons**

- **Storage**: 5–30× baseline. Tolerable for this corpus size; would be a real problem at
  10× the volume.
- **Embed cost on ingest**: 5–30× baseline. Real money if every chunk also incurs an LLM call
  for contextual prefix generation.
- **Attribution queries get harder**: a "what does Bank 1 say" query now returns 12 chunks
  from one Bank 1 note; the system must dedupe and re-aggregate by source before presenting.
- **Recency queries get noisier**: same docs appear multiple times in result lists; dedup
  becomes an explicit step.
- **Debug surface area expands**: "which chunk won, and was the doc actually relevant" becomes
  a recurring debugging question. Multiple production retrospectives (LlamaIndex Discord,
  r/LocalLLaMA) name this a top time-sink.
- **Chunk-boundary policy is a live decision**: fixed-size vs semantic vs section-aware. Each
  has known failure modes; tuning matters.

---

## 7 · Late chunking — the underrated middle path

A pattern less prominent in the v5 spec but well-supported in recent literature.

**The technique** ([Günther et al., arXiv:2409.04701](https://arxiv.org/abs/2409.04701)): encode
the **whole document once** using a long-context embedder, then mean-pool token embeddings over
chunk-sized windows. Each "chunk vector" carries cross-chunk context (anaphora, references to
earlier sections) for free, because every token's contextual embedding was computed with the
full document in attention. Independently reproduced by LlamaIndex (late 2024) and Weaviate
(Q1 2025), with ~3–8 pp nDCG@10 lift over naive fixed-size chunking on long-doc subsets of
BEIR.

**Why it matters here**: it gets most of the "contextual chunk" benefit *without* the per-chunk
LLM call for prefix generation. The v5 spec's universal Haiku-prefix line item (~$75/month
ongoing, ~$300+ for the one-time backfill) becomes optional if late chunking is available on
the chosen embedder.

**The catch**: the technique needs token-level outputs from the embedder, which not all hosted
APIs expose. Worth a Phase 0 spike on the chosen embedder to confirm.

**Pros**

- One embed call per doc (1× cost), N vectors out (one per chunk window).
- Cross-chunk context for free — no LLM prefix needed for the same retrieval lift.
- Compatible with parent-doc retriever pattern (parent = whole doc, children = late-chunked
  windows).

**Cons**

- Requires API access to token-level embeddings (not always available).
- Less battle-tested in production than hierarchical chunking; fewer reference implementations.
- Doesn't help the visual-PDF case — page-image embedding is orthogonal and still needed.

---

## 8 · Anthropic Contextual Retrieval — what holds up and what doesn't

The proposed architecture applies Anthropic's contextual-retrieval prefix
([blog, Sept 2024](https://www.anthropic.com/news/contextual-retrieval)) universally on every
chunk. The published headline numbers: contextual embeddings alone reduce retrieval failures
by 35%; with contextual BM25 added, 49%; with reranker added, 67%.

**Reproductions through mid-2026 — the consensus has nuance:**

- **Weaviate (Oct 2024)** reproduced the embeddings-only condition on FinanceBench and PubMedQA:
  ~25–35% failure reduction, broadly consistent with Anthropic.
- **LlamaIndex (Nov 2024)** showed lift on internal docs but found the magnitude depends
  heavily on chunk size: at 800-token chunks the lift compressed to ~15% because chunks were
  already self-contained; at 200-token chunks the lift matched Anthropic's 35%.
- A critical reproduction circulating in early 2025 RAG engineering (Pinecone-style "revisited"
  analyses, also discussed at length on the LlamaIndex Discord and in the RAGBench paper)
  argued that **a strong reranker captures most of the gains without the per-chunk LLM call**:
  ~55% failure reduction from hybrid + Cohere Rerank 3.5 alone, vs. Anthropic's 67% with the
  prefix added. The prefix contributes ~12 pp on top of an already-strong rerank — meaningful
  but expensive.

**When the prefix is worth it:**

- Chunk size ≤ 500 tokens.
- Corpus with heavy anaphora ("the company reported", "the chairman said", "this policy") and
  ambiguous chunks.
- Domains where BM25 has lexical traction (finance, legal — Anthropic chose these for a
  reason).

**When it isn't:**

- Chunks already > 800 tokens (the v5 spec's parent chunks are 2000 tokens — well into the
  diminishing-returns zone).
- Conversational or narrative chunks that are already self-contained.
- Tweets, short wires, single-author short notes — adding a Haiku prefix to a 60-token tweet is
  wasted spend.

**Concrete pushback on universal application**: at the v5 spec's chunk sizes, the contextual
prefix likely contributes single-digit percentage points beyond a strong reranker — at a
~$75/month ongoing cost plus a ~$300+ one-time backfill cost. The same dollars buy more
quality if redirected to a Phase 0 golden-set eval that *measures* the lift before committing.

---

## 9 · Where this study agrees with the proposed v5 architecture

- **PDF triple-layer (doc + text-children + page-image) is the strongest case for chunking in
  the corpus.** Aligns with ColPali / ViDoRe evidence; this is the right pattern for sell-side
  research.
- **Podcast text chunking is mandatory.** A single vector per 25K-token episode is a known
  failure mode (rubric log entry, May 2026: institutional-view query returned nothing from a
  podcast containing the relevant segment). ~600-token chunks with speaker name prepended is
  the cheap, well-supported fix.
- **Hybrid BM25 + dense + per-language stems** is exactly what the contextual-retrieval
  reproductions converge on as the high-ROI core.
- **Strong reranker (Cohere Rerank 3.5 or equivalent) is the single highest-ROI component.**
  Defending that spend before any other.
- **Editorial multipliers applied *after* reranking, not as a replacement.** Correct ordering;
  the rerank score is the strongest signal and shouldn't be overridden.

---

## 10 · Where this study suggests recalibrating the v5 architecture

Three concrete adjustments, each backed by mid-2026 evidence:

### 10.1 Raise the hierarchical-chunking threshold from 1500 → ~2500–3000 tokens

The v5 spec applies hierarchical chunking to any wire article over 1500 tokens. LongEmbed
(NAACL 2024) and the SBERT "Does Chunking Matter?" benchmark both put the single-vector /
chunking break-even at ~3K tokens for long-context embedders. Cohere v4 (the chosen embedder)
has 128K nominal context; effective context is well above 3K. Raising the threshold removes
hierarchical complexity from a slice of the corpus where it does not pay.

### 10.2 Apply the Haiku contextual prefix selectively, not universally

Concentrate the prefix on:

- PDF text-children (long docs, dense anaphora — where the prefix actually lifts retrieval)
- Podcast chunks (chunks lack obvious self-contained context without speaker tagging)

Skip it on:

- Tweets (already a chunk; nothing to contextualize)
- Wire articles (inverted-pyramid structure self-contextualizes)
- Email research notes under ~3K tokens (author voice carries the context the prefix would
  add)

Estimated cost reduction: ~60–70% of the ~$75/month prefix line item, with negligible
retrieval impact on the queries that dominate traffic.

### 10.3 Spike late chunking on Cohere v4 before committing to universal hierarchical + prefix

If Cohere v4's Bedrock endpoint exposes token-level outputs (verify), late chunking is the
cheaper substitute for "hierarchical + contextual prefix" on the 3K–8K class (long-form
journalism, long email notes). One embed call per doc, N vectors out, cross-chunk context for
free. The Phase 0 eval can directly compare:

- Late chunking
- Hierarchical + Haiku prefix
- Hierarchical without prefix

and pick the best on the golden set rather than committing to the most expensive option by
default.

---

## 11 · What to actually build — recommendation

A **two-tier index** that defaults to single-vector and chunks selectively:

| Tier | What it stores | When it's queried |
|---|---|---|
| **Tier 1 (default)** | One vector per doc, all classes, long-context embedder | Every query |
| **Tier 2 (selective)** | Sub-doc vectors for PDFs, podcasts, and long-form > 3K tokens | When Tier 1 confidence is low, or when the query classifier flags fact-lookup / multi-hop |
| **Tier 3 (PDFs only)** | Page-image vectors via multimodal embedder | When the query mentions a chart/table/visual element, or as a parallel leg for all PDF queries |

Concretely:

1. **Default everything to 1 vector per doc.** Tweets, wires, emails under 3K, op-eds under 3K,
   geo OSINT events — all single-vector.
2. **Multi-chunk PDFs with the full triple layer** (doc-vec + hierarchical text-children +
   page-image). This is where chunking pays. The v5 spec design is correct here.
3. **Multi-chunk podcasts** at ~600-token (~5-minute) boundaries with speaker name prepended.
   No LLM-generated prefix needed — the speaker name plus the parent-episode metadata gives
   the context.
4. **Long-form > 3K tokens**: late-chunk if the embedder supports token-level outputs;
   otherwise hierarchical without the Haiku prefix.
5. **Apply Haiku contextual prefix only to PDF text-children**, where the combination of long
   doc + dense anaphora + structurally complex content actually rewards it. Skip everywhere
   else.
6. **Run a strong reranker on every query.** This is the single biggest lever and the v5 spec
   already has it. Defend that spend before any other.
7. **Defer the Nova audio layer for podcasts** until a Phase 0 eval shows text-path failures on
   tonal / non-verbal queries. The text path with timestamps + speaker tags handles the
   documented failure modes today; the audio path is a speculative add with no production
   reproduction.
8. **Build the Phase 0 golden set first.** Without it, every claim above — including this
   document's recommendations — is unmeasured. The golden set converts every right-column
   number from a hypothesis into a measurement.

---

## 12 · Honest caveats

- **No live leaderboard verification.** Rankings on MTEB, BEIR, ViDoRe, RAGBench, and FinanceBench
  shift between releases. The directional claims here are robust; specific numbers may have
  moved.
- **The "two-tier" recommendation is a synthesis** of converging practice across Jina,
  LlamaIndex, Weaviate, and the RAGBench evaluations — not a single benchmarked SOTA paper.
  It reflects what production systems are settling on in 2025–2026, not a published
  prescription.
- **The contextual-retrieval critique is the most contested point in this document.** Anthropic
  reports 67% failure reduction with full hybrid + rerank + prefix; the "rerank alone gets you
  to ~55%" claim is widely circulated in 2025 RAG engineering but not yet formalized in a peer-
  reviewed paper. The honest framing is: the prefix probably contributes meaningful lift on top
  of strong rerank, but the magnitude is corpus-dependent and the only way to know for *this*
  corpus is to measure on the golden set.
- **Cost numbers are order-of-magnitude.** Embed and rerank pricing change quarterly; the
  *shape* of the trade-offs is more stable than any specific dollar figure.
- **This document is publication-safe by construction.** Source names are anonymized (Bank N /
  Wire N / Pub N / Indie N); vendor names mentioned (Cohere, Voyage, Anthropic, Amazon Nova,
  Jina) are public. No internal hostnames, no journalist names, no user handles, no real bank
  identities appear.

---

## 13 · References

**Primary technique papers**

- Günther et al., *Late Chunking: Contextual Chunk Embeddings Using Long-Context Embedding
  Models*, arXiv:2409.04701 (2024) — https://arxiv.org/abs/2409.04701
- Sturua et al., *Jina Embeddings v3: Multilingual Embeddings With Task LoRA*, arXiv:2409.10173
  (2024) — https://arxiv.org/abs/2409.10173
- Faysse et al., *ColPali: Efficient Document Retrieval with Vision Language Models*,
  arXiv:2407.01449 (ECIR 2025) — https://arxiv.org/abs/2407.01449
- Sarthi et al., *RAPTOR: Recursive Abstractive Processing for Tree-Organized Retrieval*,
  arXiv:2401.18059 (ICLR 2024) — https://arxiv.org/abs/2401.18059
- Khattab & Zaharia, *ColBERTv2: Effective and Efficient Retrieval via Lightweight Late
  Interaction*, arXiv:2112.01488 (2022) — https://arxiv.org/abs/2112.01488

**Benchmarks and evaluations**

- Zhu et al., *LongEmbed: Extending Embedding Models for Long Context Retrieval*,
  arXiv:2404.12096 (NAACL 2024) — https://arxiv.org/abs/2404.12096
- Modarressi et al., *NoLiMa: Long-Context Evaluation Beyond Literal Matching*,
  arXiv:2502.05167 (2025) — https://arxiv.org/abs/2502.05167
- Friel et al., *RAGBench: Explainable Benchmark for Retrieval-Augmented Generation Systems*,
  arXiv:2407.11005 (2024) — https://arxiv.org/abs/2407.11005
- Islam et al., *FinanceBench: A New Benchmark for Financial Question Answering*,
  arXiv:2311.11944 (2023) — https://arxiv.org/abs/2311.11944

**Industry references (cited as biased but informative)**

- Anthropic, *Contextual Retrieval*, blog post (Sept 2024) —
  https://www.anthropic.com/news/contextual-retrieval

---

*Companion to [README.md](README.md). Both files are publication-safe by construction.*
