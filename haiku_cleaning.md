# Pre-Embedding Document Cleaning with a Small LLM (Haiku Pass)

> **Companion to [README.md](README.md) and [chunking_study.md](chunking_study.md).** Both of
> those files focus on *how* documents are split and indexed. This file documents a distinct,
> earlier-in-the-pipeline step: a once-per-document pass through a small fast LLM (Claude
> Haiku) at ingestion time, before embedding, whose job is to **strip source-specific
> boilerplate and standardize formatting** so that the embedding (whether one-per-doc or
> per-chunk) is computed on clean signal rather than navigation menus, legal disclaimers,
> tracking pixels, and copy-paste artifacts.
>
> The pattern is in production in the live system (not unbuilt spec) and reliably improves
> downstream retrieval quality at low cost. It is **complementary to, not a substitute for**,
> Anthropic Contextual Retrieval (which prefixes *each chunk* with context about its position
> in the whole document). The two operate at different granularities and can coexist.

---

## TL;DR

- **What it is**: one Haiku call per ingested document, between raw extraction and embedding,
  that returns a cleaned version of the document text.
- **Why it works**: production embedders (Voyage-3-large, Cohere v4) embed *whatever you give
  them*. Noise gets vectorized along with signal. Removing source-specific boilerplate
  (Substack subscribe footers, legal disclaimers, navigation menus, copyright notices) before
  embedding produces vectors that better represent the actual content — typically observed as
  improved retrieval precision on near-miss queries.
- **Why a small LLM rather than regex**: each source has its own boilerplate pattern, the
  patterns change quietly, and writing regex for every source becomes unmaintainable. Haiku
  generalizes across sources for ~$0.001 per document; one prompt covers a whole class.
- **Two distinct modes** are in production use, picked per source class:
  1. **Preserve-all cleaning** — "remove boilerplate, change nothing else." Used when the
     source is the analyst's own voice (Substack columns, sell-side email research, political
     columns). The Haiku output is the *full original content* minus the noise.
  2. **Extractive cleaning** — "pull out the key economic/political insights, preserve named
     speakers and their specific calls." Used when the source bundles many topics in one
     newsletter and only a fraction is relevant downstream.
- **Cost is negligible**: ~500 docs/day × ~$0.001 per doc ≈ $15/month at current volumes.
- **The cleaning pass is orthogonal to chunking strategy** — it improves both single-vector and
  multi-chunk approaches by the same fraction, because both consume the cleaned text.

---

## 1 · Where this sits in the pipeline

```
   Raw source (HTML / email body / RSS item / PDF text)
                       │
                       ▼
       ┌────────────────────────────────┐
       │  Source-specific extraction    │   per-source scraper
       │  (BeautifulSoup, IMAP, etc.)   │   strips obvious HTML
       └───────────────┬────────────────┘
                       │
                       ▼
       ┌────────────────────────────────┐
       │  Regex pre-cleanup             │   removes well-known
       │  (collapse whitespace,         │   patterns the LLM
       │  drop common Substack lines)   │   shouldn't waste tokens on
       └───────────────┬────────────────┘
                       │
                       ▼
       ┌────────────────────────────────┐
       │  *** Haiku cleaning pass ***   │   ← THIS FILE
       │  Mode A: preserve-all          │
       │  Mode B: extract insights      │
       └───────────────┬────────────────┘
                       │
                       ▼
       ┌────────────────────────────────┐
       │  Post-cleanup regex            │   catches stubborn
       │  (phone numbers, work emails,  │   patterns the LLM
       │  copyright notices)            │   sometimes leaves in
       └───────────────┬────────────────┘
                       │
                       ▼
       ┌────────────────────────────────┐
       │  Embed → vector DB             │   Voyage-3-large 2048d
       │  (one vector or N chunks)      │   or Cohere v4 1536d
       └────────────────────────────────┘
```

The Haiku pass produces the `cleaned_text` field. Embeddings are always computed on
`cleaned_text`, never on the raw extract. This is the contract every per-source scraper
respects.

---

## 2 · Mode A — preserve-all cleaning

**Used for:** sources where the document *is* the analyst's voice, end-to-end. Substack
columns, sell-side research email bodies, political/legal columns from newspapers, op-eds.
The downstream system needs the full content because users ask both "what does this person
think about X" (gestalt) and "what did this person say about Y on date Z" (passage retrieval).

