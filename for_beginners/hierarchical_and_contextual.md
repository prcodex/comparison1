# Hierarchical Chunking and Contextual Retrieval — A Beginner-Friendly Walkthrough

> **Audience:** an engineer new to RAG (Retrieval-Augmented Generation) who has seen these
> terms thrown around — "hierarchical chunking," "contextual retrieval," "parent 2000 / child
> 400" — but doesn't yet have a mental model that ties them together. This file builds that
> model from scratch, with concrete examples and small enough steps that nothing has to be
> taken on faith.
>
> **Companion to** the main repo files [README.md](../README.md),
> [chunking_study.md](../chunking_study.md), and [haiku_cleaning.md](../haiku_cleaning.md),
> which assume RAG background and skip the basics. This file fills in those basics.

---

## Part 1 · The basics in plain English

### What is a token?

A **token** is the unit a language model counts. Roughly, one token is about ¾ of an
English word, or one short word, or about four characters. So:

- A 60-word tweet is ≈ 80 tokens
- A 1,500-word op-ed is ≈ 2,000 tokens
- A 20-page PDF research note is ≈ 15,000–25,000 tokens

When someone says "a 2,000-token chunk," they mean a piece of text that's roughly **1,500
English words** or **3 pages of normal prose**. When they say "a 400-token chunk," they mean
about **300 words** or **half a page**. Those are the only two numbers you need to keep in
your head for everything that follows.

### What is an embedding (a vector)?

An **embedding** is a list of numbers that represents the *meaning* of a piece of text. For
the embedders this system uses (Voyage-3-large, Cohere v4), that list is 1,024–2,048 numbers
long. Two pieces of text that mean similar things produce embeddings that are *close to each
other* (measured by cosine similarity).

You take a piece of text, you call an API, you get back a vector. The vector is the "address"
of that text's meaning in a high-dimensional space.

### What does "retrieval" mean here?

RAG works by:

1. Pre-processing: split your documents into pieces, embed each piece, store the (vector,
   text, metadata) rows in a database.
2. Query time: take the user's question, embed it the same way, find the rows whose vectors
   are closest to the question's vector, hand those rows' *text* to a language model to write
   the answer.

The whole game is: **make sure the right text gets retrieved when the question is asked.** If
the right paragraph is in your database but the vector lookup doesn't surface it, the LLM
never sees it and can't answer correctly.

### The one-vector-per-document baseline

The simplest possible setup: each ingested document gets **one vector** that represents
"what this document is about." You search by embedding the question and finding the documents
whose vectors are closest.

This works fine for short documents. It breaks for long documents because **a single vector
that has to represent a 15,000-token document becomes a "blurry average"** of every topic
the document covers. A 20-page bank note that covers Brazil rates, Mexico growth, US 10Y
yields, AUD positioning, and PBOC moves gets represented as one point — and that point is
the centroid of all five topics. A question about Brazil rates may not match it well because
the vector "isn't really about Brazil rates" — it's about all five things weighted together.

That's the problem the rest of this file is about solving.

---

## Part 2 · Why one rule can't fit all documents (the "per-source-class matrix")

The documents an intelligence system ingests vary enormously in shape:

| Kind of document | Typical length | What it looks like |
|---|---|---|
| Tweet | ~60 tokens | A single sentence |
| Wire-service article | ~800 tokens | A few short paragraphs |
| Op-ed / Substack column | ~5,000 tokens | ~7 pages of opinion |
| Email research note | ~3,000 tokens | Sectioned text with subheaders |
| Sell-side bank PDF | ~15,000 tokens | 20-page note with charts and tables |
| Podcast transcript | ~12,000 tokens | Multi-speaker, time-structured |

A 60-token tweet **does not need chunking.** It's already a "chunk." Splitting it into smaller
pieces is nonsense — there's no meaningful sub-structure inside one sentence.

A 15,000-token bank PDF, if you embed it as a single vector, becomes the blurry-average
problem from §1. You need to split it.

**One rule cannot be right for documents that span 60 tokens to 25,000 tokens.** So the
solution is to write down, for each *kind* of source, what chunking strategy applies. That
table is what the literature calls a **per-source-class matrix**.

An example matrix (this is what the proposed v5 architecture roughly looks like):

```
Tweet                  → 1 vector per doc, no split
Wire article           → 1 vector per doc, no split
Op-ed / long article   → hierarchical (parent 2,000 / child 400-512)
Email research note    → section-aware split (use the email's headers)
Bank PDF               → triple-layer: doc-level vector
                                       + hierarchical text-children
                                       + page-image vector per page
Podcast transcript     → speaker/timestamp boundaries (~600 tok per chunk)
```

