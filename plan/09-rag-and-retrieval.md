# 09 — RAG & Retrieval

> **Scope**: Hybrid search with Reciprocal Rank Fusion (RRF), the `page_index` query system, structured citations, cross-conversation RAG, and large TXT file handling.
>
> **Tasks**: RAG_INFRA (RAG Infrastructure), CROSS_CONV_RAG (Cross-Conversation RAG)

---

## Table of Contents

- [Architecture Overview](#architecture-overview)
- [The page_index System](#the-pageindex-system)
- [Hybrid Search with RRF](#hybrid-search-with-rrf)
- [Graceful Degradation](#graceful-degradation)
- [The Query Tool](#the-query-tool)
- [Page Context Assembly](#page-context-assembly)
- [Structured Citations](#structured-citations)
- [Cross-Conversation RAG](#cross-conversation-rag)
- [Large TXT RAG](#large-txt-rag)
- [Cross-References](#cross-references)
- [Task Specifications](#task-specifications)
- [External References](#external-references)

---

## Architecture Overview

RAG in safeagent is built around the `page_index` table, which stores per-page data for uploaded PDFs. The upload endpoint returns immediately with a `fileId` and initial status (`uploading`), then processing continues asynchronously through `summarizing` to `ready`. Each page has two representations over that lifecycle: a detailed summary (available after the async summarization stage) and raw extracted text (available after background enrichment). Hybrid search fuses results from up to three arms depending on which representations are available.

```mermaid
graph TB
    subgraph Upload["Document Upload & Processing"]
        direction TB
        BLOCK["Summarization Stage\n(async after upload returns)"]
        BG["Background Stage\n(async enrichment)"]
        BLOCK -->|"page summaries written"| PAGE_IDX_SUMM["page_index.summary\n(summaries + summary embeddings)"]
        BG -->|"raw text + embeddings written"| PAGE_IDX_RAW["page_index.raw_text\n(raw text + raw embeddings + tsvector)"]
    end
    subgraph Query["Query Time"]
        direction TB
        TOOL["searchDocument tool\n(server-side, access-controlled)\n(created by createDocumentQueryTool)"]
        EMBED["Embed query\n(EMBEDDING_PROVIDER)"]
        HYBRID["Hybrid Search\n(RRF fusion)"]
        CTX["getPageContextForPages\n(fetch S3 + summaries + chunks + page images)"]
        RESP["Evidence bundle:\n{ matchedPages: PageBundle[] }"]

        TOOL --> EMBED --> HYBRID --> CTX --> RESP
    end

    subgraph Arms["Search Arms"]
        ARM_SUMMARY_VEC["Arm 1: Vector on summaries\n(always available)"]
        ARM_RAW_VEC["Arm 2: Vector on raw text\n(enriched only)"]
        ARM_KEYWORD["Arm 3: Keyword on raw tsvector\n(enriched only)"]
    end

    PAGE_IDX_SUMM --> ARM_SUMMARY_VEC
    PAGE_IDX_RAW --> ARM_RAW_VEC
    PAGE_IDX_RAW --> ARM_KEYWORD
    ARM_SUMMARY_VEC & ARM_RAW_VEC & ARM_KEYWORD --> HYBRID
```

The two-stage processing model means retrieval quality improves automatically as background enrichment completes. A query that arrives seconds after upload gets summary-only results. The same query an hour later gets full three-arm fusion. The user never sees an error either way.

---

## The page_index System

### Table Structure

The `page_index` table is the central data structure for PDF retrieval. Each row represents one page of one uploaded file.

```mermaid
erDiagram
    page_index {
        uuid id PK
        uuid file_id FK
        integer page_number
        text summary
        vector summary_embedding
        text raw_text "nullable"
        vector raw_embedding "nullable"
        tsvector raw_tsvector "nullable GENERATED"
        jsonb metadata
        text thread_id
        text user_id
        timestamptz created_at
    }

    file_uploads {
        uuid file_id PK
        text user_id FK
        text file_name
        text s3_key
        timestamptz created_at
    }

    page_index }o--|| file_uploads : "file_id"
```

The `summary` column holds the LLM-generated page summary. It's populated during asynchronous summarization after upload acceptance. The `raw_text`, `raw_embedding`, and `raw_tsvector` columns are populated during later background enrichment. A row with null `raw_text` is in summary-only mode.

### Two Representations Per Page

```mermaid
flowchart LR
    subgraph Blocking["Summarization Stage (asynchronous)"]
        PDF_PAGE["PDF page image"]
        VISION_LLM["Vision LLM\n(Gemini Flash Lite)"]
        SUMMARY["Detailed page summary\n(structured prose)"]
        SUMM_EMBED["Summary embedding\n(EMBEDDING_PROVIDER)"]
        PDF_PAGE --> VISION_LLM --> SUMMARY --> SUMM_EMBED
    end

    subgraph Background["Background Stage (async)"]
        OCR["OCR / text extraction"]
        RAW_TEXT["Raw extracted text"]
        RAW_EMBED["Raw text embedding\n(EMBEDDING_PROVIDER)"]
        TSVEC["PostgreSQL tsvector\n(full-text index)"]
        OCR --> RAW_TEXT --> RAW_EMBED
        RAW_TEXT --> TSVEC
    end

    subgraph Storage["page_index row"]
        COL_CONTENT["summary\n(LLM-generated page summary)"]
        COL_SUMM_EMB["summary_embedding"]
        COL_RAW["raw_text"]
        COL_RAW_EMB["raw_embedding"]
        COL_TSVEC["raw_tsvector"]
    end

    SUMMARY --> COL_CONTENT
    SUMM_EMBED --> COL_SUMM_EMB
    RAW_TEXT --> COL_RAW
    RAW_EMBED --> COL_RAW_EMB
    TSVEC --> COL_TSVEC
```

Summaries are richer than raw text for retrieval purposes. A vision LLM reading a page can describe a chart, interpret a table, and capture the semantic meaning of a diagram. Raw text misses all of that. But raw text has its own advantages: exact keyword matching, verbatim quotes, and no risk of the summary LLM paraphrasing something incorrectly. The three-arm fusion gets both.

---

## Hybrid Search with RRF

### What is Reciprocal Rank Fusion

RRF is a rank-based fusion algorithm. It doesn't normalize scores across arms (which is unreliable when arms use different scoring functions). Instead, it uses each document's rank within its arm. A document ranked first in any arm gets a high RRF contribution. A document ranked 40th gets almost nothing.

The formula for a document's RRF score is the sum of `1 / (k + rank)` across all arms where the document appears, where `k` is a smoothing constant (set to 50 here). Documents that appear in multiple arms accumulate contributions from each, which is why multi-arm fusion consistently outperforms any single arm.

### Three-Arm Fusion

```mermaid
flowchart TD
    QUERY["User query\n+ queryEmbedding[3072]"]

    subgraph ARM_SUMMARY_VEC["Arm 1: Vector on Summaries\n(always available)"]
        A1_SEARCH["pgvector cosine similarity\non summary_embedding\nORDER BY similarity DESC\nLIMIT 40"]
        A1_RESULTS["40 candidates\n(file_id, page_number, rank 1..40)"]
        A1_SEARCH --> A1_RESULTS
    end

    subgraph ARM_RAW_VEC["Arm 2: Vector on Raw Text\n(enriched only)"]
        A2_CHECK{raw_embedding\nnot null?}
        A2_SEARCH["pgvector cosine similarity\non raw_embedding\nORDER BY similarity DESC\nLIMIT 40"]
        A2_RESULTS["40 candidates\n(file_id, page_number, rank 1..40)"]
        A2_ZERO["0 rows\n(graceful degradation)"]
        A2_CHECK -->|"yes"| A2_SEARCH --> A2_RESULTS
        A2_CHECK -->|"no"| A2_ZERO
    end

    subgraph ARM_KEYWORD["Arm 3: Keyword on Raw tsvector\n(enriched only)"]
        A3_CHECK{raw_tsvector\nnot null?}
        A3_SEARCH["PostgreSQL full-text search\nts_rank on raw_tsvector\nORDER BY rank DESC\nLIMIT 40"]
        A3_RESULTS["40 candidates\n(file_id, page_number, rank 1..40)"]
        A3_ZERO["0 rows\n(graceful degradation)"]
        A3_CHECK -->|"yes"| A3_SEARCH --> A3_RESULTS
        A3_CHECK -->|"no"| A3_ZERO
    end

    subgraph RRF["RRF Fusion (k=50)"]
        UNION["UNION ALL three arms\n(with arm source tag)"]
        GROUP["GROUP BY (file_id, page_number)\ndeduplicates summary + raw rows\nfor the same physical page"]
        SCORE["SUM(1 / (50 + rank))\nper (file_id, page_number)"]
        SORT["ORDER BY rrf_score DESC"]
        TOP_K_RESULTS["LIMIT 10\n(top pages returned)"]
        UNION --> GROUP --> SCORE --> SORT --> TOP_K_RESULTS
    end

    QUERY --> ARM_SUMMARY_VEC & ARM_RAW_VEC & ARM_KEYWORD
    A1_RESULTS --> UNION
    A2_RESULTS --> UNION
    A2_ZERO --> UNION
    A3_RESULTS --> UNION
    A3_ZERO --> UNION
```

### RRF Scoring Visualization

The following shows how three hypothetical pages score across arms. Page B appears in all three arms and wins despite not ranking first in any single arm.

```mermaid
xychart-beta
    title "RRF Score Contribution by Arm (k=50)"
    x-axis ["Page A", "Page B", "Page C", "Page D"]
    y-axis "RRF Score" 0 --> 0.06
    bar [0.0196, 0.0189, 0.0182, 0.0175]
    bar [0.0000, 0.0185, 0.0000, 0.0179]
    bar [0.0000, 0.0182, 0.0179, 0.0000]
```

```mermaid
flowchart LR
    subgraph Example["Example: 4 pages, 3 arms"]
        direction TB

        subgraph Arm1Col["Arm 1 (Vector/Summary)"]
            A1_1["Rank 1: Page A → 1/(50+1) = 0.0196"]
            A1_2["Rank 2: Page B → 1/(50+2) = 0.0192"]
            A1_3["Rank 3: Page C → 1/(50+3) = 0.0189"]
            A1_4["Rank 4: Page D → 1/(50+4) = 0.0185"]
        end

        subgraph Arm2Col["Arm 2 (Vector/Raw)"]
            A2_1["Rank 1: Page B → 1/(50+1) = 0.0196"]
            A2_2["Rank 2: Page D → 1/(50+2) = 0.0192"]
            A2_3["(Page A not in arm 2)"]
            A2_4["(Page C not in arm 2)"]
        end

        subgraph Arm3Col["Arm 3 (Keyword)"]
            A3_1["Rank 1: Page C → 1/(50+1) = 0.0196"]
            A3_2["Rank 2: Page B → 1/(50+2) = 0.0192"]
            A3_3["(Page A not in arm 3)"]
            A3_4["(Page D not in arm 3)"]
        end

        subgraph Final["Final RRF Scores"]
            F_A["Page A: 0.0196 + 0 + 0 = 0.0196"]
            F_B["Page B: 0.0192 + 0.0196 + 0.0192 = 0.0580 ← WINNER"]
            F_C["Page C: 0.0189 + 0 + 0.0196 = 0.0385"]
            F_D["Page D: 0.0185 + 0.0192 + 0 = 0.0377"]
        end
    end
```

Page B wins because it's relevant across all three retrieval signals. Page A ranks first in arm 1 but appears nowhere else, so it loses to pages with broader relevance. This is the core insight of RRF: cross-arm agreement is a stronger signal than single-arm dominance.

### GROUP BY Deduplication

The `page_index` table has exactly one row per physical page. The asynchronous summarization stage writes the row with `summary` and `summary_embedding` populated, while `raw_text`, `raw_embedding`, and `raw_tsvector` remain null. Background enrichment later updates the same row to fill in these nullable columns. The RRF query groups by `(file_id, page_number)` before scoring to ensure consistent deduplication.

### Parameter Choices

| Parameter | Value | Rationale |
|-----------|-------|-----------|
| `k` (RRF smoothing) | 50 | Near-standard value from the RRF literature (Cormack et al. 2009 used k=60). Values between 30 and 70 produce similar fusion quality — the exact value is not critical. k=50 is the most commonly adopted variant in modern RAG implementations and balances rank-1 dampening with sufficient discrimination between top results. Configurable if load testing reveals a different optimum. |
| Over-fetch per arm | 40 candidates | Enough to capture relevant pages that rank lower in individual arms but score well in fusion. |
| Final top-K | 10 pages | Balances context quality against Gemini's context window budget. |

---

## Graceful Degradation

Arms 2 and 3 return zero rows when background enrichment hasn't completed. The RRF query handles this naturally: zero rows from an arm contribute nothing to the fusion, and the query still returns results from arm 1. No special-case logic is needed.

```mermaid
flowchart TD
    subgraph Enriched["Enriched Mode (background stage complete)"]
        E_ARM1["Arm 1: 40 summary candidates"]
        E_ARM2["Arm 2: 40 raw-text vector candidates"]
        E_ARM3["Arm 3: 40 keyword candidates"]
        E_RRF["RRF fusion over 3 arms\nup to 120 unique candidates\n→ top 10"]
        E_ARM1 & E_ARM2 & E_ARM3 --> E_RRF
    end

    subgraph SummaryOnly["Summary-Only Mode (summarization complete, enrichment pending)"]
        S_ARM1["Arm 1: 40 summary candidates"]
        S_ARM2["Arm 2: 0 rows (raw_embedding null)"]
        S_ARM3["Arm 3: 0 rows (raw_tsvector null)"]
        S_RRF["RRF fusion over 1 effective arm\n→ top 10 (from summaries only)"]
        S_ARM1 & S_ARM2 & S_ARM3 --> S_RRF
    end

    subgraph PageCtx["Page Context Assembly"]
        E_CTX["Enriched: original PDF page\n+ detailed summary\n+ matching raw text chunks"]
        S_CTX["Summary-only: original PDF page\n+ detailed summary\n(no raw text chunks)"]
    end

    E_RRF --> E_CTX
    S_RRF --> S_CTX

    NOTE["Both modes return valid results.\nNo error. No user-visible difference\nexcept retrieval depth."]
    E_CTX & S_CTX --> NOTE
```

The user experience is identical in both modes. The agent answers from whatever is available. As background enrichment completes, subsequent queries automatically get richer results without any intervention.

---

## The Query Tool

### Why createDocumentQueryTool, Not createVectorQueryTool

AI SDK's `createVectorQueryTool` lets the LLM control the filter applied to vector search. That's a security problem for multi-tenant access control. If the LLM decides which `userId` or `threadId` to filter by, a prompt injection attack could instruct it to filter by a different user's ID and retrieve their documents.

`createDocumentQueryTool` is a server-side factory that returns the `searchDocument` tool. The server applies the `threadId` and `userId` filter from `requestContext` before the query runs; the LLM never sees or controls these values. It only provides the query string.

```mermaid
flowchart TD
    subgraph Unsafe["createVectorQueryTool (NOT used)"]
        LLM_FILTER["LLM controls filter\n{ userId: '...', threadId: '...' }"]
        RISK["Risk: prompt injection\ncould leak other users' documents"]
        LLM_FILTER --> RISK
    end

    subgraph Safe["createDocumentQueryTool (used)"]
        SERVER_CTX["requestContext\n{ userId, threadId }\n(from JWT middleware)"]
        LLM_QUERY["LLM provides only:\n{ query: 'what does it say about pricing?' }"]
        MERGE["Server merges:\nquery + server-side filter"]
        SAFE_SEARCH["Hybrid search\nscoped to userId + threadId\n(or global scope — see §8)"]
        SERVER_CTX --> MERGE
        LLM_QUERY --> MERGE
        MERGE --> SAFE_SEARCH
    end
```

### Tool Flow

```mermaid
sequenceDiagram
    participant Agent as Orchestrator Agent
    participant Tool as searchDocument (created by createDocumentQueryTool)
    participant Embed as EMBEDDING_PROVIDER
    participant DB as Postgres (page_index)
    participant S3 as Object Storage

    Agent->>Tool: call({ query: "what are the pricing tiers?", document_id?: "file-abc" })

    Note over Tool: Server injects userId + threadId
from requestContext (not from LLM)
document_id is optional — when present,
search is scoped to that file only

    Tool->>Embed: embed(query)
    Embed-->>Tool: queryEmbedding[3072]

    Tool->>DB: hybridSearch(queryEmbedding, userId, threadId, topK=10)
    Note over DB: 3-arm RRF fusion
(or 1-arm if summary-only)
    DB-->>Tool: top 10 pages [(file_id, page_number, rrf_score)]

    Tool->>S3: fetch PDF page images (parallel, top-K pages)
    S3-->>Tool: page image files (multimodal file parts)

    Tool->>DB: getPageContextForPages(matchedPages)
    DB-->>Tool: { summary, matchingChunks } per page

    Tool-->>Agent: evidence bundle (matched pages + context bundles for Evidence Gate + response generation)
```

### S3 Fetch Latency

Fetching the original PDF page image from S3 takes roughly 50ms per page. For the top 10 pages, that's 10 parallel fetches completing in about 50-80ms total. The response model's generation (after the Evidence Bundle Gate) takes 2-5 seconds. The S3 fetches are invisible in the overall latency budget.

The images are sent as multimodal file parts, not base64-encoded strings. This keeps the request payload smaller and lets Gemini process them natively.

---

## Page Context Assembly

For each page returned by hybrid search, the tool assembles a three-part context bundle and returns it to the agent for Evidence Gate + response generation.

```mermaid
flowchart TD
    subgraph Matched["Matched Page: (file_id=abc, page_number=5)"]
        direction TB

        subgraph Part1["Part 1: Original PDF Page Image"]
            S3_FETCH["Fetch from S3\nusing file_id + page_number\n→ multimodal file part"]
            IMG_NOTE["Gemini sees the actual page\nCharts, tables, diagrams included\nNot a text description"]
            S3_FETCH --> IMG_NOTE
        end

        subgraph Part2["Part 2: Detailed Summary"]
            SUMM_FETCH["page_index.summary\nfor this (file_id, page_number)"]
            SUMM_NOTE["LLM-generated structured prose\nCaptures semantic meaning\nAlways available"]
            SUMM_FETCH --> SUMM_NOTE
        end

        subgraph Part3["Part 3: Matching Raw Text Chunks"]
            CHUNK_FETCH["Raw text segments that\nscored in arms 2 or 3\nfor this page"]
            CHUNK_NOTE["Verbatim text\nExact quotes possible\nOnly available if enriched"]
            CHUNK_FETCH --> CHUNK_NOTE
        end
    end

    subgraph Bundle["Context Bundle → Response Generation"]
        ASSEMBLE["Assemble per-page bundle:\n[image_part, summary_text, chunk_text]"]
        MULTI_PAGE["Repeat for each of top-K pages\n(up to 10 bundles)"]
        GENERATE["generateAnswerFromEvidence\n(after gate, all bundles + query)"]
        ASSEMBLE --> MULTI_PAGE --> GENERATE
    end

    Part1 & Part2 & Part3 --> ASSEMBLE

    subgraph SummaryOnlyBundle["Summary-Only Mode (no raw chunks)"]
        SO_ASSEMBLE["Assemble per-page bundle:\n[image_part, summary_text]\n(no chunk_text — not yet enriched)"]
    end

    NOTE_SO["If raw_text is null for a page,\nPart 3 is omitted.\nParts 1 and 2 still sent."]
    NOTE_SO --> SO_ASSEMBLE
```

### Why Send All Three

Each part contributes something the others can't:

- **The image** lets Gemini read charts, tables, and diagrams that text extraction misses entirely. A bar chart's meaning is in its visual structure, not in the OCR'd axis labels.
- **The summary** provides semantic context. It describes what the page is about in structured prose, which helps Gemini understand the page's role in the document even before reading the raw text.
- **The raw text chunks** enable verbatim quotes. When the user asks "what exactly does it say about X?", the answer should quote the document directly. Summaries paraphrase; raw text doesn't.

Sending all three to a multimodal model is the right call. The cost is a slightly larger prompt. The benefit is answers that are simultaneously accurate, well-contextualized, and quotable.

---

## Structured Citations

### Citation Type

Every final response includes a `citations` array. Each citation is a structured object, not a free-text reference.

```mermaid
classDiagram
    class Citation {
        +string source
        +string fileId [optional]
        +integer page [optional]
        +string quote
        +string scope [optional]
        +ImageRef[] images [optional]
    }

    class ImageRef {
        +string url
        +string description
        +integer imageIndex
        +string apiPath
    }

    Citation "1" --> "0..*" ImageRef : images
```

| Field | Type | Description |
|-------|------|-------------|
| `source` | string | Human-readable label shown in the UI. Typically the filename: `"Q3 Report.pdf"`. |
| `fileId` | string? | Machine identifier for API calls. Used to fetch the file, navigate to the page, or request a presigned URL. Present for both PDF and TXT citations (both are uploaded files with stable identifiers). Absent only for web grounding citations. |
| `page` | integer? | Page number within the document. 1-indexed. Optional — absent for non-PDF sources (TXT chunks, web grounding) where pages don't apply. |
| `quote` | string | Supporting excerpt from the document. When raw text is available (enriched status), this is a verbatim extract from the source text. When only summaries are available (ready status), this is an excerpt from the page summary — not verbatim from the original document. |
| `scope` | string | `'thread'` or `'global'`. Added by cross-conversation RAG (see §8). Absent for standard single-thread queries. |
| `images` | ImageRef[] | Visual references on the cited page. Present when the cited content includes charts, tables, or diagrams. |

### ImageRef Fields

| Field | Type | Description |
|-------|------|-------------|
| `url` | string | Presigned S3 URL with a 7-day TTL. The primary way the UI displays the image. |
| `description` | string | Short description of the image content: `"Revenue bar chart, Q1-Q4 2024"`. |
| `imageIndex` | integer | Position of this image on the page (for pages with multiple images). |
| `apiPath` | string | Fallback API endpoint that regenerates a presigned URL when the 7-day TTL expires. |

### Citation Flow

```mermaid
flowchart TD
    subgraph Generation["generateObject Call"]
        SCHEMA["AnswerSchema:\n{ answer: string, citations: Citation[] }"]
        GEMINI["Gemini generates structured output\nmatching the schema exactly"]
        SCHEMA --> GEMINI
    end

    subgraph CitationConstruction["Citation Construction"]
        GEMINI_OUT["Raw citation from Gemini:\n{ source, fileId, page, quote }"]
        ENRICH["Server enriches:\n+ scope (from search metadata)\n+ images (from page_index image refs)\n+ presigned URLs (from S3)"]
        FINAL["Final Citation:\n{ source, fileId, page, quote, scope, images }"]
        GEMINI_OUT --> ENRICH --> FINAL
    end

    subgraph UI["Client UI"]
        DISPLAY["Render inline citation\n[source, page N]"]
        HOVER["On hover: show quote"]
        CLICK["On click: open document\nat cited page"]
        IMAGE_SHOW["If images[]: show thumbnails\nwith fallback to apiPath"]
    end

    GEMINI --> GEMINI_OUT
    FINAL --> DISPLAY & HOVER & CLICK & IMAGE_SHOW
```

The `generateObject` call uses the Vercel AI SDK's structured output feature. The schema is passed to the model, which returns JSON that conforms to it. This eliminates citation parsing — there's no regex or post-processing needed to extract citations from prose.

---

## Cross-Conversation RAG

### The Problem

A user uploads a company handbook. They want to query it from any conversation, not just the one where they uploaded it. Without cross-conversation RAG, every new thread starts with an empty document context.

### Scope Model

Every uploaded document has a `scope` field: either `'thread'` or `'global'`.

```mermaid
flowchart TD
    subgraph Upload["Document Upload"]
        SCOPE_CHOICE{scope parameter}
        THREAD_SCOPE["scope = 'thread'\nDefault behavior\nDocument accessible only\nwithin the uploading thread"]
        GLOBAL_SCOPE["scope = 'global'\nDocument accessible\nacross all threads\nfor this user"]
        SCOPE_CHOICE -->|"not specified"| THREAD_SCOPE
        SCOPE_CHOICE -->|"scope: 'global'"| GLOBAL_SCOPE
    end

    subgraph Storage["page_index rows"]
        THREAD_ROW["thread_id = <actual threadId>\nscope = 'thread'"]
        GLOBAL_ROW["thread_id = '__global__'\nscope = 'global'"]
        THREAD_SCOPE --> THREAD_ROW
        GLOBAL_SCOPE --> GLOBAL_ROW
    end

    subgraph Query["Query Time"]
        FILTER["WHERE user_id = :userId\nAND (thread_id = :threadId\n  OR thread_id = '__global__')"]
        THREAD_ROW & GLOBAL_ROW --> FILTER
    end
```

Global documents use a sentinel `threadId` value of `'__global__'`. Because this sentinel is a string literal, `thread_id` is stored as `TEXT` (not `UUID`). The query filter always includes both the current thread and the global sentinel. This means global documents are automatically included in every search without any special-case logic.

### Thread vs Global Ranking

Thread-scoped results rank higher than global results in the RRF fusion. A document uploaded in the current conversation is more likely to be what the user is asking about than a document uploaded weeks ago in a different thread.

```mermaid
flowchart LR
    subgraph RRF_Boost["RRF with Scope Boost"]
        THREAD_RESULT["Thread-scoped page\nRRF score: 0.058"]
        GLOBAL_RESULT["Global-scoped page\nRRF score: 0.058 × 0.85 = 0.049"]
        NOTE["Thread results get full RRF score.\nGlobal results get a 0.85 multiplier.\nSame content, different priority."]
        THREAD_RESULT & GLOBAL_RESULT --> NOTE
    end
```

The boost factor is configurable. The default (0.85) means global documents are still retrieved and cited, but thread documents win ties.

### Cross-Conversation RAG Flow

```mermaid
sequenceDiagram
    participant User
    participant Thread1 as Thread A (handbook upload)
    participant Thread2 as Thread B (new conversation)
    participant DB as page_index

    User->>Thread1: Upload "company-handbook.pdf" (scope: global)
    Thread1->>DB: INSERT page_index rows\n(thread_id='__global__', scope='global')

    Note over Thread1,DB: Handbook is now globally accessible

    User->>Thread2: "What's the vacation policy?"
    Thread2->>DB: hybridSearch(query, userId, threadId=Thread2)\nWHERE user_id=userId\nAND (thread_id=Thread2 OR thread_id='__global__')
    DB-->>Thread2: Handbook pages (from __global__)\n+ any Thread2 documents

    Thread2-->>User: Answer with citation\n{ source: "company-handbook.pdf", scope: "global", page: 12 }
```

### Security Boundary

Global scope is per-user. A document uploaded as global by User A is never visible to User B. The `user_id` filter is always applied first, before the thread filter. There's no way to query across user boundaries.

```mermaid
flowchart TD
    subgraph UserA["User A"]
        A_GLOBAL["company-handbook.pdf\nscope: global\nuser_id: user_a"]
        A_THREAD["q3-report.pdf\nscope: thread\nuser_id: user_a"]
    end

    subgraph UserB["User B"]
        B_GLOBAL["my-notes.pdf\nscope: global\nuser_id: user_b"]
    end

    subgraph QueryA["User A's query"]
        A_FILTER["WHERE user_id = 'user_a'\nAND (thread_id = :threadId\n  OR thread_id = '__global__')"]
        A_SEES["Sees: company-handbook.pdf + q3-report.pdf\n(if in current thread)"]
        A_FILTER --> A_SEES
    end

    subgraph QueryB["User B's query"]
        B_FILTER["WHERE user_id = 'user_b'\nAND (thread_id = :threadId\n  OR thread_id = '__global__')"]
        B_SEES["Sees: my-notes.pdf\nNEVER sees User A's documents"]
        B_FILTER --> B_SEES
    end

    A_GLOBAL & A_THREAD --> A_FILTER
    B_GLOBAL --> B_FILTER
```

Documents never leak across user boundaries. The `user_id` column is not nullable and is always included in the WHERE clause. There's no admin override or cross-user query path.

---

## Large TXT RAG

PDF files go through the `page_index` system. Large plain-text files (`.txt`, `.md`, code files) use a separate chunking pipeline built on AI SDK's RAG utilities.

### Why a Separate Pipeline

PDFs have natural page boundaries. The page is the right unit of retrieval: it maps to a physical location in the document, it's the right size for a multimodal LLM to process, and it gives users a meaningful citation ("page 5"). Plain text has no pages. It needs to be chunked at the semantic level.

### Chunking Strategy

```mermaid
flowchart TD
    subgraph Input["Large TXT File"]
        RAW_TXT["Raw text content\n(potentially hundreds of KB)"]
    end

    subgraph Chunking["custom text chunker Chunking (custom pgvector via Drizzle)"]
        MDOC["custom text chunker instance\nwrapping the raw text"]
        CHUNK_CFG["Chunking config:\nmaxSize: 4000 chars (~1000 tokens)\noverlap: 800 chars (~200 tokens)"]
        CHUNKS["Text chunks\n(overlapping windows)"]
        MDOC --> CHUNK_CFG --> CHUNKS
    end

    subgraph Embedding["Embedding + Storage"]
        EMBED_CHUNKS["Embed each chunk\n(EMBEDDING_PROVIDER)"]
        PGVEC["PgVector (custom pgvector via Drizzle)\nstores chunks + embeddings\nin Postgres"]
        EMBED_CHUNKS --> PGVEC
    end

    subgraph Retrieval["Query Time"]
        Q_EMBED["Embed query"]
        VEC_SEARCH["PgVector similarity search\nreturns top-K chunks"]
        CHUNKS_OUT["Relevant text chunks\n(with source file + position metadata)"]
        Q_EMBED --> VEC_SEARCH --> CHUNKS_OUT
    end

    RAW_TXT --> MDOC
    CHUNKS --> EMBED_CHUNKS
    PGVEC --> VEC_SEARCH
```

### Chunk Size Rationale

The 4000-character chunk size (roughly 1000 tokens) is larger than many RAG implementations use. Smaller chunks (256-512 tokens) improve precision but hurt recall: a question about a concept that spans two paragraphs may not match any single small chunk well. Larger chunks keep related content together, which improves retrieval quality for questions that require multi-sentence context.

The 800-character overlap ensures that content near chunk boundaries appears in at least two chunks. A sentence split across a boundary will be fully present in one of the two overlapping chunks.

### PgVector vs page_index

| Aspect | page_index (PDFs) | PgVector (TXT) |
|--------|-------------------|----------------|
| Unit of retrieval | Page | Chunk (4000 chars) |
| Citation | Page number | Character offset / chunk index |
| Visual content | Yes (PDF page image) | No |
| Hybrid search | 3-arm RRF | Vector only |
| Implementation | Custom SQL | custom pgvector via Drizzle PgVector class |

The two systems are independent. A query tool for TXT files uses PgVector search directly. A query tool for PDFs uses the hybrid RRF pipeline. The agent uses the returned evidence bundles to produce the same `Citation[]` structure in the final response.

---

## Cross-References

| Component | Interaction |
|-----------|-------------|
| **Document Processing** ([08](./08-document-processing.md)) | Blocking stage writes summaries + summary embeddings to `page_index`. Background stage writes raw text + raw embeddings + tsvector. |
| **File Intelligence** ([12](./12-file-intelligence.md)) | FileRegistry resolves file references before the query tool runs. The Evidence Bundle Gate applies after hybrid search returns results. |
| **Agent & Orchestration** ([05](./05-agent-and-orchestration.md)) | `searchDocument` (created by `createDocumentQueryTool`) is registered as an agent tool. The agent calls it when the query involves uploaded documents. |
| **Query Pipeline** ([11](./11-query-pipeline.md)) | `document_qa` is a valid source in `sourcesPriority`. The source router calls the document query tool as part of the parallel fan-out. |
| **Configuration** ([02](./02-configuration.md)) | `EMBEDDING_PROVIDER`, `EMBEDDING_DIMS`, `DATABASE_URL`, `S3_BUCKET`, `PRIMARY_MODEL` all affect RAG behavior. |
| **Memory System** ([07](./07-memory-system.md)) | Memory recall and document RAG are separate sources. Both can appear in `sourcesPriority` for the same topic. |

---

## Task Specifications

### Task RAG_INFRA: RAG Infrastructure

**What to do**: Build the complete hybrid search pipeline for PDF documents. Implement the three-arm RRF fusion query against `page_index`, the `getPageContextForPages` function that assembles per-page context bundles, and the `createDocumentQueryTool` factory with server-side access control (returns the `searchDocument` tool — retrieval only, returns an evidence bundle). Implement the `generateAnswerFromEvidence` helper function that uses `generateObject` to produce `{ answer, citations: Citation[] }` from the evidence bundle — this helper is invoked by the agent orchestration layer after the Evidence Gate opens, NOT by `searchDocument` itself. Include the graceful degradation path for summary-only mode.

**Depends on**: STORAGE_WRAPPER, DOC_PIPELINE

**Acceptance Criteria**:
- The `createDocumentQueryTool` factory (accepting database, file storage, and optional config) returns a AI SDK-compatible tool definition
- `userId` and `threadId` filters come from `requestContext`, never from LLM input
- Hybrid search runs all three arms; arms 2 and 3 return zero rows gracefully when `raw_embedding` / `raw_tsvector` are null
- RRF uses `k=50`, over-fetches 40 candidates per arm, returns top 10 pages
- `BY` deduplicates rows before scoring
- `getPageContextForPages` fetches S3 page images in parallel (not sequentially)
- Evidence bundle includes all three parts per matched page: image, summary, matching chunks
- Summary-only pages include image + summary (no chunks — not an error)
- `generateAnswerFromEvidence` helper uses `generateObject` with a typed schema: `{ answer: string, citations: Citation[] }` — called by orchestration layer post-gate, not by `searchDocument`
- Citations include `source`, `fileId`, `page`, `quote` at minimum
- `ImageRef` objects include presigned S3 URL with 7-day TTL and `apiPath` fallback
- Unit tests with mocked DB and mocked S3
- Integration test: upload a PDF, run a query, verify citations reference correct pages

**QA Scenarios**:
- Query against a freshly uploaded PDF (summary-only, no raw text) → returns valid answer from summaries, no error
- Query against an enriched PDF → three-arm fusion runs, citations include verbatim quotes from raw text
- Query with no relevant pages → `citations` array is empty, `answer` acknowledges no relevant content found
- Two users with documents on the same topic → each user's query returns only their own documents
- S3 fetch fails for one page → that page is skipped, remaining pages still processed
- `generateObject` schema validation fails → typed error propagated to agent, not silent null
- Top-K pages include pages from multiple files → citations correctly attribute each page to its source file

---

### Task CROSS_CONV_RAG: Cross-Conversation RAG

**What to do**: Extend the `page_index` schema and hybrid search pipeline to support global-scope documents. Add a `scope` field (`'thread'` | `'global'`) and use a sentinel `threadId` value of `'__global__'` for global documents. Update the hybrid search query to include global documents in every search. Apply a configurable rank boost that favors thread-scoped results over global ones. Add `scope` to the `Citation` type. Implement the upload API parameter that lets clients specify global scope.

**Depends on**: RAG_INFRA (RAG Infrastructure — must be complete before scope can be layered on top)

**Acceptance Criteria**:
- `page_index` has a non-nullable `scope` column (`'thread'` | `'global'`)
- Global documents stored with `thread_id = '__global__'`
- Hybrid search WHERE clause includes user isolation and dual scope: `WHERE user_id = :userId AND (thread_id = :threadId OR thread_id = '__global__')`
- Thread-scoped results receive full RRF score; global results receive a configurable multiplier (default 0.85)
- `Citation.scope` is `'thread'` or `'global'` depending on the source document
- Upload API accepts an optional `scope` parameter; defaults to `'thread'` if not provided
- Global documents are accessible from any thread belonging to the same user
- Global documents are never accessible to a different user (user_id filter always applied)
- Existing thread-scoped queries are unaffected when no global documents exist for the user
- Unit tests: verify global documents appear in cross-thread queries
- Unit tests: verify global documents from User A never appear in User B's queries
- Integration test: upload a global document in Thread A, query from Thread B, verify citation appears with `scope: 'global'`

**QA Scenarios**:
- User uploads handbook as global, queries from a new thread → handbook pages appear in results with `scope: 'global'`
- User uploads two documents: one thread-scoped, one global. Query matches both → thread-scoped result ranks higher
- User A uploads a global document. User B queries on the same topic → User B sees no results from User A's document
- User uploads a document as thread-scoped (default). Queries from a different thread → document not found (correct behavior)
- Global document upload, then user deletes it → subsequent queries from any thread no longer return it
- Query in a thread with both thread and global matches → `citations` array contains both, each with correct `scope` value
- Rank boost multiplier set to 1.0 in config → thread and global results treated equally (no preference)

---

## External References

- AI SDK RAG overview: https://sdk.vercel.ai/docs
- Vercel AI SDK `generateObject`: https://sdk.vercel.ai/docs/ai-sdk-core/generating-structured-data
- Reciprocal Rank Fusion: Cormack, Clarke, and Buettcher (2009). "Reciprocal Rank Fusion Outperforms Condorcet and Individual Rank Learning Methods." SIGIR 2009.

---

*Previous: [08 — Document Processing](./08-document-processing.md)*
*Next: [10 — Intent Detection & Routing](./10-intent-and-routing.md)*
