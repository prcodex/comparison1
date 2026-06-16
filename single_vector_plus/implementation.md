# Implementation Guide — Single-Vector + Compensations

> Concrete how-to for the architecture described in [README.md](README.md). Schema, Haiku
> prompts, code skeletons for the ingest and query pipelines, the reranker integration, the
> two opt-ins, and a migration path from the existing single-vector-only setup.

---

## 1 · Storage schema

LanceDB table (one row per document). Existing fields stay; the four new fields are marked.

```
unified_v_doc
  ├─ doc_id                  STRING       PRIMARY KEY (e.g. "em_weekly_2026_06_12")
  ├─ source_id               STRING       e.g. "bank_n_em_weekly"
  ├─ domain                  STRING       macro | brazil | ai
  ├─ created_at              TIMESTAMP    UTC
  ├─ title                   STRING
  ├─ body_text               STRING       output of Haiku cleaning pass
  ├─ contextual_prefix       STRING   ◄── NEW: 100-200 tok Haiku-generated prefix
  ├─ tags_json               STRING   ◄── NEW: JSON {entities, topics, themes, geographies, data_points}
  ├─ entities                ARRAY<STRING> ◄── NEW: flat array of entity strings (indexed for filters)
  ├─ topic_tags              ARRAY<STRING> ◄── NEW: flat array of topic tags (indexed for BM25)
  ├─ vector                  FIXED_SIZE_LIST<FLOAT, 2048>  Voyage-3-large embedding of (prefix + body)
  ├─ has_vector              FLOAT        1.0 if vector populated, 0.0 otherwise
  ├─ priority_mult           FLOAT        editorial multiplier (existing scoring v3 field)
  └─ scope_mult              FLOAT        editorial multiplier (existing scoring v3 field)
```

**Two Tantivy BM25 indexes:**

1. **Body index** — `body_text` field, language-aware tokenizer (`en_stem` for macro/AI,
   `pt_stem` for brazil).
2. **Tags index** — `topic_tags` + `entities` arrays joined into one searchable field, with a
   document-frequency boost of **1.5–2×** at query time (lexical hits on tags carry more
   weight than hits on the body because tags are denser and curated).

For the late-chunking opt-in (§4), a sibling table `unified_v_doc_late_chunks` holds the
sub-document vectors keyed back to `doc_id`. For the ColPali opt-in (§5), a sibling table
`unified_v_pdf_pages` holds page-image vectors.

---

## 2 · Ingest pipeline

Three sequential steps. The third one is the new combined Haiku call.

### 2.1 Step 1 — Haiku CLEAN (existing)

Already documented in [../haiku_cleaning.md](../haiku_cleaning.md). One Haiku call per
document, returns the cleaned body text. Output goes into `body_text`. No change from the
current production pattern.

### 2.2 Step 2 — Haiku TAG EXTRACTION + DOC-LEVEL PREFIX (one call)

The merge insight from the design discussion: the same Haiku call returns two outputs.
One JSON block with structured tags, one prose paragraph that serves as the contextual
prefix.

**Prompt template:**

```
You are analyzing a {source_class} document. Produce TWO outputs:

OUTPUT 1 — Structured tags as a single JSON object with these exact keys:
{
  "entities":     [up to 8 named entities: institutions, people, places, instruments]
  "topics":       [up to 6 topic slugs from the controlled vocabulary]
  "themes":       [up to 4 free-text themes specific to this document]
  "geographies":  [up to 4 country/region codes]
  "data_points":  [up to 5 specific numbers/claims worth indexing, each ≤120 chars]
}

OUTPUT 2 — A 100-200 token prose prefix that describes what this document is
and what it contains, using the entities and topics from OUTPUT 1 in natural
sentences. Start with "This document is..." and end with the document's
publication date. This prefix will be prepended to the document text before
embedding, so write it for downstream search retrieval, not as a summary.

Controlled topic vocabulary: {topics_vocab_list}

Document title: {title}
Document date: {date}
Source class: {source_class}
Source identifier: {source_id}

Document body:
{cleaned_body_text}

Return both outputs in this exact format:

<tags>
{ ...json... }
</tags>

<prefix>
This document is...
</prefix>
```