That table **is** the matrix. Each row says: "for this kind of source, use this chunking
rule." The matrix is the recognition that there is no universal answer — only document-shape-
specific answers.

The rest of this file goes deep on one row of that matrix: **hierarchical (parent 2,000 /
child 400-512)**, which is the most common and most-discussed strategy.

---

## Part 3 · Hierarchical chunking, in detail

### The textbook analogy

Think of how you'd find something in a textbook:

- A textbook has **chapters** (long, give you context).
- Each chapter has **paragraphs** (short, precise).
- When someone asks "where does this book discuss World War 1?", you mentally scan
  *paragraphs* to find the right one (paragraphs are precise — each one is about one thing).
- But once you've found the right paragraph, you don't read just that paragraph in isolation
  — you read **the whole chapter it's in**, because that gives you the framing the paragraph
  was assuming.

**Hierarchical chunking does exactly that to a document.** You store two levels of pieces:

- **Parent chunk** ≈ **2,000 tokens** (≈ 1,500 words, ≈ 3 pages) — the "context unit." This
  is what the LLM eventually reads when generating its answer.
- **Child chunk** ≈ **400–512 tokens** (≈ 300–400 words, ≈ half a page) — the "search unit."
  This is what gets embedded and matched against queries.

### The key trick: search small, return big

Here's the picture of how a single 5,000-token document gets sliced:

```
Document (5,000 tokens — one bank's weekly EM strategy note)
│
├── Parent A (tokens 0–2000) ────── what the LLM eventually reads
│   ├── Child 1 (tokens 0–400)  ──── what gets embedded + searched
│   ├── Child 2 (tokens 350–750) ─── chunks overlap a bit (~80 tok)
│   ├── Child 3 (tokens 700–1100)
│   ├── Child 4 (tokens 1050–1450)
│   └── Child 5 (tokens 1400–1800)
│
├── Parent B (tokens 1800–3800) ─── parents also overlap (~200 tok)
│   ├── Child 6, 7, 8, 9
│
└── Parent C (tokens 3600–5000)
    └── Child 10, 11, 12
```

What's stored in the vector database:

- 12 child vectors (each represents ~400 tokens of text)
- 3 parent texts (each ~2,000 tokens), linked to their children by ID

At query time:

1. User asks: *"What did the bank say about Brazil rates?"*
2. The system embeds the question and compares it against **every child vector** (12
   comparisons for this document; millions across the whole database).
3. Best match — say, **Child 7** — has very high similarity. Child 7 happens to contain the
   Brazil-rates paragraph.
4. The system does **not** return Child 7 alone to the LLM. Child 7 is only 400 tokens —
   maybe missing context like "this Brazil discussion is in the context of broader sovereign-
   risk views in the section above."
5. Instead, the system looks up **Parent B** (the 2,000-token parent that *contains* Child 7)
   and returns *that* to the LLM.
6. The LLM reads Parent B and answers with full surrounding context.

**Search happens at the child level (precision); reading happens at the parent level
(context).** That's the entire trick.

### Why these specific numbers?

The numbers (2,000 for parent, 400–512 for child) come from empirical benchmarks across
multiple 2023–2025 evaluations on long-document retrieval:

- **400–512 tokens for the child**: the sweet spot for dense embeddings in finance/news
  content. Smaller chunks (say 100 tokens) become fragmentary — half-sentences embed poorly.
  Bigger chunks (say 1,500 tokens) start mixing multiple topics, and the embedding becomes a
  blurry average for that smaller scale.
- **2,000 tokens for the parent**: roughly the maximum amount of context that's actually
  useful for the LLM to read to answer one passage-level question. Bigger wastes context-
  window budget on irrelevant material; smaller starts losing the surrounding context that
  made the chunk meaningful.

These are defaults, not laws. Different document classes can use different numbers:
podcasts work better with ~600-token chunks aligned to speaker turns; very long PDFs
sometimes use a 3,000/600 ratio. The matrix lets you tune per source class.

### Why overlap?

You'll notice in the diagram that Child 1 ends at token 400 and Child 2 starts at token 350.
The 50-token overlap is deliberate. Reason: if a key sentence happens to land exactly at
token 400 (the chunk boundary), a naive split would cut the sentence in half — neither chunk
would contain the full thought, and neither would embed well for that idea.

With overlap, the second chunk re-includes the last bit of the previous one, so no thought
gets cut at a boundary. Same logic applies between parents (~200 token overlap there). The
overlap is "the seam allowance" of chunking.

### Worked example — same document, two strategies

Imagine a 4,500-word sell-side weekly note. Sections inside it:

- Introduction (300 words)
- Brazil: rates view (800 words)
- Mexico: growth outlook (600 words)
- LatAm FX positioning (500 words)
- Asia: PBOC moves (700 words)
- Conclusions (200 words)

Total: ~3,100 words ≈ 4,200 tokens.

**Strategy A: one vector per document.**

```
[entire 4,200-token note] ──── embed ────► one 2,048-dim vector
```

The Brazil-rates section is averaged together with Mexico, FX, Asia, and the conclusion.
The single vector represents the *centroid* of all five topics. A query about "Brazil rates"
gets a mediocre match because the vector "isn't really about Brazil rates" — it's about all
five things weighted by their share of the document.

**Strategy B: hierarchical (parent 2,000 / child 400-512).**

```
Parent A: [Intro + Brazil + half of Mexico]  (~2,000 tok)
  ├ Child 1: "Introduction"                            (~400 tok) → vector
  ├ Child 2: "Brazil rates — first half"               (~400 tok) → vector
  ├ Child 3: "Brazil rates — second half + Mexico"     (~400 tok) → vector
  ├ Child 4: "Mexico growth — first half"              (~400 tok) → vector
  └ Child 5: "Mexico growth — second half"             (~400 tok) → vector

Parent B: [rest of Mexico + LatAm FX + half of Asia]  (~2,000 tok)
  ├ Child 6, 7, 8, 9, 10                                          → vectors

Parent C: [rest of Asia + Conclusions]                (~1,200 tok)
  └ Child 11, 12, 13                                              → vectors
```

Query: *"What does this bank say about Brazil rates?"*

- Compute query embedding.
- Compare against 13 child vectors.
- Best matches: **Child 2** and **Child 3** (both contain Brazil-rates content) with very
  high similarity scores.
- Return **Parent A** (the parent that contains both Children 2 and 3, plus the introduction
  that frames the discussion).
- LLM reads Parent A → has the full Brazil-rates section plus the framing context.

The hierarchical strategy retrieves a *specific passage* by searching small, then gives the
LLM *full context* by returning the parent. Strategy A can only retrieve the whole document
or nothing.

### When hierarchical chunking pays off

It pays when:

- Documents are long (≥ ~3,000 tokens) — short docs have nothing to gain.
- Documents are topically heterogeneous (one note covers multiple distinct topics, like the
  EM weekly above).
- Queries ask about specific passages, not document-level themes. ("What did the bank say
  about Brazil rates?" pays. "Which banks publish weekly EM notes?" does not.)

It does not pay when:

- Documents are short (tweets, wires) — no internal structure to recover.
- Documents are about one topic end-to-end (most op-eds, single-thesis Substack columns) —
  the doc-level vector represents the whole thing well already.
- Queries are attribution-shaped ("what does Author X think") — the document is the unit;
  splitting it just creates duplicate-author noise in the result list.

That's why hierarchical chunking belongs to specific *rows* of the per-source-class matrix
(long PDFs, podcasts, long-form articles) rather than being a universal default.

---

## Part 4 · Contextual retrieval, in detail

Hierarchical chunking solves one problem (long docs need to be searched at small granularity
and returned at large granularity). It does **not** solve another problem: **chunks lose
context when separated from their document.**

### The chunks-lose-context problem

Consider this chunk, taken from the middle of a sell-side note:

> *"The committee shifted to a more dovish stance this quarter, with three of seven members
> now favoring a 25 bp cut at the next meeting. Inflation prints have undershot the central
> projection for two consecutive months. The next decision is scheduled for September 18."*

If you embed this chunk and someone searches "Brazil central bank dovish shift," does this
chunk get retrieved?

**It depends.** The chunk doesn't say "Brazil" anywhere. It doesn't say "central bank." It
says "the committee" and "the next meeting." Those phrases are unambiguous *to a reader of
the whole document* — the surrounding paragraphs made clear which committee, in which
country. But to a search engine looking at *this chunk in isolation*, the words "Brazil" and
"central bank" are missing, so the embedding doesn't lean strongly toward them, and the chunk
may not surface for that query.

This is the problem **Anthropic's Contextual Retrieval** technique is designed to solve.

### The technique

Before embedding each chunk, you call a small fast LLM (Claude Haiku) and ask it to write a
**short prefix** (50–100 tokens) that situates the chunk inside its document. You then prepend
the prefix to the chunk and embed the combined text. The prefix is small enough not to dilute
the chunk's content but specific enough to add the missing context.

The Haiku call uses Anthropic's published prompt template:

```
<document>
{WHOLE_DOCUMENT}
</document>

Here is the chunk we want to situate within the whole document:
<chunk>
{CHUNK_CONTENT}
</chunk>

Please give a short succinct context to situate this chunk within
the overall document for the purposes of improving search retrieval
of the chunk. Answer only with the succinct context and nothing else.
```

For the "committee shifted to dovish" chunk above, Haiku might return:

> *"This chunk is from a sell-side weekly note dated 2026-06-12 discussing the Banco Central
> do Brasil COPOM meeting on September 18, in the section reviewing the recent dovish shift
> among committee members."*

You then store and embed:

```
{Haiku-generated prefix}

{original chunk text}
```

Now when someone searches "Brazil central bank dovish shift," the prefix contains "Banco
Central do Brasil," "COPOM," and "dovish shift" — the chunk surfaces.

### The published lift numbers

Anthropic's September 2024 blog post reported, on internal benchmarks:

- Contextual embeddings alone: **35% reduction** in retrieval failures (failure rate dropped
  from 5.7% to 3.7%).
- + Contextual BM25 (lexical search on the prefix+chunk): **49% reduction** (to 2.9%).
- + Reranker on top: **67% reduction** (to 1.9%).

These are headline numbers from the vendor. They've been partially reproduced (Weaviate,
LlamaIndex, late 2024) with the consensus that the *direction* is robust but the *magnitude*
depends heavily on chunk size and corpus.

