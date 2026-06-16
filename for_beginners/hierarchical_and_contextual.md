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

## Part 2.5: A real worked example — end-to-end

Let's actually run the EM Strategy Weekly through the pipeline, with concrete (mock) content
at every stage so you can see exactly what each step does.

### Step 0 — what arrives from the scraper

The raw email body, as the IMAP fetcher delivers it. Lots of noise:

```
View report online: https://research.bank-n.example/em-weekly/2026-06-12-v4

From: research-notifications@bank-n.example
To: subscriber@example.com
Subject: EM Strategy Weekly — Jun 12, 2026
Date: Thu, 12 Jun 2026 06:14:22 +0000

==============================================================
        EM STRATEGY WEEKLY  |  Jun 12, 2026  |  Issue #423
==============================================================

[Click here to view in browser]    [Forward to a colleague]
[Manage subscriptions]              [Unsubscribe]

----- INTRODUCTION -----

Welcome to this week's edition. Below we cover Brazil rates,
Mexico growth, LatAm FX positioning, Asia PBOC moves, and our
trade-of-the-week. Conditions across the EM complex shifted
materially over the past five sessions...

[... ~700 more words across Brazil, Mexico, FX, Asia sections ...]

----- CONCLUSIONS -----

In sum, we see a window for selective EM exposure over the next
2-3 weeks, with Brazil duration and MXN longs as the preferred
expressions.

==============================================================

This communication is being distributed for informational
purposes only. It does not constitute an offer to sell or a
solicitation of an offer to buy any security. Past performance
is not a guarantee of future results. Please see important
disclosures at the end of this document.

Bank N Research  |  +1 212 555 0100
research@bank-n.example  |  www.bank-n.example/research

© 2026 Bank N & Co. All rights reserved. Bank N is a registered
trademark of Bank N Holdings. This message and any attachments
are confidential and intended solely for the use of the
individual or entity to whom they are addressed...

[300+ more characters of legal disclaimer]
```

Total: ~5,800 tokens. Of those, ~1,600 tokens (28%) are boilerplate — headers, footers,
navigation links, disclaimers.

### Step 1 — after Haiku cleaning pass

One Haiku call with the preserve-all prompt from
[haiku_cleaning.md](../haiku_cleaning.md). The output:

```markdown
# EM Strategy Weekly — Jun 12, 2026

## Introduction

Welcome to this week's edition. Below we cover Brazil rates, Mexico growth,
LatAm FX positioning, Asia PBOC moves, and our trade-of-the-week. Conditions
across the EM complex shifted materially over the past five sessions...

## Brazil: rates view

The central bank's communication this week marked the clearest dovish pivot
of the cycle. Three of seven committee members now publicly favor a 25 bp
cut at the September meeting, up from one member at the May print. Headline
inflation has undershot the central projection for two consecutive months,
and the breakevens curve has flattened by 22 bp since May 28...

## Mexico: growth outlook

[~600 words on Mexico growth dynamics...]

## LatAm FX positioning

[~500 words on FX flow data, positioning surveys, technical levels...]

## Asia: PBOC moves

[~700 words on China rate corridor, liquidity injections, CNH fixings...]

## Conclusions

In sum, we see a window for selective EM exposure over the next 2-3 weeks,
with Brazil duration and MXN longs as the preferred expressions.
```

Total: ~4,200 tokens. The 1,600 tokens of boilerplate are gone. The markdown headers (`##`)
will become useful in the next step because the chunker can use them as semantic boundaries.

### Step 2 — hierarchical split

The chunker reads the cleaned markdown. It does two passes:

**Parent pass (~2,000 tokens, ~200 tok overlap)** — walks the document by character offset,
preferring to break on markdown headers (`##`) when one falls near the target size:

```
Parent A (tokens 0–2000):
  "# EM Strategy Weekly — Jun 12, 2026
   ## Introduction        [300 words]
   ## Brazil: rates view  [800 words]
   ## Mexico: growth outlook (first half) [~400 of 600 words]"

Parent B (tokens 1800–3800):           ← starts 200 tokens before Parent A ends (overlap)
  "(...Mexico growth tail...)
   ## LatAm FX positioning [500 words]
   ## Asia: PBOC moves (first half) [~400 of 700 words]"

Parent C (tokens 3600–4200):           ← starts 200 tokens before Parent B ends
  "(...Asia PBOC tail...)
   ## Conclusions [200 words]"
```

**Child pass (~400–512 tokens, ~80 tok overlap)** — walks each parent, breaking at sentence
boundaries when a candidate boundary falls near the target size. For Parent A:

```
Parent A
├── Child 1 (tokens 0–410):
│   "# EM Strategy Weekly — Jun 12, 2026
│    ## Introduction
│    Welcome to this week's edition. Below we cover Brazil rates,
│    Mexico growth, LatAm FX positioning, Asia PBOC moves, and our
│    trade-of-the-week. Conditions across the EM complex shifted
│    materially over the past five sessions...
│    [continues to fill the chunk]"
│
├── Child 2 (tokens 330–760):           ← 80 tok overlap with Child 1
│   "## Brazil: rates view
│    The central bank's communication this week marked the clearest
│    dovish pivot of the cycle. Three of seven committee members now
│    publicly favor a 25 bp cut at the September meeting, up from one
│    member at the May print. Headline inflation has undershot the
│    central projection for two consecutive months..."
│
├── Child 3 (tokens 680–1090):
│   "...and the breakevens curve has flattened by 22 bp since May 28.
│    Our view: we now see one cut at the September meeting as the
│    base case (60% probability), with a second cut by year-end (40%).
│    Risks are tilted to the dovish side given the inflation print..."
│
├── Child 4 (tokens 1010–1450):
│   "## Mexico: growth outlook
│    Q1 GDP came in at 1.8% q/q annualized, above the 1.3% consensus
│    and our 1.5% estimate. The composition was constructive: private
│    investment +4.2%, household consumption +2.1%..."
│
└── Child 5 (tokens 1370–1820):
    "...Banxico's stance has not yet adjusted to reflect the upside
     surprise. We expect the September minutes to soften the hawkish
     bias modestly, but no near-term cut signal..."
```