**Why the controlled topic vocabulary matters.** Without it, Haiku invents free-text topic
labels that drift across documents (`"brazil_rates"` vs `"brazil rates"` vs
`"banco_central_brazil"`). With a controlled list, every document is tagged against the same
~80–120 slugs, which makes the tag-BM25 leg in §3 actually work (BM25 doesn't normalize
spelling). The vocabulary lives in a flat YAML file checked into the repo and is the
**single most important configuration artifact** in this whole architecture.

Example controlled vocabulary excerpt:

```yaml
# topics_vocab.yml
macro:
  - brazil_rates
  - brazil_fiscal
  - selic_path
  - copom
  - us_economy
  - us_fed
  - fomc
  - rate_cut_expectations
  - inflation
  - cpi
  - ppi
  - employment
  - jobs_report
  - fx_emerging_markets
  - oil_supply
  - oil_demand
  - hormuz_shipping
  - china_pboc
  - cnh_fixing
  # ... ~80 more
geopolitics:
  - iran_war
  - iran_nuclear
  - hormuz_strait
  - israel_lebanon
  - russia_ukraine
  - trump_admin
  - us_china_trade
  # ... ~40 more
brazil:
  - eleicoes_2026
  - stf_decisoes
  - banco_central
  - selic
  - copom
  - lula_governo
  # ... ~50 more (PT slugs)
```

**Why mix EN and PT slugs.** Because the same document can mention Brazilian topics in
English (a Bank N research note about Selic) and Brazilian queries can come in either
language. Tagging in the language of the *topic* rather than the *document* keeps queries
in either language matching consistently.

### 2.3 Step 2 — code skeleton

```python
import json, re, requests

HAIKU_MODEL = "claude-haiku-4-5-20251001"
HAIKU_ENDPOINT = "https://api.anthropic.com/v1/messages"

def haiku_tags_and_prefix(title, date, source_class, source_id,
                          cleaned_body, topics_vocab, api_key):
    """One Haiku call returns both structured tags and a prose prefix."""
    prompt = TAGS_AND_PREFIX_PROMPT.format(
        title=title, date=date, source_class=source_class,
        source_id=source_id, cleaned_body=cleaned_body[:20000],
        topics_vocab_list=", ".join(topics_vocab),
    )
    resp = requests.post(
        HAIKU_ENDPOINT,
        headers={
            "x-api-key": api_key,
            "anthropic-version": "2023-06-01",
            "content-type": "application/json",
        },
        json={
            "model": HAIKU_MODEL,
            "max_tokens": 2000,
            "messages": [{"role": "user", "content": prompt}],
        },
        timeout=60,
    )
    if resp.status_code != 200:
        # Graceful degradation: empty tags + empty prefix.
        # The body still gets embedded; we just lose this lift for this doc.
        return {}, ""
    text = resp.json()["content"][0]["text"]

    # Parse the <tags> ... </tags> block as JSON
    tags_match = re.search(r"<tags>(.*?)</tags>", text, re.DOTALL)
    tags = {}
    if tags_match:
        try:
            tags = json.loads(tags_match.group(1).strip())
        except json.JSONDecodeError:
            tags = {}

    # Parse the <prefix> ... </prefix> block as a string
    prefix_match = re.search(r"<prefix>(.*?)</prefix>", text, re.DOTALL)
    prefix = prefix_match.group(1).strip() if prefix_match else ""

    return tags, prefix
```

**Validate the tags against the vocabulary** before storing. Drop any topic that isn't in
the controlled list; keep entities and data_points as free text. This prevents vocabulary
drift over time.

### 2.4 Step 3 — embed

The text that gets embedded is `prefix + "\n\n" + body_text`:

```python
import voyageai

def embed_document(prefix, body, api_key):
    client = voyageai.Client(api_key=api_key)
    text = (prefix + "\n\n" + body)[:30000]  # safety cap
    result = client.embed(
        [text],
        model="voyage-3-large",
        input_type="document",
        output_dimension=2048,
    )
    return result.embeddings[0]
```

**Cap the combined text at the embedder's effective context.** Voyage-3-large is nominally
32K; effective context (per NoLiMa-style needle evals) is closer to ~16K. For documents
longer than that, this is the moment to trigger the late-chunking opt-in (§4) — the doc-
level vector still gets stored, but it's complemented by sub-doc vectors that aren't
subject to the long-context degradation.

### 2.5 Putting the ingest pipeline together

```python
def ingest_one_document(raw_text, title, date, source_class, source_id, api_keys):
    # Step 1 — clean
    cleaned = haiku_clean(title, raw_text, api_keys["anthropic"])

    # Step 2 — tags + prefix in one call
    tags, prefix = haiku_tags_and_prefix(
        title=title, date=date, source_class=source_class,
        source_id=source_id, cleaned_body=cleaned,
        topics_vocab=load_topics_vocab(), api_key=api_keys["anthropic"],
    )

    # Step 3 — embed (prefix + body)
    vector = embed_document(prefix, cleaned, api_keys["voyage"])

    # Flatten arrays out of the tags JSON for separate columns
    entities   = tags.get("entities", [])
    topic_tags = tags.get("topics", [])

    record = {
        "doc_id":            make_doc_id(source_id, date),
        "source_id":         source_id,
        "domain":            domain_for(source_class),
        "created_at":        date,
        "title":             title,
        "body_text":         cleaned,
        "contextual_prefix": prefix,
        "tags_json":         json.dumps(tags),
        "entities":          entities,
        "topic_tags":        topic_tags,
        "vector":            vector,
        "has_vector":        1.0 if vector else 0.0,
        "priority_mult":     priority_for(source_id),
        "scope_mult":        scope_for(source_id, domain_for(source_class)),
    }
    insert_into_lancedb(record)

    # Opt-in: late chunking for long docs (§4)
    if estimate_tokens(cleaned) > 4000:
        late_chunks = late_chunk(cleaned, prefix)
        for chunk_vec, chunk_text, chunk_idx in late_chunks:
            insert_late_chunk(record["doc_id"], chunk_idx, chunk_text, chunk_vec)

    # Opt-in: ColPali for PDFs (§5)
    if source_class == "pdf":
        for page_idx, page_image in render_pdf_pages(raw_pdf_bytes):
            page_vec = embed_image(page_image, api_keys["cohere"])
            insert_page_image(record["doc_id"], page_idx, page_vec)
```

---

## 3 · Query pipeline — the three-leg retrieval

The retrieval flow expands the current pipeline by adding a third BM25 leg on the tags
field and stacking a cross-encoder reranker before the editorial multipliers.

### 3.1 Three parallel legs

```python
def retrieve_candidates(query, domain, time_window_hours, k=100):
    query_vec = embed_query(query)

    # Leg A — dense vector search on doc vectors
    dense_hits = lancedb_table.search(query_vec) \
        .where(f"domain = '{domain}' AND has_vector = 1.0") \
        .where(time_window_clause(time_window_hours)) \
        .limit(k) \
        .to_pandas()

    # Leg B — BM25 on body text
    body_hits = bm25_body_index.search(query, domain=domain, k=k)

    # Leg C — BM25 on tags + entities, with field boost
    tag_hits  = bm25_tags_index.search(query, domain=domain, k=k, field_boost=1.7)

    return dense_hits, body_hits, tag_hits
```

The **`field_boost=1.7`** on the tags leg is the lever that says "a lexical match on a
curated topic tag is worth ~70% more than a match in the document body." Tune this on the
golden set (§ phase0_eval.md); typical values land in the 1.5–2× range.