### When contextual retrieval pays off

It pays most when:

- **Chunks are small** (≤ ~500 tokens). At 200-token chunks, the published 35% lift
  reproduces well — chunks really do lack context at that size.
- **Documents have heavy anaphora.** "The committee," "this view," "the company" — pronouns
  and references that depend on the surrounding document. Finance and legal docs are full
  of this.
- **Corpus has lexical traction.** When proper nouns and specific terms matter (BCB, COPOM,
  ticker symbols, person names), the prefix that injects those terms into BM25's lexical
  index is high-value.

It pays less (or not at all) when:

- **Chunks are already large** (≥ ~800 tokens). LlamaIndex's reproduction in late 2024 showed
  the prefix lift compresses to ~15% at 800-token chunks, because chunks at that size are
  already mostly self-contained.
- **The corpus is conversational or narrative** and each chunk is already a complete thought.
- **Docs are tiny** (tweets, single-sentence wires). The chunk *is* the doc.

### The cost

Every chunk needs its own Haiku call to generate the prefix. That cost scales with the
number of *chunks*, not the number of *documents*. If your average document produces 5
chunks, the per-chunk Haiku spend is 5× the per-document spend.

At the volumes this system ingests (~500 docs/day, ~5 chunks per doc on average for long-doc
classes), contextual retrieval costs around **$75/month** in Haiku calls. This is the
single most expensive component of the contextual-v5 architecture proposed in
[chunking_study.md](../chunking_study.md), and the main reason the study argues for
**selective** application rather than universal.

---

## Part 5 · Putting it together — the three-stage stack

The proposed v5 architecture combines **three separate Haiku-based techniques**, applied at
three different stages of ingestion. People often conflate them because they all use Haiku
somewhere; they are in fact distinct operations operating at different granularities. Knowing
them apart is the single most useful clarification this document offers.

| | What it does | When | Granularity |
|---|---|---|---|
| **(1) Per-document Haiku cleaning** (already in production) | Strips boilerplate, standardizes formatting | At ingest, **before** chunking | Once **per document** |
| **(2) Hierarchical chunking** | Splits long docs into parent (2,000) + child (400–512) pieces | At ingest, **after** cleaning | N parents + M children **per document** |
| **(3) Contextual retrieval prefix** | Generates a 50–100 token "where does this chunk fit" prefix | At ingest, **after** chunking, before embedding | Once **per chunk** |

### What happens to a single 5,000-token bank note under the full v5 stack

```
Raw PDF text
  │
  ▼
(1) Haiku CLEAN   ─────────────────  ~$0.001 per doc       ← already in prod
    (preserve-all mode: remove
     boilerplate, disclaimers,
     legal footers)
  │
  ▼
(2) Hierarchical SPLIT
    → Parent A, B, C  (~2,000 tok each)                    ← chunking
    → Child 1..15     (~400–512 tok each)
  │
  ▼
(3) For EACH of 15 children:
    Haiku generates a 50–100 tok
    "this chunk is from {note} on {date},
     section {X}, discussing {Y}" prefix    ── ~$0.001 per CHUNK
  │
  ▼
For each child:  prefix + child text  →  Embed  →  vector  →  stored in DB
For each parent: parent text                              ─►  stored in DB
                                                              (no embedding)
```