…and similarly Children 6–10 for Parent B, Children 11–13 for Parent C.

What's in the database after this step: **13 child rows + 3 parent rows**, all unembedded
so far. Each child row carries: `chunk_id`, `parent_id`, `doc_id`, `text`, `chunk_role =
"child"`, `chunk_idx`, `section_heading`, `created_at`.

### Step 3 — contextual prefix per child (the per-chunk Haiku call)

For each of the 13 children, the system calls Haiku with the contextual-retrieval prompt
from Part 3 below, passing in the *whole cleaned document* + the chunk. Take Child 3 — the
one that contains "we now see one cut at the September meeting as the base case":

Haiku returns:

```
This chunk is from the EM Strategy Weekly published 2026-06-12, in the
"Brazil: rates view" section, summarizing the analyst's base-case forecast
for the September COPOM meeting (one 25 bp cut at 60% probability, second
cut by year-end at 40%) and the dovish risk skew driven by the recent
inflation undershoot.
```

That prefix gets prepended to Child 3's text:

```
This chunk is from the EM Strategy Weekly published 2026-06-12, in the
"Brazil: rates view" section, summarizing the analyst's base-case forecast
for the September COPOM meeting (one 25 bp cut at 60% probability, second
cut by year-end at 40%) and the dovish risk skew driven by the recent
inflation undershoot.

...and the breakevens curve has flattened by 22 bp since May 28. Our view:
we now see one cut at the September meeting as the base case (60%
probability), with a second cut by year-end (40%). Risks are tilted to the
dovish side given the inflation print...
```

The **combined text** (prefix + chunk) is what gets embedded. Notice that the prefix has
injected "Brazil," "COPOM," "September," "base case," "dovish" — none of which were
explicitly in the chunk's words but all of which a search for "Brazil central bank base
case for September" would now match.

### Step 4 — embedding

Each child's (prefix + text) goes to Voyage / Cohere v4 and comes back as a 1,536- or
2,048-dimensional vector. The vector is stored back on the child row in the `vector` column,
and `has_vector` is set to `1.0`.

What ends up in the database for Child 3:

```
{
  "chunk_id":         "em_weekly_2026_06_12__c3",
  "parent_id":        "em_weekly_2026_06_12__pA",
  "doc_id":           "em_weekly_2026_06_12",
  "chunk_role":       "child",
  "chunk_idx":        3,
  "section_heading":  "Brazil: rates view",
  "text":             "...and the breakevens curve has flattened by 22 bp...",
  "contextual_prefix":"This chunk is from the EM Strategy Weekly published 2026-06-12...",
  "vector":           [0.0134, -0.0289, 0.1142, ... (2048 numbers)],
  "has_vector":       1.0,
  "source_id":        "bank_n_em_weekly",
  "created_at":       "2026-06-12T06:14:22Z",
  "entity_id":        "bank_n",
  "domain":           "macro"
}
```

The parent row for Parent A is *not* embedded (parents are storage for context, not search
targets):

```
{
  "chunk_id":        "em_weekly_2026_06_12__pA",
  "doc_id":          "em_weekly_2026_06_12",
  "chunk_role":      "parent",
  "chunk_idx":       0,
  "text":            "# EM Strategy Weekly — Jun 12, 2026\n## Introduction...",
  "vector":          null,
  "has_vector":      0.0,
  "created_at":      "2026-06-12T06:14:22Z"
}
```

### Step 5 — query time

A few days later, a user asks:

> *"What's the bank's base case for the September COPOM?"*

Walk through:

1. The query gets embedded by the same model that embedded the children → a 2,048-dim
   vector.
2. The vector database runs a cosine-similarity search against **all child vectors** in the
   table, returning the top 100 or so by similarity.
3. The reranker (Cohere Rerank 3.5) re-scores those 100 candidates against the *query text*
   and surfaces the top 30.
4. Child 3 is the top result. Its `parent_id` is `em_weekly_2026_06_12__pA`.
5. The system fetches Parent A's text and hands *that* (the full 2,000-token parent, not
   the 400-token child) to the synthesizer LLM.
6. The synthesizer LLM reads Parent A — which contains the Introduction, the full Brazil
   rates section, and the first half of Mexico — and answers:

   > *"Bank N's base case is one 25 bp cut at the September COPOM meeting (60% probability),
   > with a second cut by year-end (40%). The case rests on a recent two-month inflation
   > undershoot of the central projection and a 22 bp flattening of the breakevens curve
   > since May 28."*

Notice three things about that answer:

- The **specific numbers** ("25 bp," "60%," "22 bp") came from Child 3's text — the small
  precise unit that won the search.
- The **framing** ("base case," "September COPOM") came from the parent context — Child 3
  alone didn't say "September COPOM," just "the September meeting."
- The **attribution to Bank N** came from the parent header — Child 3 didn't repeat the
  document title.

This is the whole point of the hierarchical pattern: the **child won the search** (high-
precision match on the embedded numbers and views), but the **parent supplied the framing**
that made the answer correct and citable.

---



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