**Prompt template (English):**

```
You are a document cleaner. Output the FULL TEXT from this {source_label}
{document_type}, cleaned of noise and formatted beautifully. DO NOT SUMMARIZE —
keep ALL the actual content.

TITLE: {title}

RAW CONTENT:
{raw_text}

REMOVE: analyst contact emails, phone numbers, "View Report Online" links,
legal disclaimers, disclosures, footer/signature blocks, copyright notices,
"unsubscribe" text.

KEEP: ALL paragraphs of analysis, ALL data points, ALL forecasts, ALL quotes,
ALL tables (format with | columns), ALL bullet points, section headers.

FORMAT: use ## for sections, **bold** for key numbers, proper paragraph breaks,
- for bullet lists.

OUTPUT THE FULL CLEANED ARTICLE TEXT:
```

**Prompt template (Portuguese variant, for Brazilian columnists):**

```
Você é um formatador de texto. Limpe e formate o texto abaixo desta {source_label}
sobre {topic_area}.

REGRAS:
- NÃO resuma. Mantenha TODO o conteúdo original na íntegra.
- Mantenha o texto em português.
- Remova lixo de formatação (links quebrados, boilerplate de assinatura, rodapés).
- Corrija quebras de linha estranhas e espaçamento.
- Mantenha parágrafos, citações e estrutura do texto original.
- NÃO adicione comentários seus.

Título: {title}

Texto:
{raw_text}
```

**Key design choices:**

- *"DO NOT SUMMARIZE"* / *"NÃO resuma"* is repeated explicitly. Without this, Haiku will
  silently compress 5000-word columns to 800-word summaries, destroying every passage-level
  query downstream. This is the single most important instruction in the preserve-all prompt.
- *"Mantenha em PORTUGUÊS"* / language-preservation directives prevent silent translation to
  English on PT-source content.
- *REMOVE lists are source-specific*. A sell-side email needs different stripping than a
  Substack column — the prompt encodes domain knowledge about what counts as boilerplate.
- *Output format is markdown* — `##` headers, `|` tables, `-` lists — so downstream chunkers
  can use markdown headers as semantic boundaries when chunking is applied.

---

## 3 · Mode B — extractive cleaning

**Used for:** sources where one document bundles many topics, only some of which matter to
downstream users. Daily research newsletters, weekly "5-things" digests, multi-asset roundups
covering 10 markets when readers only ever ask about 3 of them.

**Prompt template:**

```
Extract the key {domain} insights from this {source_label} newsletter.
Focus on: {topic_list — e.g. rate decisions, FX views, commodity outlook,
named economist views with their specific calls}.

Keep the named speakers and their specific views (e.g. "{example named speaker
+ specific call}"). Remove podcast/subscription boilerplate. Output clean,
dense text.

Title: {title}

Content:
{raw_text}
```

**Key design choices:**

- *"Keep the named speakers and their specific views"* is the lever that preserves
  attribution. Without this clause, Haiku homogenizes ("analysts think rates will fall")
  instead of preserving the named-source detail ("Speaker X says rates fall to Y% by Q3").
  Attribution queries downstream depend on this clause.
- *Topic list is explicit and source-specific*. Each extractive prompt names the 4–6 topics
  the source covers that the downstream consumers actually care about. Topics not in the list
  get dropped — that's the point.
- *Lower `max_tokens` than preserve-all* (~2000 vs ~8000) because the output is genuinely
  shorter.

The trade-off is explicit: extractive mode **loses information** that fell outside the topic
list. The cost is recoverable (re-process from raw if topic priorities change) but not free.
Use preserve-all unless extraction is clearly worth the loss.

---

## 4 · Picking the mode per source class