### The cost asymmetry

This is the most important practical fact in this whole document. Step (1) scales **per
document**; step (3) scales **per chunk**. If a typical document produces 5 chunks, step (3)
costs ~5× what step (1) costs. At system-wide volumes:

- Step (1) per-doc cleaning: ~$4/month total
- Step (3) per-chunk contextual prefix: ~$75/month

That's a ~20× cost gap, with step (3) being the expensive one. Worth defending only where it
actually pays.

### Where this lands in the recommendation

[chunking_study.md](../chunking_study.md) argues for keeping (1) and (2) but applying (3)
**selectively**, not universally:

- **Keep (1) everywhere.** It's cheap, already in production, and improves both single-vector
  and multi-chunk approaches equally because both consume the cleaned text.
- **Keep (2) only on long docs** (PDFs, podcasts, long-form > 3,000 tokens). Short docs gain
  nothing from chunking; applying it to them is overhead without benefit.
- **Apply (3) only on PDF text-children, maybe podcast chunks.** These are the document
  classes where (a) chunks are most likely to lack self-contained context, (b) anaphora
  ("the bank," "this view") is densest, and (c) the corpus is heterogeneous enough that the
  prefix actually lifts retrieval.

Three reasons the v5 spec's *universal* stacking of (3) is probably overkill:

1. **Diminishing returns at v5's chunk sizes.** LlamaIndex's reproduction showed the
   contextual-prefix lift drops from ~35% at 200-token chunks to ~15% at 800-token chunks.
   The v5 child size (400–512 tok) is in the middle of that decay; parents (2,000 tok) are
   well past it.
2. **A strong reranker eats most of the gain.** Independent 2025 reproductions found that
   hybrid retrieval + Cohere Rerank 3.5 alone achieves ~55% failure reduction, vs.
   Anthropic's 67% with the prefix added. The prefix is worth ~12 pp on top of strong rerank
   — real, but not transformative at universal scale.
3. **Step (1) already solves part of what (3) is solving.** Anaphora and boilerplate noise
   that (3) is designed to address at the chunk level have already been reduced by the
   per-document cleaning pass. Stacking them gives diminishing returns.

The honest framing: **(2) is the technique that solves the "blurry-average" problem for long
documents; (3) is a separate, smaller technique that fixes a residual problem with chunks
losing context after splitting.** If you're not chunking, (3) is N/A. If you are chunking,
(3) is an *optional refinement* whose marginal value depends on chunk size, doc class, and
whether you already run (1) and a strong reranker. The right way to settle "should we use
(3) here?" is to measure it on the Phase 0 golden set, not to assume.

---

## Part 6 · Glossary (one-line definitions)

- **Token** — the smallest unit a language model counts. Roughly ¾ of an English word.
- **Chunk** — a piece of a document, sized for either embedding or LLM reading.
- **Embedding (vector)** — a list of numbers representing the meaning of a piece of text.
  Used to find semantically similar text via cosine similarity.
- **Retrieval** — given a question, find the documents/chunks in the database whose vectors
  are closest to the question's vector.
- **Reranker** — a separate model that re-scores a list of candidate chunks for relevance
  given the query. Run *after* vector retrieval, *before* sending to the LLM.
- **Parent chunk** — a larger piece (~2,000 tokens) used as the unit of *context* given to
  the LLM.
- **Child chunk** — a smaller piece (~400–512 tokens) used as the unit of *search* (the
  thing that gets embedded and matched).
- **Hierarchical chunking** — the strategy of storing both parent and child chunks: search
  on children, return parents.
- **Contextual retrieval** — Anthropic's technique of prepending an LLM-generated context
  prefix to each chunk before embedding, to compensate for context lost by splitting.
- **Per-source-class matrix** — a table that maps each kind of source (tweet, wire, op-ed,
  PDF, podcast, etc.) to the chunking strategy appropriate for that kind.
- **Anaphora** — when a chunk uses references like "the committee," "this view," "the
  company" that depend on something earlier in the document for their meaning. The main
  reason chunks lose context when split out.
- **BM25** — a classical keyword-based search algorithm. Often combined with dense vector
  search ("hybrid search") because it catches proper nouns and specific terms that dense
  embeddings can blur.
- **Hybrid search** — running BM25 (keyword) and dense (vector) retrieval in parallel and
  combining the results, usually via Reciprocal Rank Fusion (RRF).

---

*Companion to the main repo files. Both the broader study
([chunking_study.md](../chunking_study.md)) and the existing per-document cleaning pattern
([haiku_cleaning.md](../haiku_cleaning.md)) build on the concepts introduced here.*