### 3.2 Fuse with RRF

```python
def rrf_fuse(hit_lists, k_constant=60):
    """Standard reciprocal rank fusion across N hit lists."""
    scores = {}
    for hits in hit_lists:
        for rank, doc_id in enumerate(hits["doc_id"].tolist(), start=1):
            scores[doc_id] = scores.get(doc_id, 0.0) + 1.0 / (k_constant + rank)
    fused = sorted(scores.items(), key=lambda kv: -kv[1])[:150]
    return [doc_id for doc_id, _ in fused]
```

**Why vanilla RRF and not weighted RRF.** The agent-research finding from the discussion:
once a strong reranker sits behind the fusion, the score-distribution mismatch between BM25
and dense gets re-scored away. Tuning RRF weights stops paying. Spend the engineering time
on rerank quality and tag-extraction quality instead.

### 3.3 Cross-encoder rerank

```python
import cohere

def rerank(query, candidate_doc_ids, full_docs, top_n=40):
    """Re-score candidates by reading query + doc body together."""
    client = cohere.Client(api_key=COHERE_API_KEY)
    docs_for_rerank = [full_docs[doc_id]["body_text"][:8000] for doc_id in candidate_doc_ids]
    result = client.rerank(
        model="rerank-3.5",
        query=query,
        documents=docs_for_rerank,
        top_n=top_n,
    )
    return [(candidate_doc_ids[r.index], r.relevance_score) for r in result.results]
```

The reranker reads up to 8K tokens of document body per candidate (Cohere Rerank 3.5
effective context). For documents longer than 8K, send the prefix + the first 7K tokens of
body — the prefix tells the reranker what the document is about and the first 7K usually
contains the lede.

### 3.4 Editorial multipliers (the existing layer)

```python
def apply_editorial(reranked, query_entity_hits, top_k=30):
    """Apply priority, recency, and entity-floor multipliers."""
    scored = []
    for doc_id, rerank_score in reranked:
        doc = lookup_doc(doc_id)
        time_decay = exp(-(now() - doc["created_at"]).total_hours() / 168)  # 7-day tau
        priority = doc["priority_mult"]
        scope = doc["scope_mult"]
        score = rerank_score * priority * scope * time_decay
        scored.append((doc_id, score))

    # Entity floor: if query mentions an entity, ensure ≥N docs from that entity appear
    if query_entity_hits:
        scored = ensure_entity_floor(scored, query_entity_hits, floor_n=5)

    scored.sort(key=lambda x: -x[1])
    return scored[:top_k]
```

This layer is unchanged from the existing scoring v3 — it just operates on rerank scores
instead of raw semantic similarity scores. Same multipliers, same anchor mode.

### 3.5 Putting the query pipeline together

```python
def answer_query(query, domain, time_window_hours):
    dense_hits, body_hits, tag_hits = retrieve_candidates(query, domain, time_window_hours)

    fused_doc_ids = rrf_fuse([dense_hits, body_hits, tag_hits])
    full_docs = batch_lookup(fused_doc_ids)

    reranked = rerank(query, fused_doc_ids, full_docs)

    query_entity_hits = detect_entities_in_query(query)
    final_top = apply_editorial(reranked, query_entity_hits)

    # Optional: parent expansion for late-chunk and ColPali winners
    final_with_context = expand_to_parent(final_top)

    context = assemble_context(final_with_context, max_tokens=20000)
    return synthesizer_llm(query=query, context=context)
```

---

## 4 · Late chunking opt-in (for docs > 4K tokens)

A subset of documents (long-form journalism, long email research, op-eds over 4K tokens) is
long enough that the single doc-vector starts losing recall on intra-document span queries.
For these documents only, run late chunking as an *additional* indexing pass — the doc-
level vector stays in `unified_v_doc`; sub-document vectors live in `unified_v_doc_late_chunks`.

