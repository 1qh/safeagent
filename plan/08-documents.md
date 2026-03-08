# 08 — Document Processing

> **Scope**: Upload pipeline, multimodal-first processing, per-page summarization, progressive retrieval, S3 storage layout, file metadata tables, and cleanup.
>
> **Why this document exists**: Document processing is the most operationally complex part of safeagent. A large PDF (up to the configured upload limit) touches six distinct systems before a user can query it. This document defines every step, every routing decision, every table column, and every failure mode so the implementation has no ambiguity.

---

## Table of Contents

- [Upload Pipeline Overview](#upload-pipeline-overview)
- [Document Routing by Type and Size](#document-routing-by-type-and-size)
- [DOCX to PDF Conversion](#docx-to-pdf-conversion)
- [Blocking Stage: Per-Page Summarization](#blocking-stage-per-page-summarization)
- [Background Stage: Raw Text Enrichment](#background-stage-raw-text-enrichment)
- [File Status State Machine](#file-status-state-machine)
- [Per-Page Streaming Architecture](#per-page-streaming-architecture)
- [Image Extraction Pipeline](#image-extraction-pipeline)
- [S3 Storage Layout](#s3-storage-layout)
- [page_index Table Design](#page_index-table-design)
- [File Metadata Tables](#file-metadata-tables)
- [Progress Tracking](#progress-tracking)
- [Cleanup](#cleanup)
- [Cross-References](#cross-references)
- [Task Specifications](#task-specifications)
- [External References](#external-references)

---

## Upload Pipeline Overview

The upload pipeline runs in two phases. The synchronous phase (client through S3 storage) completes in under a second and returns a `fileId` to the client. The processing phase runs on the server and may take seconds to minutes depending on file size and page count.

```mermaid
flowchart TD
    CLIENT["Client\nmultipart file upload"]

    subgraph VALIDATE["Validation (synchronous)"]
        MAGIC["Magic bytes check\n(not extension-based)"]
        SIZE["Size check\n≤5MB per file\n≤5 files per turn"]
        QUOTA["Quota check\n≤100MB per user\n(Postgres + Valkey)"]
        TYPE["Type allowlist\nPDF · DOCX · TXT\nPNG · JPG · JPEG · WEBP\nReject: .doc (legacy binary)"]
    end

    subgraph STORE["Storage (synchronous)"]
        STORAGE_ORIGINAL_OBJECT["Store original object in S3\nusing a hierarchical key\norganized by user, thread, and file identity"]
        PG_META["INSERT file_uploads\n(Drizzle ORM)\nstatus = 'uploading'"]
        QUOTA_UP["UPDATE user_storage_quotas\n+byteSize"]
    end

    RESPOND["Return {fileId, status: 'uploading'}\nto client (201 Created)"]

    subgraph ROUTE["Routing decision (synchronous)"]
        DETECT_TYPE["Detect type + page count"]
        SET_MODE["Set mode:\ndirect / indexed / rag"]
        STATUS_SUMM["UPDATE status = 'summarizing'\n(before blocking stage begins)"]
    end

    subgraph PROCESS["Processing (blocking stage)"]
        DOCX_CONV["DOCX → PDF\n(LibreOffice headless)"]
        PDF_SPLIT["pdf-lib: split PDF\ninto per-page PDFs"]
        IMG_EXTRACT["pdfjs: extract raster images\nper page"]
        GEMINI_SUM["Gemini generateObject\nper-page summarization"]
        EMBED_SUM["Embed summaries\n→ EMBEDDING_PROVIDER"]
        STORE_IDX["INSERT page_index rows\n(summary + vector)"]
        STORAGE_PAGE_PDFS["PUT per-page PDFs to S3"]
        STORAGE_PAGE_IMAGES["PUT extracted images to S3"]
        STATUS_READY["UPDATE status = 'ready'"]
    end

    subgraph BACKGROUND["Background stage (Trigger.dev)"]
        STATUS_ENRICHING["UPDATE status = 'enriching'"]
        UNPDF["unpdf: extract raw text\nper page"]
        TRUNC["Truncate to 1800 tokens"]
        EMBED_RAW["Embed raw text\n→ EMBEDDING_PROVIDER"]
        STORE_RAW["UPDATE page_index rows\n(raw_text + raw_embedding + tsvector)"]
        STATUS_ENRICHED["UPDATE status = 'enriched'"]
    end

    CLIENT --> MAGIC --> SIZE --> QUOTA --> TYPE
    TYPE --> STORAGE_ORIGINAL_OBJECT --> PG_META --> QUOTA_UP --> RESPOND
    RESPOND --> DETECT_TYPE --> SET_MODE

    SET_MODE -->|"DOCX"| DOCX_CONV
    DOCX_CONV -->|">6 pages"| PDF_SPLIT
    DOCX_CONV -->|"≤6 pages"| STATUS_READY
    SET_MODE -->|"PDF >6 pages"| PDF_SPLIT
    PDF_SPLIT --> IMG_EXTRACT --> GEMINI_SUM
    GEMINI_SUM --> EMBED_SUM --> STORE_IDX
    GEMINI_SUM --> STORAGE_PAGE_PDFS
    IMG_EXTRACT --> STORAGE_PAGE_IMAGES
    STORE_IDX --> STATUS_READY

    STATUS_READY --> UNPDF --> TRUNC --> EMBED_RAW --> STORE_RAW --> STATUS_ENRICHED

    SET_MODE -->|"TXT >8K tokens"| TXT_CHUNK["Chunk text\n(4000 chars, 800 overlap)"]
    TXT_CHUNK --> TXT_EMBED["Embed chunks\nvia PgVector"]
    TXT_EMBED --> TXT_STORE["Store in txt_chunks table"]
    TXT_STORE --> STATUS_READY

    SET_MODE -->|"PDF ≤6 pages\nTXT ≤8K tokens\nImage"| STATUS_READY
```

### Validation Details

**Magic bytes check**: The server reads the first 8–16 bytes of each uploaded file and compares against known file signatures. A file named `malicious.pdf` that is actually a ZIP archive fails this check. Extension alone is never trusted.

**Size limits**: Each individual file must be ≤5MB. A single turn may include up to 5 files. These limits are enforced before any S3 write happens.

**Quota enforcement**: `checkStorageQuota(userId)` reads the user's current storage usage from Postgres (`user_storage_quotas`) with a Valkey cache for speed and compares it against their limit (default 100MB). If the upload would exceed the quota, the request is rejected with a clear error before any storage write.

**Quota reservation atomicity**: Storage quota enforcement uses atomic reservation. Before accepting file bytes, the system atomically increments the user's `used_bytes` counter. If the result exceeds quota, the upload is rejected immediately. On processing failure, the reservation is rolled back. This prevents concurrent uploads from exceeding quotas.

**Supported types**:

| Type | MIME | Notes |
|------|------|-------|
| PDF | `application/pdf` | Native processing |
| DOCX | `application/vnd.openxmlformats-officedocument.wordprocessingml.document` | Converted to PDF first |
| TXT | `text/plain` | Chunked directly |
| PNG | `image/png` | Direct mode only |
| JPG / JPEG | `image/jpeg` | Direct mode only |
| WEBP | `image/webp` | Direct mode only |

**Not supported**: `.doc` (legacy binary Word format). The magic bytes for `.doc` are distinct from `.docx`. If detected, the server returns a clear error asking the user to save as `.docx` first.

---

## Document Routing by Type and Size

Every uploaded file gets assigned a `mode` immediately after validation. The mode determines how the file is processed and how it's used at query time.

```mermaid
flowchart TD
    FILE["Uploaded file\n(validated)"]

    TYPE_CHECK{File type?}

    subgraph IMAGE_PATH["Image path"]
        IMG_DIRECT["mode = 'direct'\nReady immediately\nSent to Gemini on query"]
    end

    subgraph TXT_PATH["TXT path"]
        TXT_SIZE{Token count?}
        TXT_DIRECT["mode = 'direct'\n≤8K tokens\nReady immediately"]
        TXT_RAG["mode = 'rag'\n>8K tokens\nChunk → embed → PgVector"]
    end

    subgraph PDF_PATH["PDF / DOCX path"]
        PAGE_COUNT{Page count?}
        PDF_DIRECT["mode = 'direct'\n≤6 pages\nReady immediately\nAll pages sent to Gemini on query"]
        PDF_INDEXED["mode = 'indexed'\n>6 pages\nBlocking stage + background stage"]
    end

    FILE --> TYPE_CHECK
    TYPE_CHECK -->|"PNG / JPG / WEBP"| IMG_DIRECT
    TYPE_CHECK -->|"TXT"| TXT_SIZE
    TXT_SIZE -->|"≤8K tokens"| TXT_DIRECT
    TXT_SIZE -->|">8K tokens"| TXT_RAG
    TYPE_CHECK -->|"PDF / DOCX"| PAGE_COUNT
    PAGE_COUNT -->|"≤6 pages"| PDF_DIRECT
    PAGE_COUNT -->|">6 pages"| PDF_INDEXED
```

### Mode Behavior at Query Time

| Mode | Processing | Query behavior |
|------|-----------|----------------|
| `direct` (image) | None | Image bytes sent directly to Gemini with the query |
| `direct` (PDF/DOCX ≤6 pages) | None | All page PDFs sent to Gemini with the query |
| `direct` (TXT ≤8K tokens) | None | Full text included in context |
| `rag` (TXT >8K tokens) | Chunk → embed → PgVector | Vector search at query time |
| `indexed` (PDF/DOCX >6 pages) | Blocking + background stages | Hybrid search: HNSW + tsvector RRF |

The `direct` mode is intentionally simple. For small files, sending everything to the LLM is cheaper and more accurate than retrieval. The threshold of 6 pages for PDFs and 8K tokens for text is calibrated to stay within Gemini's context window while leaving room for the conversation history and system prompt.

---

## DOCX to PDF Conversion

DOCX files must be converted to PDF before the PDF processing pipeline can run. The conversion uses LibreOffice headless, which runs as a separate Docker service.

```mermaid
flowchart TD
    DOCX_FILE["DOCX file\n(validated, stored in S3)"]

    FETCH_FROM_STORAGE["Fetch original DOCX\nfrom S3 to temp buffer"]

    subgraph LIBREOFFICE["LibreOffice Headless (Docker sidecar :2002)"]
        SEND["Send DOCX to sidecar\nover Docker network\n(30s timeout)"]
        WAIT["Await conversion\n(timeout: 30s)"]
        CHECK_EXIT{Success?}
    end

    PDF_BYTES["Read PDF bytes\nfrom temporary processing output"]
    UPLOAD_PDF["Store converted PDF in S3\nusing the same hierarchical key\norganization"]
    UPDATE_META["UPDATE file_uploads\nmimeType = 'application/pdf'\ns3Key = new PDF key"]

    FAIL_CONVERT["Mark status = 'failed'\nError: conversion failed"]

    CONTINUE["Continue to PDF\nprocessing pipeline"]

    DOCX_FILE --> FETCH_FROM_STORAGE --> SEND --> WAIT --> CHECK_EXIT
    CHECK_EXIT -->|"Yes"| PDF_BYTES --> UPLOAD_PDF --> UPDATE_META --> CONTINUE
    CHECK_EXIT -->|"No"| FAIL_CONVERT
```

### Why LibreOffice

LibreOffice headless is the most faithful DOCX renderer available outside of Microsoft Word itself. It preserves tables, embedded images, headers, footers, and complex formatting that simpler converters lose. The Docker service runs persistently so there's no cold-start cost per conversion.

The API server sends the DOCX file to the LibreOffice sidecar over the internal Docker network on its dedicated service port. The sidecar runs LibreOffice headless, converts the file to PDF, and returns the result. The 30-second timeout handles pathological documents without hanging the pipeline. For local development without Docker, developers can install LibreOffice on the host machine and the conversion module falls back to a direct `Bun.spawn` subprocess call.

**Temp file cleanup**: The temporary processing directory is cleaned after each conversion. LibreOffice writes intermediate output to the system temporary directory, which is ephemeral.

---

## Blocking Stage: Per-Page Summarization

The blocking stage runs synchronously before the file is marked `ready`. The user cannot query the file until this stage completes. It produces the primary retrieval artifacts: per-page summaries with dense vector embeddings.

```mermaid
flowchart TD
    PDF_INPUT["PDF file\n(original or converted from DOCX)"]

    subgraph STEP_A["Step A: PDF Split (pdf-lib)"]
        LOAD_PDF["Load PDF with pdf-lib"]
        COUNT_PAGES["Count pages\nset progress_total"]
        SPLIT_PAGES["Split into N single-page PDFs\neach as base64 string"]
    end

    subgraph STEP_B["Step B: Image Extraction (pdfjs)"]
        GET_OPS["pdfjs getOperatorList\nper page"]
        FILTER_IMGS["Filter images:\nwidth ≥100px AND height ≥100px\n(discard decorative)"]
        DECODE_IMGS["Decode image bytes\nvia JIMP"]
    end

    subgraph STEP_C["Step C: Gemini Summarization"]
        PLIMIT["p-limit concurrency\nConfigurable per-key limit\n(overrides KeyPool default\nfor summarization task)"]
        GEMINI_CALL["generateObject call:\ninput: page PDF (base64) + extracted images\noutput: {summary, imageDescriptions[], hasVectorCharts}"]
        DENSE_PARA["summary: dense paragraphs\n150–400 words\nEach summary = 1 RAG chunk"]
    end

    subgraph STEP_D["Step D: Vector Chart Fallback"]
        CHECK_VECTOR{hasVectorCharts AND\nno raster images?}
        RENDER_PNG["pdfjs canvas:\nrender page to PNG"]
        SKIP_RENDER["Skip render\n(raster images sufficient)"]
    end

    subgraph STEP_E["Step E: Store"]
        EMBED_SUMMARY["Embed summary text\n→ EMBEDDING_PROVIDER"]
        STORAGE_SINGLE_PAGE_PDF["PUT page-{N}.pdf to S3"]
        STORAGE_EXTRACTED_IMAGES["PUT page-{N}-img-{M}.{ext} to S3"]
        INSERT_IDX["UPSERT page_index row\nsummary=page summary\nsummary_embedding=vector\nmetadata={images[]}"]
        UPDATE_PROGRESS["UPDATE progress_current = N"]
    end

    STATUS_READY["UPDATE file_uploads\nstatus = 'ready'"]

    PDF_INPUT --> LOAD_PDF --> COUNT_PAGES --> SPLIT_PAGES
    SPLIT_PAGES --> GET_OPS --> FILTER_IMGS --> DECODE_IMGS
    DECODE_IMGS --> PLIMIT --> GEMINI_CALL --> DENSE_PARA
    DENSE_PARA --> CHECK_VECTOR
    CHECK_VECTOR -->|"Yes"| RENDER_PNG --> EMBED_SUMMARY
    CHECK_VECTOR -->|"No"| SKIP_RENDER --> EMBED_SUMMARY
    EMBED_SUMMARY --> STORAGE_SINGLE_PAGE_PDF
    EMBED_SUMMARY --> STORAGE_EXTRACTED_IMAGES
    EMBED_SUMMARY --> INSERT_IDX --> UPDATE_PROGRESS
    UPDATE_PROGRESS -->|"All pages done"| STATUS_READY
```

### Step A: PDF Split

pdf-lib loads the full PDF and extracts each page as a standalone single-page PDF. Each page is base64-encoded for transmission to Gemini. The page count is written to `progress_total` immediately so the client can show accurate progress.

### Step B: Image Extraction

pdfjs `getOperatorList` returns the raw drawing operations for each page. Image operations are filtered to extract raster images. Images smaller than 100×100 pixels are discarded as decorative (bullets, icons, dividers). The remaining images are decoded with JIMP to get raw bytes and dimensions.

This step is pure PDF parsing. No LLM is involved. The goal is to collect the visual evidence that Gemini will need to produce an accurate summary.

### Step C: Gemini Summarization

Each page is summarized with a `generateObject` call. The call sends:
- The single-page PDF as base64
- Any extracted raster images from that page
- A schema requiring `{ summary: string, imageDescriptions: ImageDescription[], hasVectorCharts: boolean }`

The `summary` field must be a dense paragraph of 150–400 words. This density is intentional: sparse summaries lose retrieval precision. Each summary becomes exactly one RAG chunk in `page_index`.

`imageDescriptions` is an array of structured descriptions for each image on the page. These are stored in the `metadata` JSONB column and used by the visual grounding pipeline.

`hasVectorCharts` signals that the page contains charts drawn as PDF vector graphics (not raster images). These can't be extracted by pdfjs image operations.

**Concurrency**: p-limit controls how many Gemini calls run in parallel. The limit is configurable via `summarizationConcurrencyPerKey` in the upload config (default: 50). This value overrides KeyPool's default `perKeyConcurrency` for the summarization task only. Operators with higher Google AI API rate limits can increase this value (e.g., 150 for paid-tier keys). With a pool of 10 keys and a higher per-key limit, very large PDFs can be processed quickly in benchmark scenarios (e.g., ~200 pages when the upload size limit is increased for internal load testing).

### Step D: Vector Chart Fallback

If `hasVectorCharts` is true and no raster images were extracted in Step B, the page is rendered to a PNG using pdfjs canvas. This PNG is then included in the Gemini call as a supplementary image. The fallback only fires when needed — most pages with charts have raster images already.

### Step E: Store

For each page:
1. The summary is embedded via `EMBEDDING_PROVIDER`
2. The per-page PDF is uploaded to S3
3. Extracted images are uploaded to S3
4. A `page_index` row is inserted or updated with the `summary`, `summary_embedding`, and a `metadata` JSONB object containing image descriptions and S3 keys
5. `progress_current` is incremented

After all pages complete, `file_uploads.status` is set to `'ready'`.

---

## Background Stage: Raw Text Enrichment

The background stage runs after the file is already `ready`. It enriches the existing per-page `page_index` row by populating nullable `raw_text`, `raw_embedding`, and `raw_tsvector` columns for full-text and hybrid retrieval.

```mermaid
flowchart TD
    TRIGGER["Trigger.dev task triggered\nafter blocking stage completes"]

    subgraph PER_PAGE["Per page (parallel)"]
        FETCH_PAGE["Fetch page-{N}.pdf from S3"]
        UNPDF_EXTRACT["unpdf extractText\nraw text from page"]
        TRUNCATE["Truncate to 1800 tokens\n(EMBEDDING_MODEL limit: 2048\n200 token safety margin)"]
        EMBED_RAW["Embed raw text\n→ EMBEDDING_PROVIDER"]
        INSERT_RAW["UPDATE page_index row\nraw_text=extracted text\nraw_embedding=vector\nraw_tsvector=generated"]
    end

    STATUS_ENRICHED["UPDATE file_uploads\nstatus = 'enriched'"]

    subgraph DEV_MODE["Dev mode fallback"]
        IN_PROCESS["Run in-process\n(no Trigger.dev required)\nSame logic, no queue"]
    end

    TRIGGER --> FETCH_PAGE --> UNPDF_EXTRACT --> TRUNCATE --> EMBED_RAW --> INSERT_RAW
    INSERT_RAW -->|"All pages done"| STATUS_ENRICHED

    TRIGGER -.->|"Trigger env vars absent"| IN_PROCESS
```

### Why Two Representations Per Page

The `summary` representation (from the blocking stage) is optimized for semantic retrieval. It's a dense, LLM-generated paragraph that captures the meaning of the page. It retrieves well for conceptual queries.

The `raw_text` representation (from the background stage) is optimized for keyword retrieval. The tsvector enables BM25-style full-text search that finds exact terms the summary might have paraphrased. Together, these two representations feed a Reciprocal Rank Fusion query that outperforms either alone.

### Token Truncation

unpdf may extract more text than the embedding model can handle. The raw text is truncated to 1800 tokens before embedding. The embedding model's hard limit is 2048 tokens; the 200-token margin prevents edge cases from failing silently.

### Dev Mode

When `TRIGGER_DEV_API_URL` and `TRIGGER_DEV_API_KEY` are absent, the background stage runs in-process immediately after the blocking stage, without requiring a Trigger.dev deployment. This makes local development possible without the full Docker Compose stack. The adapter selection is handled by `createQueueAdapter` (see 15-Infrastructure).

---

## File Status State Machine

```mermaid
stateDiagram-v2
    [*] --> UPLOADING : Client sends file

    UPLOADING --> SUMMARIZING : Validation passes\nS3 write succeeds\nBlocking stage begins

    UPLOADING --> FAILED : Validation fails\nor S3 write fails

    SUMMARIZING --> READY : All pages summarized\npage_index rows inserted\n(blocking stage complete)

    SUMMARIZING --> FAILED : Gemini error\nor pdf-lib error\nor timeout

    READY --> ENRICHING : Trigger.dev task starts\n(background stage begins)

    ENRICHING --> ENRICHED : All raw_text columns populated\nraw_tsvectors generated\n(background stage complete)

    ENRICHING --> READY : Background stage fails\n(file still queryable\nfrom summary entries)

    READY --> DELETED : cleanupThread called

    ENRICHED --> DELETED : cleanupThread called

    FAILED --> [*]
    DELETED --> [*]
```

### State Descriptions

| Status | Meaning | Queryable? |
|--------|---------|-----------|
| `uploading` | File received, validation in progress | No |
| `summarizing` | Blocking stage running, pages being processed | No |
| `ready` | Blocking stage complete, summary entries available | Yes (semantic search) |
| `enriching` | Background stage running, raw text being indexed | Yes (semantic search) |
| `enriched` | Both stages complete, full hybrid search available | Yes (semantic + keyword) |
| `failed` | Processing failed at some stage | No |
| `deleted` | Cleanup complete | No |

The transition from `enriching` back to `ready` on background stage failure is intentional. The file remains queryable from its summary entries. The user doesn't lose access to their document because the background enrichment failed.

### Stranded File Recovery

If the API process crashes or restarts while a file is in `summarizing` state, the file is stranded — no active process is working on it, and the client sees it stuck forever. The TTL cleanup job (see [15 — Infrastructure](./15-infrastructure.md)) handles recovery:

- On each scheduled run, the cleanup job queries for files in `summarizing` state whose `updated_at` exceeds a configurable staleness threshold (default: 10 minutes).
- For each stranded file, the job resets the file status to `uploading` and re-enqueues the blocking stage via the queue adapter. The re-enqueued job picks up from scratch (idempotent — `page_index` rows are upserted, not inserted).
- If a file has been re-enqueued more than a configurable retry limit (default: 3), the job transitions it to `failed` with a diagnostic reason and emits a structured log at `error` severity.

This guarantees that no file remains permanently stranded in `summarizing` at 10M-user scale, even under rolling deployments or unexpected process termination. The recovery path uses the same queue adapter abstraction as the initial processing — Trigger.dev in production, in-process in development.

---

## Per-Page Streaming Architecture

Each page flows through the blocking stage independently. Pages don't wait for each other. This is what makes very large PDFs process quickly in parallel rather than sequentially.

```mermaid
flowchart TD
    PDF["N-page PDF"]

    subgraph SPLIT["Step A: Split (sequential)"]
        SPLIT_ALL["pdf-lib splits all N pages\ninto N single-page PDFs\n(fast, in-memory)"]
    end

    subgraph POOL["p-limit pool (concurrent)"]
        direction LR
        PAGE_ONE_FLOW["Page 1\nExtract → Summarize → Store"]
        PAGE_TWO_FLOW["Page 2\nExtract → Summarize → Store"]
        PAGE_THREE_FLOW["Page 3\nExtract → Summarize → Store"]
        PDOT["..."]
        PN["Page N\nExtract → Summarize → Store"]
    end

    subgraph PROGRESS["Progress updates (per page)"]
        PROGRESS_FIRST["progress_current = 1"]
        PROGRESS_SECOND["progress_current = 2"]
        PROGN["progress_current = N"]
    end

    subgraph READY["Completion"]
        ALL_DONE["All N futures resolve"]
        SET_READY["status = 'ready'"]
    end

    PDF --> SPLIT_ALL
    SPLIT_ALL --> PAGE_ONE_FLOW & PAGE_TWO_FLOW & PAGE_THREE_FLOW & PDOT & PN
    PAGE_ONE_FLOW --> PROGRESS_FIRST
    PAGE_TWO_FLOW --> PROGRESS_SECOND
    PN --> PROGN
    PROGRESS_FIRST & PROGRESS_SECOND & PROGN --> ALL_DONE --> SET_READY
```

### Concurrency Model

The p-limit pool is sized based on the API key pool. Each key has a rate limit from the provider. The pool distributes pages across keys round-robin, staying within each key's concurrency limit.

Pages that fail (Gemini error, network timeout) are retried up to 3 times with exponential backoff before the page is marked as failed. A single failed page doesn't abort the entire document — the other pages continue. The final `page_index` will have gaps for failed pages, which the retrieval layer handles gracefully.

### Why Not Stream to the Client

The blocking stage doesn't stream partial results to the client. The client polls the file status endpoint and sees `progress_current` increment. Streaming partial results would complicate the client significantly for minimal UX benefit — the user sees "Summarizing 15/50 pages" either way.

---

## Image Extraction Pipeline

```mermaid
flowchart TD
    PAGE_PDF["Single-page PDF\n(from pdf-lib split)"]

    subgraph RASTER["Raster image extraction (pdfjs)"]
        GET_OPS["getOperatorList\nfor page"]
        FIND_IMG["Find paintImageXObject\noperations"]
        GET_RES["getResources\nfor page"]
        DECODE["Decode image data\nvia JIMP"]
        SIZE_CHECK{Width ≥100px\nAND Height ≥100px?}
        KEEP["Keep image\n(meaningful content)"]
        DISCARD["Discard image\n(decorative)"]
    end

    subgraph VECTOR_FALLBACK["Vector chart fallback"]
        HAS_VECTOR{hasVectorCharts\nfrom Gemini?}
        NO_RASTER{No raster images\nextracted?}
        RENDER_CANVAS["pdfjs canvas\nrender page to PNG\n(full page render)"]
        USE_PNG["Use rendered PNG\nas supplementary image"]
        SKIP["Skip fallback\n(raster images available)"]
    end

    subgraph UPLOAD["S3 upload"]
        STORAGE_IMAGE_KEY["Hierarchical image key\norganized by user, thread, file, page, and image index"]
        PRESIGN["Generate presigned URL\n7-day TTL"]
        STORE_META["Store S3 key + URL\nin metadata JSONB"]
    end

    subgraph DELIVERY["Image delivery"]
        API_ENDPOINT["Page Image Redirect Endpoint"]
        REDIRECT["302 redirect\nto presigned S3 URL"]
        DIRECT_URL["Direct presigned URL\n(preferred, from metadata)"]
    end

    PAGE_PDF --> GET_OPS --> FIND_IMG --> GET_RES --> DECODE --> SIZE_CHECK
    SIZE_CHECK -->|"Yes"| KEEP --> STORAGE_IMAGE_KEY
    SIZE_CHECK -->|"No"| DISCARD

    KEEP --> HAS_VECTOR
    HAS_VECTOR -->|"Yes"| NO_RASTER
    NO_RASTER -->|"Yes"| RENDER_CANVAS --> USE_PNG --> STORAGE_IMAGE_KEY
    NO_RASTER -->|"No"| SKIP
    HAS_VECTOR -->|"No"| SKIP

    STORAGE_IMAGE_KEY --> PRESIGN --> STORE_META

    STORE_META --> DIRECT_URL
    API_ENDPOINT --> REDIRECT
```

### Image Description Schema

Each extracted image gets a structured description from Gemini's `generateObject` call. The `ImageDescription` type includes:

| Field | Type | Description |
|-------|------|-------------|
| `index` | number | Position of this image on the page (0-indexed) |
| `s3Key` | string | S3 object key for the image |
| `description` | string | Gemini's natural language description |
| `type` | string | Inferred type: `chart`, `table`, `photo`, `diagram`, `other` |
| `width` | number | Pixel width |
| `height` | number | Pixel height |

These descriptions are stored in the `metadata` JSONB column of `page_index` and are used by the visual grounding pipeline when a user asks about charts or images.

### Web Grounding Images

Images that come from web search (Gemini's web grounding feature) are handled differently. They appear naturally in the agent's markdown response as standard image URLs. They don't go through the S3 pipeline. The distinction matters: document images need S3 storage and presigned URLs; web grounding images are already publicly accessible URLs.

### Presigned URL Delivery

Presigned URLs have a 7-day TTL. The page image redirect endpoint exists as a fallback for clients that don't have the presigned URL cached. It generates a fresh presigned URL and returns a 302 redirect. Clients that store the presigned URL from the `metadata` JSONB can skip the API call entirely.

---

## S3 Storage Layout

All objects live in the `safeagent` bucket. The key structure encodes ownership (userId, threadId, fileId) so cleanup can target a specific thread without scanning the entire bucket.

```mermaid
graph TD
    BUCKET["safeagent bucket"]

    subgraph USER["userId/"]
        subgraph THREAD["threadId/"]
            subgraph FILE["fileId/"]
                ORIG["original.{ext}\n(original uploaded file)"]
                subgraph PAGES["pages/"]
                    PAGE_FIRST_IMG["page-1.pdf"]
                    PAGE_SECOND_IMG["page-2.pdf"]
                    PAGEN["page-N.pdf"]
                end
                IMG_FIRST["page-1-img-0.png\n(first image on page 1)"]
                IMG_SECOND["page-1-img-1.jpg\n(second image on page 1)"]
                IMG_THIRD["page-2-img-0.png\n(first image on page 2)"]
            end
        end
    end

    BUCKET --> USER --> THREAD --> FILE
    FILE --> ORIG
    FILE --> PAGES
    FILE --> IMG_FIRST & IMG_SECOND & IMG_THIRD
```

### Key Patterns

| Object | Key pattern |
|--------|------------|
| Original file | Hierarchical key organized by user, thread, and file identity, with an original-object suffix |
| Per-page PDF | Hierarchical key organized by user, thread, file identity, and page number in a page-artifacts segment |
| Extracted image | Hierarchical key organized by user, thread, file identity, page number, and image index |

Page numbers are 1-indexed. Image indices are 0-indexed (matching the `ImageDescription.index` field).

### Bun.S3Client

The system uses `Bun.S3Client` for all S3 operations. This is a zero-dependency S3 client built into the Bun runtime. The alternative (`@aws-sdk/client-s3`) adds approximately 15MB to the bundle. Since the server runs on Bun, the built-in client is always preferred.

`Bun.S3Client` supports presigned URL generation, multipart upload, and all standard S3 operations. It works with any S3-compatible endpoint, including MinIO.

---

## page_index Table Design

The `page_index` table stores all retrieval artifacts for document pages. It's managed by Drizzle ORM.

### Schema

| Column | Type | Description |
|--------|------|-------------|
| `id` | uuid (PK) | Row identifier |
| `file_id` | uuid (FK → file_uploads) | Parent file |
| `thread_id` | text | Thread that owns this file (TEXT to support `'__global__'` sentinel for cross-conversation documents) |
| `user_id` | text | User that owns this file |
| `page_number` | integer | 1-indexed page number |
| `summary` | text | LLM-generated page summary (blocking stage) |
| `summary_embedding` | vector(EMBEDDING_DIMS) | Dense embedding of `summary` (dimension from configuration constants) |
| `raw_text` | text (nullable) | Background extracted text; null before enrichment |
| `raw_embedding` | vector(EMBEDDING_DIMS) (nullable) | Dense embedding of `raw_text`; null before enrichment (dimension from configuration constants) |
| `raw_tsvector` | tsvector (nullable, GENERATED) | Generated from `raw_text` for full-text search |
| `metadata` | jsonb | `{ images: ImageDescription[], s3PageKey: string }` |
| `created_at` | timestamptz | Row creation time |

> **Note**: The CROSS_CONV_RAG task (document 09) adds a non-nullable `scope` column (`TEXT`, values `'thread'` or `'global'`) to this table. The column is not listed above because it does not exist until CROSS_CONV_RAG is implemented. See document 09 for the full cross-conversation retrieval design.

### Indexes

| Index | Type | Purpose |
|-------|------|---------|
| `page_index_embedding_idx` | HNSW (pgvector) | Approximate nearest-neighbor vector search |
| `page_index_search_vector_idx` | GIN | Full-text search via tsvector |
| `page_index_file_id_idx` | B-tree | Scoped retrieval by file |
| `page_index_user_id_idx` | B-tree | User-scoped queries |

### Why No page_data Column

The original page PDF bytes are stored in S3, not in this table. Storing page bytes in Postgres would cause TOAST bloat. A 200-page document with 50KB per page PDF is 10MB per document. At 10,000 documents, that's 100GB of TOAST data in a single table. S3 is the right place for binary objects.

### Partitioning at Scale

At 10M users, `page_index` is the highest-growth table (each document produces one row per page). Range partitioning by `user_id` hash keeps individual partition indexes (HNSW, GIN, B-tree) manageable and distributes write I/O. All queries include `user_id` in their WHERE clause, enabling automatic partition pruning. See file 01 (Horizontal Scaling Model) and file 15 (Capacity Planning) for the broader partitioning strategy.

### Hybrid Retrieval

At query time, the retrieval layer runs two searches in parallel:
1. HNSW vector search on `summary_embedding` (semantic similarity)
2. GIN full-text search on `raw_tsvector` (keyword matching)

The results are merged with Reciprocal Rank Fusion. Semantic retrieval uses `summary_embedding`, while keyword retrieval uses `raw_tsvector` when available.

---

## File Metadata Tables

### file_uploads

Tracks every uploaded file from receipt through cleanup.

| Column | Type | Description |
|--------|------|-------------|
| `file_id` | uuid (PK) | Stable identifier returned to client |
| `file_name` | text | Original filename from the upload |
| `mime_type` | text | Detected MIME type (from magic bytes) |
| `byte_size` | integer | File size in bytes |
| `user_id` | text | Owning user |
| `thread_id` | text | Thread this file was uploaded in (text to align with page_index sentinel support) |
| `status` | enum | `uploading`, `summarizing`, `ready`, `enriching`, `enriched`, `failed`, `deleted` |
| `mode` | enum | `direct`, `indexed`, `rag` |
| `page_count` | integer | Total pages (null for non-PDF/DOCX) |
| `s3_key` | text | S3 key for the original file |
| `progress_current` | integer | Pages processed so far |
| `progress_total` | integer | Total pages to process |
| `expires_at` | timestamptz | When this file will be auto-deleted |
| `deleted_at` | timestamptz? | When soft-delete occurred (null while active) |
| `error_message` | text? | Error details when status is `failed` (null on success) |
| `created_at` | timestamptz | Upload timestamp |

### user_storage_quotas

Tracks per-user storage consumption.

| Column | Type | Description |
|--------|------|-------------|
| `user_id` | text (PK) | User identifier |
| `used_bytes` | bigint | Current storage consumption |
| `max_bytes` | bigint | Quota limit (default: 100MB = 104,857,600 bytes) |
| `updated_at` | timestamptz | Last update time |

### Quota Safety

When a file is deleted, the quota is released with `GREATEST`. This prevents the `used_bytes` column from going negative due to race conditions or double-deletes. The `GREATEST` guard is applied at the SQL level, not in application code.

---

## Progress Tracking

```mermaid
sequenceDiagram
    participant C as Client
    participant API as Elysia API
    participant PG as PostgreSQL

    C->>API: Upload file (multipart)
    API-->>C: {fileId, status: "uploading"}

    Note over API: Blocking stage begins

    loop Every page processed
        API->>PG: UPDATE progress_current = N
    end

    C->>API: Poll file status
    API->>PG: SELECT status, progress_current, progress_total
    PG-->>API: {status: "summarizing", progress_current: 15, progress_total: 50}
    API-->>C: {status: "summarizing", progress: "15/50"}

    Note over API: All pages done

    API->>PG: UPDATE status = "ready"

    C->>API: Poll file status
    API-->>C: {status: "ready"}
```

The client polls the file status endpoint. The response includes:
- `status`: current state machine value
- `progress_current`: pages processed
- `progress_total`: total pages
- A human-readable `message` field: `"Summarizing 15/50 pages"`

Polling interval is left to the client. A 2-second interval is reasonable for most use cases. The endpoint is cheap: a single indexed Postgres read.

---

## Cleanup

```mermaid
flowchart TD
    subgraph PER_FILE["cleanupFile(fileId, userId, deps)\ncalled on per-file deletion or TTL expiry"]
        direction TB
        CHECK_FILE["Require BOTH fileId AND userId"]
        STORAGE_DELETE_FILE_OBJECTS["Delete S3 objects\nfor this file only"]
        PAGE_DEL_FILE["DELETE FROM page_index\nWHERE file_id = ? AND user_id = ?"]
        CHUNK_DEL_FILE["DELETE PgVector chunks\nWHERE file_id = ? AND user_id = ?"]
        META_DEL_FILE["UPDATE file_uploads\nSET status='deleted',\ndeleted_at=NOW()\nWHERE file_id = ? AND user_id = ?"]
        QUOTA_REL_FILE["Release storage quota\ndecrBy(userId, fileSize)"]

        CHECK_FILE --> STORAGE_DELETE_FILE_OBJECTS --> PAGE_DEL_FILE --> CHUNK_DEL_FILE --> META_DEL_FILE --> QUOTA_REL_FILE
    end

    subgraph PER_THREAD["cleanupThread(threadId, userId, deps)\ncalled on thread deletion"]
        direction TB
        LIST_FILES["SELECT file_id FROM file_uploads\nWHERE thread_id = ? AND user_id = ?\nAND thread_id != '__global__'\nAND status != 'deleted'"]
        LOOP["For each file: cleanupFile(fileId, userId, deps)"]

        LIST_FILES --> LOOP
    end

    subgraph TTL["TTL-based automatic cleanup"]
        SCHEDULED["Trigger.dev scheduled task\nRuns daily"]
        FIND_EXPIRED["SELECT * FROM file_uploads\nWHERE expires_at < NOW()\nAND status != 'deleted'"]
        CALL_CLEANUP["Call cleanupFile\nfor each expired file"]
    end

    LOOP --> PER_FILE
    SCHEDULED --> FIND_EXPIRED --> CALL_CLEANUP --> PER_FILE
```

### Why Both threadId and userId

Requiring both parameters is a defense-in-depth measure. A bug that passes the wrong `threadId` can't accidentally delete another user's files because the `userId` check will fail. A bug that passes the wrong `userId` can't delete the thread because the `threadId` check will fail. Both must be correct for any deletion to proceed.

### Valkey Lock

The optional distributed lock prevents two concurrent cleanup calls for the same thread from racing. Without the lock, two simultaneous calls could both read the same `used_bytes` value and both subtract the file size, resulting in a double-release. The lock is optional because the `GREATEST` guard already prevents negative values — the lock is belt-and-suspenders for correctness.

### TTL-Based Cleanup

Files have an `expires_at` timestamp set at upload time. The Trigger.dev scheduled task runs daily, finds all expired files, and calls `cleanupFile` for each. This handles the case where a user never explicitly deletes their files.

---

## Cross-References

| Document | Relationship |
|----------|-------------|
| **System Architecture** ([03 — System Architecture](./03-architecture.md)) | Defines the end-to-end upload and retrieval architecture that this document's processing pipeline implements in detail. |
| **Foundation** ([04 — Foundation](./04-foundation.md)) | Supplies processing thresholds, provider settings, and environment values used by routing, summarization, and storage flows. |
| **RAG & Retrieval** ([09 — Retrieval & Evidence](./09-retrieval.md)) | Consumes `page_index` summaries and enrichment outputs produced here as primary retrieval inputs. |
| **Server Implementation** ([12 — Server Implementation](./12-server.md)) | Hosts upload/status/image endpoints and invokes this processing pipeline from route handlers. |
| **Infrastructure** ([15 — Infrastructure](./15-infrastructure.md)) | Provides Compose services (Postgres, MinIO, Valkey, Trigger.dev, LibreOffice) required to run blocking and background stages. |

---

## Task Specifications

> **SPIKE_RAG_DEPS** — canonical task specification is in [04 — Foundation](./04-foundation.md#task-spike_rag_deps-rag--multimodal-dependencies-spike). This document's pipeline tasks depend on its output.

---

### Task DOC_PIPELINE: Document Processing Pipeline

**What to do**: Implement the full blocking stage pipeline. pdf-lib splits PDFs into per-page PDFs. pdfjs extracts raster images per page with the 100×100px size filter. Gemini `generateObject` summarizes each page with the `{ summary, imageDescriptions[], hasVectorCharts }` schema. p-limit controls concurrency based on the key pool tier. Vector chart fallback renders pages to PNG when `hasVectorCharts` is true and no raster images were found. Summaries are embedded and stored in `page_index`. Progress is tracked via `progress_current`.

**Depends on**: SPIKE_RAG_DEPS, KEY_POOL, CONFIG_DEFAULTS

**Acceptance Criteria**:
- Large-PDF benchmark (e.g., ~200 pages when the upload size limit is increased for internal load testing) processes in under 15 seconds with a 10-key pool at elevated `summarizationConcurrencyPerKey`
- Each page produces exactly one `page_index` row with populated `summary` and `summary_embedding`
- Images smaller than 100×100px are not extracted
- `hasVectorCharts=true` pages without raster images trigger the PNG render fallback
- `progress_current` increments after each page completes
- `status` transitions to `'ready'` only after all pages complete
- A single failed page doesn't abort the entire document

**QA Scenarios**:
- Large-PDF benchmark with mixed text and charts processes completely
- Page with only vector graphics (no raster images) triggers fallback render
- Page with decorative icons (all <100×100px) produces no extracted images
- Gemini API error on page 50 retries 3 times, marks page as failed, continues with page 51
- Progress endpoint shows accurate count during processing

---

### Task FILE_STORAGE: File Storage

**What to do**: Implement the S3 storage layer using `Bun.S3Client`. Implement the key naming scheme for original files, per-page PDFs, and extracted images. Implement presigned URL generation with 7-day TTL. Implement the page image redirect endpoint that generates a fresh presigned URL and returns a 302 redirect. Implement the `cleanupThread` function that lists and batch-deletes all S3 objects under a thread prefix.

**Depends on**: DOCKER_COMPOSE, CONFIG_DEFAULTS

**Acceptance Criteria**:
- All S3 keys follow a hierarchical structure organized by user, thread, and file identity
- Presigned URLs expire after 7 days
- The image delivery endpoint returns 302 to a valid presigned URL
- `cleanupThread` deletes all objects under the thread prefix in a single batch operation
- `Bun.S3Client` is used exclusively (no `@aws-sdk` dependency)

**QA Scenarios**:
- Upload a file, verify original stored at correct S3 key
- Request image via API endpoint, verify 302 redirect to presigned URL
- Presigned URL is accessible without authentication
- Call `cleanupThread`, verify all objects under the thread prefix are deleted
- Call `cleanupThread` twice for the same thread, verify no error on second call (idempotent)

---

### Task UPLOAD_PIPELINE: Upload Pipeline

**What to do**: Implement the complete upload pipeline from validation through routing. Magic bytes validation for all supported types. Size enforcement (≤5MB per file, ≤5 files per turn). Quota check against `user_storage_quotas`. DOCX detection and routing to LibreOffice conversion. Page count detection for PDFs. Mode assignment (`direct`, `indexed`, `rag`). `file_uploads` record creation. Quota increment. FileRegistry cache invalidation in Valkey.

**Depends on**: FILE_STORAGE, STORAGE_WRAPPER, VALKEY_CACHE, DOCKER_COMPOSE

**Acceptance Criteria**:
- Magic bytes check rejects files with mismatched content and extension
- `.doc` files are rejected with a clear error message
- Files over 5MB are rejected before any S3 write
- Quota exceeded returns a clear error before any S3 write
- DOCX files are routed to LibreOffice conversion before page count detection
- Mode is set correctly for all type/size combinations
- `file_uploads` record is created with correct initial status

**QA Scenarios**:
- Upload a `.pdf` file that is actually a ZIP archive → rejected at magic bytes check
- Upload a `.doc` file → rejected with message to save as `.docx`
- Upload a 6MB file → rejected before S3 write
- Upload files that would exceed 100MB quota → rejected with quota error
- Upload a 3-page DOCX → converted to PDF, mode set to `direct`
- Upload a 50-page DOCX → converted to PDF, mode set to `indexed`
- Upload a TXT file with 5K tokens → mode set to `direct`
- Upload a TXT file with 20K tokens → mode set to `rag`

---

> **UPLOAD_ENDPOINT** — canonical task specification is in [12 — Server Implementation](./12-server.md#task-upload_endpoint-upload-endpoint). The server route wires UPLOAD_PIPELINE to the HTTP layer.

---

> **DOCKER_COMPOSE** — canonical task specification is in [15 — Infrastructure](./15-infrastructure.md#task-docker_compose-docker-compose-infrastructure). That spec includes LibreOffice headless, MinIO bucket init, and all compose profiles.

---

## External References

- [pdf-lib documentation](https://pdf-lib.js.org)
- [unpdf documentation](https://github.com/unjs/unpdf)
- [JIMP documentation](https://jimp-dev.github.io/jimp)
- [Drizzle ORM documentation](https://orm.drizzle.team/docs)
- [Bun S3 documentation](https://bun.sh/docs/api/s3)

---

*Previous: [07 — Memory & Intelligence](./07-memory.md) | Next: [09 — Retrieval & Evidence](./09-retrieval.md)*