| Source class | Mode | Why |
|---|---|---|
| Substack columns (single-author opinion) | Preserve-all | The column IS the signal. No part is throwaway. |
| Sell-side research emails (single bank's note) | Preserve-all | Each paragraph is a separately-cited claim. Summarizing destroys attribution. |
| Newspaper political/legal columns | Preserve-all | Quote fidelity matters; passage queries common. |
| Op-eds / long-form journalism | Preserve-all | Same as above. |
| Daily "X-in-X" research digests | Extractive | One newsletter bundles 5 unrelated topics; user cares about 2. |
| Multi-bank weekly research bundles | Extractive | Long, repetitive, low signal-to-noise per topic. |
| Generic news wires (Wire 1–7) | **No Haiku pass** | Already clean by source convention. Regex cleanup is sufficient. |
| Tweets | **No Haiku pass** | Already short. The cost/benefit doesn't work. |
| Calendar / market data / structured | **N/A** | Not text. No cleaning needed. |
| PDFs (sell-side research, ~5K–25K tokens) | **Different pattern — see §6** | Needs vision-model metadata extraction, not text cleaning. |

The rule of thumb: **Haiku-clean any source where the raw text contains >10% boilerplate**.
Below that, regex is cheaper and the LLM call doesn't pay.

---

## 5 · Concrete code skeleton (anonymized)

A self-contained reference implementation that captures the production pattern:

```python
import re
import requests

HAIKU_MODEL = "claude-haiku-4-5-20251001"
HAIKU_ENDPOINT = "https://api.anthropic.com/v1/messages"
HAIKU_API_VERSION = "2023-06-01"
HAIKU_MAX_INPUT_CHARS = 20000   # safety cap; per-source tuned in production
HAIKU_TIMEOUT_SEC = 60

PRESERVE_ALL_PROMPT = """You are a document cleaner. Output the FULL TEXT from
this {source_label} {document_type}, cleaned of noise and formatted beautifully.
DO NOT SUMMARIZE — keep ALL the actual content.

TITLE: {title}

RAW CONTENT:
{raw_text}

REMOVE: contact emails, phone numbers, "View Report Online" links, legal
disclaimers, disclosures, footer/signature blocks, copyright notices,
"unsubscribe" text.

KEEP: ALL paragraphs of analysis, ALL data points, ALL forecasts, ALL quotes,
ALL tables (format with | columns), ALL bullet points, section headers.

FORMAT: use ## for sections, **bold** for key numbers, proper paragraph breaks,
- for bullet lists.

OUTPUT THE FULL CLEANED ARTICLE TEXT:"""


def clean_with_haiku(title, raw_text, api_key, *,
                     source_label="research note",
                     document_type="email",
                     prompt_template=PRESERVE_ALL_PROMPT,
                     max_tokens=8192):
    """One Haiku call per document. Returns cleaned text, or raw on failure."""
    raw_text = raw_text[:HAIKU_MAX_INPUT_CHARS]
    prompt = prompt_template.format(
        source_label=source_label,
        document_type=document_type,
        title=title,
        raw_text=raw_text,
    )
    try:
        resp = requests.post(
            HAIKU_ENDPOINT,
            headers={
                "x-api-key": api_key,
                "anthropic-version": HAIKU_API_VERSION,
                "content-type": "application/json",
            },
            json={
                "model": HAIKU_MODEL,
                "max_tokens": max_tokens,
                "messages": [{"role": "user", "content": prompt}],
            },
            timeout=HAIKU_TIMEOUT_SEC,
        )
        if resp.status_code != 200:
            # Graceful degradation: failed Haiku call should not block ingest.
            return raw_text
        cleaned = resp.json()["content"][0]["text"].strip()

        # Post-cleanup regex: things Haiku occasionally leaves in.
        for pattern in [
            r"\+\d{1,3}[- ]?\d{3}[- ]?\d{3}[- ]?\d{4}",      # phone numbers
            r"[a-z]+\.[a-z]+@[a-z]+\.com",                    # work emails
            r"©\s*\d{4}.*?(?=\n|$)",                          # copyright lines
        ]:
            cleaned = re.sub(pattern, "", cleaned, flags=re.IGNORECASE)
        cleaned = re.sub(r"\n{3,}", "\n\n", cleaned)
        return cleaned
    except Exception:
        return raw_text
```

Notes on production behavior captured by this skeleton:

- **Direct REST call**, not the SDK. The SDK adds a dependency for one endpoint; the REST
  shape is stable and trivial to maintain.
- **`raw_text` cap before sending** prevents token overruns on rogue inputs (occasional
  40K-char emails that are mostly forwarded thread history).
- **Failed Haiku call returns raw text, not None.** Ingestion must not block on a single
  failed Haiku call — partial cleanup is better than dropping the document. The downstream
  embedder embeds whatever it receives.
- **Post-cleanup regex catches the long tail.** Haiku reliably removes ~95% of boilerplate;
  the remaining 5% (phone numbers, internal email addresses, copyright lines) gets a regex
  pass that's cheap and stable.
- **`\n{3,}` → `\n\n`** collapses excessive blank lines that Haiku introduces around removed
  blocks.

---

## 6 · The PDF special case — vision-model metadata extraction

PDFs (5K–25K-token sell-side research notes) follow a different pattern because the cleaning
problem is different: the text content extracted from the PDF is usually already clean (the
document was designed to be read), but the **metadata** (issuing institution, author, date,
report title) lives in the visual layout — letterhead, masthead, footer date stamps — which
text extraction destroys or scrambles.

For this class, a separate vision-model pass extracts metadata in a structured format:

```
You are extracting metadata from a sell-side research PDF. Output ONLY the
following fields in markdown header format. Use the first page's visual
information (letterhead, masthead, byline, date stamps) when text extraction
is ambiguous.

## Company/Institution
**{the issuing institution exactly as branded}**

## Author(s)
**{primary author(s), comma-separated}**

## Title
**{full report title}**

## Publication Date
**{YYYY-MM-DD if determinable, else "unknown"}**

## Document Type
**{daily note | weekly | thematic | initiation | flash | other}**
```

The parser on the receiving side handles both `Key: Value` (inline) and the markdown-header
format above. Both formats are common LLM outputs and a tolerant parser avoids losing
metadata when the model picks one over the other.

This pattern fixed a previously-documented failure where 45% of incoming PDFs were classified
as generic "Research" instead of their actual issuing institution — the original parser
only matched the inline `Key: Value` form and silently dropped values that arrived in
markdown-header form.

---

## 7 · Relationship to Anthropic Contextual Retrieval (and to chunking)

The Haiku cleaning pass is sometimes confused with Anthropic's contextual-retrieval prefix
pattern (the universal Haiku-prefix that the v5 spec proposes). They are different operations
operating at different granularities:

| | **Haiku cleaning pass (this file)** | **Anthropic Contextual Retrieval prefix** |
|---|---|---|
| Granularity | One call per **document** | One call per **chunk** |
| Purpose | Remove boilerplate, standardize formatting | Add cross-chunk context to chunks that lack it |
| What goes in | Raw extracted document | A chunk + the surrounding document |
| What comes out | Cleaned version of the document | A short prefix to prepend to the chunk before embedding |
| Cost scaling | Per ingested document (~500/day) | Per chunk after splitting (~2500/day at v5 spec sizes) |
| Order in pipeline | Before chunking, before embedding | After chunking, before embedding |
| Operates on | The text itself (rewrites it) | Generates new text (a prefix) |
| Compatible with single-vector indexing? | **Yes — fully compatible** | **N/A — only meaningful when chunking** |
| Compatible with multi-chunk indexing? | **Yes — fully compatible** | Yes |

**They are complementary, not alternatives.** A document can be Haiku-cleaned at ingestion
(removing boilerplate), then chunked, then optionally have each chunk wrapped in a contextual
prefix before embedding. The cleaning pass strictly improves both single-vector and
multi-chunk approaches because both compute embeddings on the cleaned text.

The relevant connection to the broader chunking discussion: **if a system already runs a
Haiku cleaning pass per document, the marginal value of also running Anthropic Contextual
Retrieval per chunk goes down**. Many of the things contextual retrieval is solving for —
ambiguous references, missing context, boilerplate noise leaking into chunks — are partially
or fully solved by upstream cleaning. The two should be evaluated jointly on the golden set,
not stacked by default.

---

## 8 · Cost and latency

At current ingestion volumes:

| Source class | Docs/day | Avg input chars | Cost/doc | Daily | Monthly |
|---|---|---|---|---|---|
| Sell-side research emails | ~30 | ~20K | ~$0.002 | $0.06 | ~$2 |
| Substack columns | ~25 | ~8K | ~$0.001 | $0.025 | ~$0.75 |
| Newspaper political columns | ~5 | ~8K | ~$0.001 | $0.005 | ~$0.15 |
| Analyst newsletters (extractive) | ~10 | ~6K | ~$0.0005 | $0.005 | ~$0.15 |
| PDFs (vision-model metadata) | ~7 | (vision) | ~$0.005 | $0.035 | ~$1 |
| **Total** | **~75** | | | **~$0.13** | **~$4** |

Latency per call: typically 2–8 seconds for preserve-all on a 20K-character input. This is
not on the user-query critical path — it runs at ingest time on a cron schedule.

For comparison, Anthropic Contextual Retrieval at the v5 spec's volumes is estimated at
~$75/month — roughly **20× the cost** of the per-document cleaning pass, because the
per-chunk granularity multiplies the call count by the chunks-per-doc ratio. This is the
cost asymmetry that motivates applying the contextual-prefix pass *selectively* rather than
universally.

---

## 9 · Failure modes observed in production

- **Silent summarization despite "DO NOT SUMMARIZE."** Haiku occasionally compresses anyway,
  especially on PT-language inputs. Mitigation: log `len(raw) → len(cleaned)` ratio on every
  call; alert when compression exceeds 60%. Reprocess flagged docs with a stronger
  "preservation" directive or fall back to raw.
- **Silent translation to English on PT inputs.** Catch with a quick character-set ratio
  check; if cleaned output has <10% non-ASCII characters when input had >5%, treat as a
  translation event and fall back to raw.
- **Truncation at `max_tokens`.** Long preserve-all outputs can hit the cap mid-paragraph.
  Mitigation: set `max_tokens` ≥ ceil(input_chars / 3.5) to leave room for the output to be
  as long as the input.
- **JSON-mode-style escaping leaks** ("`\\n`" appearing as literal text). Mitigation: the
  post-cleanup regex normalizes these.
- **Boilerplate the prompt didn't anticipate.** A new ad placement, a new disclaimer block.
  Mitigation: spot-check newly-ingested cleaned text weekly; add to the prompt REMOVE list
  when new patterns appear.
- **Haiku API rate limit / outage.** Mitigation: fall back to raw text on any non-200
  response. Ingestion must never block on Haiku availability.

---

## 10 · When this pattern earns its cost

The cleaning pass is worth running when:

- The raw extracted text contains >10% boilerplate by character count.
- The source has *structured* boilerplate (predictable templates, disclaimers, signatures)
  that an LLM can recognize across slight variants better than a regex.
- The downstream system uses embeddings (so noise gets vectorized and dilutes the signal).
- Document volume is low enough that ~$0.001/doc is trivial (~hundreds to low thousands per
  day; not millions).

It is **not** worth running when:

- The source is already clean by convention (most wire articles, structured data feeds).
- Documents are short (<300 tokens — the boilerplate-to-content ratio doesn't justify the
  call, and cleaning short text occasionally degrades it).
- The downstream system uses keyword search only (boilerplate hurts BM25 less than dense
  retrieval because it gets discounted by IDF anyway).

---

## 11 · Caveats

- All cost numbers are order-of-magnitude. Anthropic pricing changes quarterly; the *shape*
  of the cost trade-off (per-doc cleaning ≪ per-chunk prefixing) is stable.
- The "DO NOT SUMMARIZE" violation rate cited above is qualitative — measured by spot-check,
  not by a formal eval. Building a per-source compression-ratio dashboard is on the backlog.
- The vision-model PDF metadata pattern is shown in abstract because the prompt details have
  evolved iteratively; the structural pattern (markdown-header output, tolerant parser) is
  stable across iterations.
- This document is publication-safe by construction. Source names are anonymized; vendor
  names mentioned (Anthropic, Voyage, Cohere) are public.

---

## 12 · References

- [Anthropic Messages API](https://docs.anthropic.com/en/api/messages) — REST endpoint used
  by every Haiku-cleaning scraper in production.
- Anthropic, *Contextual Retrieval*, blog post (Sept 2024) —
  https://www.anthropic.com/news/contextual-retrieval (the *per-chunk* technique discussed in
  §7; distinct from the per-document cleaning pass documented in this file).
- [README.md](README.md) — high-level current-vs-proposed architecture comparison.
- [chunking_study.md](chunking_study.md) — single-vector vs multi-chunk indexing study;
  cross-references this file in §10 ("Where this study suggests recalibrating") and §11
  ("What to actually build").

---

*Companion to [README.md](README.md) and [chunking_study.md](chunking_study.md). All three
files are publication-safe by construction.*