**The late-chunking technique** (Jina, arXiv:2409.04701): encode the *whole document* once
with a long-context embedder that exposes token-level outputs, then mean-pool tokens over
fixed-size windows. Each window's vector carries cross-window context for free because
every token saw the whole document in attention.

**Why this fits the architecture's premise.** No smart-boundary detection needed — naive
fixed-size windows work. This sidesteps the "rule-based chunking introduces noise" concern
that motivated single-vector in the first place.

```python
def late_chunk(body_text, prefix, window_tokens=600, stride_tokens=500):
    """
    Returns: list of (vector, chunk_text, chunk_idx).
    Each vector represents a window of the document, but was computed
    with the whole document in attention.
    """
    # Encode the whole (prefix + body) once, getting token-level outputs
    full_text = prefix + "\n\n" + body_text
    token_embeddings = embedder_with_token_outputs(full_text)
    # token_embeddings: shape [num_tokens, dim]

    chunks = []
    cursor = 0
    idx = 0
    while cursor < len(token_embeddings):
        end = min(cursor + window_tokens, len(token_embeddings))
        window_vec = token_embeddings[cursor:end].mean(axis=0)
        chunk_text = detokenize(token_ids[cursor:end])
        chunks.append((window_vec, chunk_text, idx))
        cursor += stride_tokens
        idx += 1
    return chunks
```

**At query time**, the late-chunk vectors join Leg A as an additional source of dense hits,
keyed back to `doc_id`. A late-chunk hit returns the parent `doc_id` for assembly (the
reranker still reads the full document body).

**Embedder availability.** Late chunking requires token-level outputs. Jina v3 supports it
natively; BGE-M3 in its multi-vector mode does too. Voyage-3-large and Cohere v4 currently
expose only pooled outputs via their hosted APIs; if you stay on those, late chunking
requires either self-hosting a compatible model or skipping this opt-in entirely. **This is
a real architectural decision worth a Phase 0 spike** — see [phase0_eval.md](phase0_eval.md).

---

## 5 · ColPali opt-in (for PDFs)

PDFs are the one class where text-only single-vector fundamentally fails on chart/table
queries. The fix is to render each page as an image and embed via Cohere v4 in image mode.

```python
from pdf2image import convert_from_bytes

def render_and_embed_pdf_pages(pdf_bytes, doc_id, api_key):
    """Render PDF pages → images → Cohere v4 (image mode) → page-image vectors."""
    images = convert_from_bytes(pdf_bytes, dpi=150)
    page_records = []
    for page_idx, image in enumerate(images, start=1):
        page_bytes = image_to_bytes(image, format="png", max_pixels=2_000_000)
        page_vec = cohere_embed_image(page_bytes, api_key)
        page_records.append({
            "doc_id":      doc_id,
            "page_index":  page_idx,
            "page_vector": page_vec,
        })
    return page_records
```

**At query time**, the page-image vectors form a fourth retrieval leg (or extend Leg A).
Hits return their parent `doc_id` for assembly. The reranker reads the full PDF body text;
the page-image hit is just the *recall* signal that the chart/table on page N is relevant.

This opt-in adds ~$3/month at typical PDF ingest rates (~50 PDFs/week × ~25 pages × Cohere
image-embed pricing). Trivial relative to the value of catching chart/table queries that
text-only retrieval misses entirely.

---

## 6 · Migration path from the existing single-vector-only system

The transition is **additive**, not replacing. Existing vectors keep working throughout.

