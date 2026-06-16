# Per-source-class matrix · Hierarchical parent 2,000 / child 400–512

A walk-through from scratch, with concrete examples.

---

## Part 1: What "per-source-class matrix" means

It's not a special technique — it's just a **policy table**. The rows are the different kinds
of documents you ingest. The columns are the chunking choices made for each kind. Instead of
one rule for everything, you have a rule per kind.

Why have a matrix at all? Because the documents you ingest have wildly different shapes:

| Kind of doc | Typical length | What it looks like |
|---|---|---|
| Tweet | ~60 tokens | One sentence |
| Wire article | ~800 tokens | A few short paragraphs |
| Op-ed / Substack column | ~5,000 tokens | ~7 pages of opinion |
| Sell-side email research | ~3,000 tokens | Sectioned text with subheaders |
| Bank PDF | ~15,000 tokens | 20-page note with charts |
| Podcast transcript | ~12,000 tokens | Multi-speaker, time-structured |

A tweet is already small enough that one vector represents it well — splitting it makes no
sense. A 15,000-token PDF, if you embed it as one vector, becomes a "blurry average" of 5–10
different topics — you'd lose any ability to find a specific passage. **One rule cannot be
right for both ends of that range.** So you write a matrix:

```
Tweet         → 1 vector, no split
Wire article  → 1 vector, no split
Op-ed         → hierarchical (parent 2,000 / child 400-512)
Email research → section-aware split (use the email's headers)
Bank PDF      → triple-layer (doc-vector + hierarchical text + page-images)
Podcast       → speaker/timestamp boundaries (~600 tok per chunk)
```

That table **is** the matrix. Each row is a "source class" (kind of source). Each row gets
the chunking strategy that fits *its* shape.

---

## Part 2: What "hierarchical parent 2,000 / child 400–512" means

Think of it like a textbook:

- A textbook has **chapters** (big, give you context).
- Each chapter has **paragraphs** (small, precise).
- When you search "where does the book talk about World War 1?", you scan *paragraphs* to
  find the right one. But then you read the *whole chapter* containing that paragraph to
  actually understand it.

Hierarchical chunking does exactly that to a document:

- **Parent chunk** ≈ **2,000 tokens** ≈ ~1,500 words ≈ ~3 pages. This is the "context unit."
  It's the size of text the LLM gets to *read* when generating an answer.
- **Child chunk** ≈ **400–512 tokens** ≈ ~300–400 words ≈ ~half a page. This is the "search
  unit." It's the size of text that gets *embedded* and *matched against queries*.

### The key trick: search small, return big

```
Document (5,000 tokens — one bank's weekly EM strategy note)
│
├── Parent A (tokens 0–2000) ────── what the LLM reads
│   ├── Child 1 (tokens 0–400)  ──── what gets embedded + searched
│   ├── Child 2 (tokens 350–750) ─── (chunks overlap a bit, ~80 tok)
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

At query time:

1. User asks: *"What did the bank say about Brazil rates?"*
2. The system embeds the query and compares it against **every child vector**.
3. Best match → say, **Child 7** (which happens to contain the Brazil rates paragraph).
4. The system does **not** return Child 7 alone (it's only 400 tokens — might be missing
   context like "this is the rates section, in the context of the Brazil sovereign
   discussion above").
5. Instead, it returns **Parent B** — the 2,000-token chunk that *contains* Child 7.
6. The LLM reads Parent B and answers with full surrounding context.

So **search happens at the child level (precision); reading happens at the parent level
(context).**

### Why these specific numbers?

- **400–512 tokens for the child:** empirically the sweet spot for dense embeddings in
  finance/news content. Smaller → chunks become fragmentary (half-sentences embed poorly).
  Bigger → chunks start mixing multiple topics, and the embedding becomes a blurry average.
- **2,000 tokens for the parent:** roughly the maximum amount of context that's actually
  useful for the LLM to read to answer one passage-level question. Bigger wastes context
  window on irrelevant material; smaller starts losing the surrounding context.
- **Overlap (~80 tok between children, ~200 tok between parents):** chunks don't snap to
  clean boundaries. If a key sentence happens to land exactly on a boundary, you'd cut it in
  half. Overlap means each chunk re-includes the last bit of the previous one, so no thought
  gets split.

### Concrete example — same doc, two strategies

A 4,500-word sell-side note titled "EM Strategy Weekly," with sections: Intro, Brazil rates,
Mexico growth, LatAm FX, Asia PBOC, Conclusion (~4,200 tokens total).

**Current approach (1 vector per doc):**

```
[entire 4,200-token document]  →  ONE vector
```

The Brazil section gets averaged with Mexico, FX, Asia, and the conclusion. The single
vector is the "centroid" of all those topics. Query "Brazil rates" → mediocre match because
the vector represents *the whole note*, not the Brazil section.

**Hierarchical (parent 2,000 / child 400-512):**

```
Parent A: [Intro + Brazil + half of Mexico]              (2,000 tok)
  ├ Child 1: "Intro"                                     (400 tok) → vector
  ├ Child 2: "Brazil rates — first half"                 (400 tok) → vector
  ├ Child 3: "Brazil rates — second half + Mexico start" (400 tok) → vector
  ├ Child 4: "Mexico growth — first half"                (400 tok) → vector
  └ Child 5: "Mexico growth — second half"               (400 tok) → vector