| Phase | Change | Effort | Eval gate |
|---|---|---|---|
| **0** | Build the controlled `topics_vocab.yml` (~80–120 slugs across macro/geo/brazil) | 1 day | Manual review; not user-facing |
| **0** | Build the 200-query golden set with per-source-class breakdowns (see [phase0_eval.md](phase0_eval.md)) | 2–3 days | Baseline numbers documented |
| **1** | Add the Haiku TAGS + PREFIX call to ingest; backfill 30 days of existing docs | 2 days code + ~6 hr backfill | Spot-check tag quality on 100 random docs |
| **1** | Re-embed those 30 days of docs with (prefix + body) | ~2 hr (Voyage backfill) | Embedding cost ~$5 total |
| **2** | Stand up the second Tantivy index over `topic_tags + entities` | 1 day | Index build completes |
| **2** | Wire the 3-leg retrieval behind `RETRIEVAL_MODE=svp_v1` env var (current path stays default) | 2 days | Shadow eval starts |
| **3** | Wire Cohere Rerank 3.5 (or Voyage rerank-2.5) at stage 4 of the query pipeline | 1 day | Latency budget verified |
| **3** | Shadow eval for 2 weeks: log both retrievers' top-30 on every query, score with the existing self-evaluator | 2 weeks | Lift vs baseline measured per query class |
| **4** | Late-chunking spike: verify whether Voyage or Cohere expose token-level outputs; if not, decide on self-host vs skip | 2–3 days | Decision documented |
| **4** | If late chunking is feasible: implement, backfill for docs > 4K tok, add to retrieval leg A | 3–4 days | Long-doc query subset shows lift |
| **5** | ColPali / page-image: render existing PDFs, embed pages, add fourth retrieval leg | 4–5 days | Chart-query subset shows lift |
| **6** | Cut over default `RETRIEVAL_MODE` to `svp_v1` once the golden-set delta is ≥ +5% relevance@10 and there's no per-source-class regression | 1 day | Production cutover |

Total: ~3–4 weeks of focused engineering plus 2 weeks of shadow eval. No big-bang
migration; the existing single-vector path stays operational throughout.

---

## 7 · Cost analysis at production volumes

At current ingest rates (~500 docs/day) and query rates (~100 queries/day):

| Item | Volume | Unit cost | Monthly |
|---|---|---|---|
| Haiku CLEAN (existing) | 500 docs/day | $0.001 | $15 |
| Haiku TAGS + PREFIX (new, one call returning both) | 500 docs/day | $0.001 | $15 |
| Voyage embed of (prefix + body) | 500 docs/day × ~3K tok | $0.18/1M tok | $8 |
| Cohere Rerank 3.5 (new) | 100 queries/day × 150 candidates | $2/1K reranks | $9 |
| Late chunking embed (opt-in, ~15% of docs) | 75 docs/day × extra tok | $0.18/1M tok | $3 |
| ColPali page embed (opt-in, PDFs only) | ~50 PDFs/week × 25 pages | per-image | $3 |
| **Total ongoing** | | | **~$53/month** |
| One-time tag + prefix backfill | ~50K existing docs | $0.001 each | **~$50 one-time** |

Compare to the v5 spec's ~$95/month ongoing (driven primarily by the per-chunk Haiku
contextual prefix at ~$75/month). **The architecture in this directory is about ~45%
cheaper to operate** at current volumes, with the cost savings concentrated in the per-
chunk Haiku line that this design eliminates.

---

## 8 · What this implementation does NOT do

Called out so the boundaries of the design are explicit:

- **No per-chunk Haiku contextual prefix.** That's the v5 spec's most expensive line item;
  this architecture handles the same problem at doc level for ~5% of the cost.
- **No hierarchical parent/child chunking on the bulk of the corpus.** Reserved for the
  late-chunking opt-in (docs > 4K) and ColPali (PDFs), which are the only two classes where
  single-vector demonstrably fails.
- **No RAPTOR-style summary trees.** Considered and rejected — they age badly with corpus
  churn (every leaf change invalidates summary nodes), and the doc-level prefix in §2
  captures most of the benefit for attribution-shaped queries without the rebuild cost.
- **No span-level citation in the UX.** The synthesizer cites documents, not sentences. A
  separate engineering project if needed.
- **No multi-vector-per-doc ("doc + 2-3 topic summary vectors") hedge.** The late-chunking
  opt-in is a stronger, better-supported alternative for the same goal (multiple search
  angles per document).