Parent B: [rest of Mexico + LatAm FX + half of Asia]     (2,000 tok)
  ├ Child 6, 7, 8, 9, 10 → vectors

Parent C: [rest of Asia + Conclusion]                    (~1,200 tok)
  └ Child 11, 12, 13 → vectors
```

Query *"Brazil rates"*:

- Compares against 13 child vectors
- Best matches: **Child 2** and **Child 3** (both have Brazil rates content) — high
  similarity scores
- Returns **Parent A** (which contains those two children)
- LLM reads Parent A → has the full Brazil rates section + the intro context

---

## Part 3: How this stacks with the other two ingestion-time techniques

The proposed architecture combines **three separate techniques** at three different stages.
They get confused because they all involve a Haiku call somewhere; they are in fact distinct
operations.

| | What it does | When it runs | Granularity |
|---|---|---|---|
| **(1) Haiku cleaning** | Strips boilerplate, standardizes format | At ingest, **before** chunking | Once **per document** |
| **(2) Hierarchical chunking** | Splits long docs into parent (2,000) + child (400–512) pieces | At ingest, **after** cleaning | N parents + M children **per document** |
| **(3) Contextual retrieval prefix** | Generates a 50–100 tok "where does this chunk fit" prefix | At ingest, **after** chunking, before embedding | Once **per chunk** |

What happens to a single 5,000-tok bank note under the full stack:

```
Raw PDF text
  │
  ▼
(1) Haiku CLEAN  ────────────────  ~$0.001 per doc
  │
  ▼
(2) Hierarchical SPLIT
    → Parent A, B, C (2,000 tok each)
    → Child 1..15 (400-512 tok each)
  │
  ▼
(3) For EACH of 15 children:
    Haiku generates a 50-100 tok
    "this chunk is from {note} on {date},
     section {X}, discussing {Y}" prefix    ─── ~$0.001 per CHUNK
  │
  ▼
prefix + child text  →  Embed  →  vector
```

The cost asymmetry is what makes step (3) the expensive one: ~500 docs/day × **(1)** ≈ 500
calls, but ~500 docs/day × ~5–15 chunks each × **(3)** ≈ 2,500–7,500 calls. Step (3) costs
roughly **20× what step (1) costs** at typical volumes.

### What contextual retrieval actually adds

Step (3) solves a different problem from step (2). Hierarchical chunking solves "long docs
need to be searched at small granularity and returned at large granularity." It does **not**
solve: **chunks lose context when separated from their document.**

Example chunk taken from the middle of a sell-side note:

> *"The committee shifted to a more dovish stance this quarter, with three of seven members
> now favoring a 25 bp cut at the next meeting. Inflation prints have undershot the central
> projection for two consecutive months."*

If you embed this chunk and someone searches *"Brazil central bank dovish shift,"* the chunk
may not surface — it doesn't say "Brazil," doesn't say "central bank," it says "the
committee." The reader of the whole document knows which committee. The embedding doesn't.

Contextual retrieval fixes this by calling Haiku at ingest time with this prompt:

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

Haiku returns something like:

> *"This chunk is from a sell-side weekly note dated 2026-06-12 discussing the Banco Central
> do Brasil COPOM meeting on September 18, in the section reviewing the recent dovish shift
> among committee members."*

That prefix gets prepended to the chunk before embedding. Now the embedding contains "Banco
Central do Brasil," "COPOM," and "dovish shift" — the chunk surfaces.

### Where this lands

The recommendation in [chunking_study.md](../chunking_study.md) is to keep (1) and (2)
broadly but apply (3) **selectively**:

- **Keep (1) everywhere.** Cheap, in production, improves both single-vector and multi-chunk
  approaches equally.
- **Keep (2) on long docs only** (PDFs, podcasts, long-form > 3,000 tok). Short docs gain
  nothing from chunking.
- **Apply (3) only on PDF text-children, maybe podcast chunks.** Those are the classes where
  chunks lack self-contained context and anaphora ("the committee," "this view") is densest.

Three reasons applying (3) *universally* is overkill:

1. LlamaIndex's reproduction showed the contextual-prefix lift drops from ~35% at 200-tok
   chunks to ~15% at 800-tok chunks. The proposed child size (400–512 tok) is in the middle
   of that decay; parents (2,000 tok) are well past it.
2. A strong reranker (Cohere Rerank 3.5) gets ~55% failure reduction on its own; the prefix
   adds another ~12 pp on top. Real, but expensive at universal scale.
3. Step (1) already removes a lot of the boilerplate noise that (3) is partly trying to
   solve.

---

## Putting it together

The matrix tells you **which strategy applies to which kind of source.** Hierarchical
parent/child is one entry in that matrix — the strategy used for long-form documents
(op-eds, long emails, the text part of bank PDFs). Other entries use different strategies
(1-vector for tweets, page-image vectors for PDF charts, time-aligned chunks for podcasts),
because those source types have different shapes that the parent/child pattern wouldn't fit
well.

Contextual retrieval (step 3 above) is an *optional refinement* layered on top of chunking
— it doesn't replace anything, it just adds a per-chunk prefix to compensate for context
lost in the split. Whether it's worth applying depends on chunk size, document class, and
whether you're already running per-document cleaning and a strong reranker. The right way to
settle "should we use (3) here?" is to measure on the Phase 0 golden set, not to assume.
