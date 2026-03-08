# 11 — Streaming & Transport

> **Scope**: SSE streaming pipeline, custom SSE event protocol boundary, trace-step event family, verbosity-level filtering, CTA tool streaming, and the `@safeagent/client` SDK.
>
> **Tasks**: SSE_STREAMING (SSE Streaming Layer), CTA_STREAMING (CTA Streaming), CLIENT_SDK (Client SDK)

---

## Table of Contents

- [Architecture Overview](#architecture-overview)
- [SSE Streaming Layer (SSE_STREAMING)](#sse-streaming-layer-ssestreaming)
- [Stream Format Boundary](#stream-format-boundary)
- [Session Metadata Delivery](#session-metadata-delivery)
- [Trace-Step Events](#trace-step-events)
- [Verbosity Levels](#verbosity-levels)
- [CTA Streaming (CTA_STREAMING)](#cta-streaming-ctastreaming)
- [Client SDK (CLIENT_SDK)](#client-sdk-clientsdk)
- [SSE Event Type Reference](#sse-event-type-reference)
- [Cross-References](#cross-references)
- [Task Specifications](#task-specifications)

---

## Architecture Overview

The streaming system keeps `@openai/agents` `RunStreamEvent` items internal to the safeagent library. At the HTTP boundary, `createStreamHandler` iterates `Runner.run()` stream output and translates framework events into a custom named-event SSE protocol (`session-meta`, `text-delta`, `trace-step`, `cta`, `citation`, `location`, `tripwire`, `done`, `error`) designed for `@safeagent/client` and other SSE consumers. The `trace-step` events provide real-time pipeline visibility for developer debugging and are only emitted when the verbosity level is `full`.

```mermaid
graph TB
    subgraph AGENT_LAYER["Agent Layer (RunStreamEvent internal format)"]
        AGENT["Runner.run(agent)\nAsyncIterable of RunStreamEvent"]
        GUARD_PROC["Output Guardrail\nProcessor"]
        CTA_PROC["CTA Stream\nProcessor"]
        LOC_PROC["Location Stream\nProcessor"]
        TRACE_COLLECT["Trace-Step\nCollector"]
        TUI["TUI Client\n(direct library stream, no SSE)"]
    end

    subgraph HTTP_BOUNDARY["HTTP Boundary (Elysia)"]
        HANDLER["createStreamHandler"]
        META_INJECT["Session Meta Injector\n{ traceId, threadId, agentId }"]
        VERBOSITY_FILTER["Verbosity Filter\n(standard: user-facing only\nfull: all events)"]
        KEEPALIVE["Keepalive Timer\n(comment: ping)"]
        ERR_CATCH["TripWire / Error\nCatcher"]
    end

    subgraph SSE_WIRE["SSE Wire Protocol (named events)"]
        EVT_META["event: session-meta\ndata: { traceId, threadId }"]
        EVT_TEXT["event: text-delta\ndata: { delta }"]
        EVT_TRACE["event: trace-step\ndata: { step, data, latencyMs }"]
        EVT_CTA["event: cta\ndata: { cta: [...] }"]
        EVT_CIT["event: citation\ndata: { citation }"]
        EVT_LOC["event: location\ndata: { location: { name, type, lat, lng, images, context? } }"]
        EVT_TRIP["event: tripwire\ndata: { conceptId, reason, fallbackMessage }"]
        EVT_DONE["event: done\ndata: {}"]
        EVT_ERR["event: error\ndata: { code, message }"]
    end

    subgraph CLIENT_SDK["@safeagent/client"]
        PARSER["SSE Parser"]
        TYPED["Typed Event\nDispatcher"]
        QUEUE["Offline Message\nQueue"]
        RECONNECT["Reconnection\nManager"]
    end

    AGENT --> GUARD_PROC --> CTA_PROC --> LOC_PROC --> TRACE_COLLECT
    TRACE_COLLECT --> TUI
    TRACE_COLLECT --> HANDLER
    HANDLER --> VERBOSITY_FILTER
    ERR_CATCH -.->|"catches TripWire"| EVT_TRIP
    VERBOSITY_FILTER --> EVT_META & EVT_TEXT & EVT_CTA & EVT_CIT & EVT_LOC & EVT_TRIP & EVT_DONE & EVT_ERR
    VERBOSITY_FILTER -->|"full only"| EVT_TRACE
    KEEPALIVE -.-> EVT_META
    EVT_META & EVT_TEXT & EVT_TRACE & EVT_CTA & EVT_CIT & EVT_LOC & EVT_TRIP & EVT_DONE & EVT_ERR --> PARSER
    PARSER --> TYPED --> QUEUE
    QUEUE --> RECONNECT
```

---

## SSE Streaming Layer (SSE_STREAMING)

### createStreamHandler

`createStreamHandler` is an Elysia route handler factory exported by the library. It wires together every concern that touches the HTTP streaming path: context injection, `Runner.run()` invocation, `RunStreamEvent` → SSE translation, keepalive, and error handling. The server registers it as a route and configures auth middleware; the library owns all the streaming logic inside. The library also exports the underlying framework-agnostic stream processing primitives for non-Elysia consumers (e.g., tests, TUI).

The handler calls `Runner.run(agent, input, { stream: true })` which returns `AsyncIterable<RunStreamEvent>`. Event types include `raw_model_stream_event` (text deltas), `run_item_stream_event` (tool calls, messages, handoffs), and `agent_updated_stream_event` (agent switches via handoff). The handler maps these to the eight SSE event types. Framework guardrail exceptions (`InputGuardrailTripwireTriggered`, `OutputGuardrailTripwireTriggered`) are caught at the boundary and emitted as `tripwire` SSE events.

The factory accepts an `errorMessageMap` parameter — a plain object keyed by error code string, where values are either a static message string or a function `(metadata) => string` for dynamic messages. When the handler catches an error and emits an `error` SSE event, it looks up the error's code in this map to populate the `message` field. If the code is not in the map, a generic fallback message is used. The server passes its error message map (defined in [12 — Server Implementation](./12-server.md)) when constructing the handler. This is the DI mechanism that keeps the library language-agnostic while letting the server control user-facing tone.

```mermaid
flowchart TB
    REQ["Chat Streaming Request"]

    subgraph AUTH["Auth Middleware (runs before handler)"]
        JWT["JWT Verification"]
        CTX["requestContext\n{ userId, threadId }"]
    end

    subgraph HANDLER["createStreamHandler internals"]
        direction TB
        CTX_PROV["contextProvider()\nDI hook — injects file context\nfor direct-mode files"]
        META_FIRST["Emit session-meta event\n(first event on stream,\nBEFORE any guardrail/agent work)"]
        AGENT_CALL["Runner.run(agent, input, { stream: true })"]
        PROC_CHAIN["RunStreamEvent iteration\nguardrails → CTA → location"]
        SSE_WRITE["Elysia sse() generator"]
        KEEPALIVE_T["Keepalive interval\n(comment: ping every N seconds)"]
        ERR_BOUND["Error boundary\ncatch TripWire → emit tripwire event\ncatch unknown → emit error event"]
    end

    RESP["SSE Response\nContent-Type: text/event-stream"]

    REQ --> JWT --> CTX
    CTX --> CTX_PROV
    CTX_PROV --> META_FIRST
    META_FIRST --> AGENT_CALL
    AGENT_CALL --> PROC_CHAIN
    PROC_CHAIN --> SSE_WRITE
    KEEPALIVE_T -.->|"parallel"| SSE_WRITE
    ERR_BOUND -.->|"wraps entire chain"| SSE_WRITE
    SSE_WRITE --> RESP
```

### contextProvider DI Hook

The server passes a `contextProvider` function when constructing the handler. The function signature is:

**`contextProvider: (ctx: { userId: string, threadId: string, messageId: string }) => Promise<AdditionalContext | null>`**

Where `AdditionalContext` is `{ role: 'system', content: string | MessageContentPart[] }[]` — an array of message objects to prepend to the agent's message list. Returning `null` or an empty array means no additional context is injected.

Its primary use is injecting file context in direct mode, where the user has uploaded files that should be visible to the agent without going through the full RAG pipeline. The server implements the function to fetch file content from storage, read from a local cache, or return nothing based on whether the request includes file references.

The hook is dependency-injected so the library stays agnostic about how the server resolves files. The agent never knows the difference.

### userId Flow

The JWT auth lifecycle hook extracts `userId` from the bearer token and attaches it to Elysia context via `derive` or `resolve`. `createStreamHandler` reads `userId` from context and passes it into the agent call via `requestContext` along with the `threadId` from the request body. The agent and all its tools receive `userId` through this channel. Nothing in the library reads `userId` from anywhere else.

```mermaid
sequenceDiagram
    participant CLIENT
    participant ELYSIA
    participant AUTH_MIDDLEWARE
    participant CREATE_STREAM_HANDLER
    participant AGENT

    Client->>Elysia: Chat streaming request\nAuthorization: Bearer token
    Elysia->>AuthMiddleware: verify JWT
    AuthMiddleware->>Elysia: resolve ctx.userId
    Elysia->>createStreamHandler: handle(ctx)
    createStreamHandler->>createStreamHandler: read userId from ctx, threadId from body
    createStreamHandler->>Agent: stream(messages, { userId, threadId })
    AGENT-->>createStreamHandler: AsyncIterable<RunStreamEvent>
    CREATE_STREAM_HANDLER-->>Client: SSE stream
```

### Keepalive

Proxies and load balancers close idle connections after a timeout, typically 30 to 60 seconds. Long-running agent calls can easily exceed this. The handler starts a keepalive interval that writes SSE comment lines (`": ping"`) on a fixed cadence. Comment lines are valid SSE but carry no data, so the client ignores them. The interval is cleared when the stream ends or errors.

### Error Handling

Two error types get special treatment at the HTTP boundary:

- **TripWire**: thrown by the guardrail system when a p0 input violation is detected, or when a p0 output violation is detected in development mode. The handler catches it, stops the stream, and emits a `tripwire` SSE event containing `conceptId`, `reason`, and a fallback message. `reason` is the user-facing message, while `conceptId` is the canonical guardrail concept identifier. In production mode, output p0 violations do NOT throw TripWire — they suppress remaining chunks and inject a fallback as a normal `text-delta` event (see [10 — Guardrails & Safety](./10-guardrails.md) for details).
- **Unknown errors**: anything else becomes a generic `error` SSE event. The stream closes cleanly rather than dropping the connection mid-response.

The output sliding window used by output guardrails also runs eld language detection on accumulated text. When LanguageGuardConfig.supportedLanguages is configured and eld detects output drift into an unsupported language, the output guardrail returns p0 and the stream follows the same tripwire or suppress-and-fallback path as any other output guardrail violation. This acts as the final safety net for cases where input passed earlier checks but generated output drifts into an unsupported language.

```mermaid
flowchart LR
    CHUNK["text-delta chunk"] --> WINDOW["Sliding window text"]
    WINDOW --> LANG_SCAN["eld language scan"]
    LANG_SCAN --> CHECK{"Supported language?"}
    CHECK -->|PASS| EMIT["Emit chunk"]
    CHECK -->|FAIL| PRIORITY_BLOCK["Output guardrail p0"]
    PRIORITY_BLOCK --> DEV["Development: tripwire event"]
    PRIORITY_BLOCK --> PROD["Production: suppress remaining output and inject fallback"]
```

```mermaid
flowchart LR
    STREAM["Agent stream\n(running)"]
    TRIP{"TripWire\nthrown?"}
    UNKNOWN{"Other error\nthrown?"}
    EMIT_TRIP["emit: tripwire event\n{ conceptId, reason, fallbackMessage }"]
    EMIT_ERR["emit: error event\n{ code, message }"]
    CLOSE["close SSE stream"]

    STREAM --> TRIP
    TRIP -->|yes| EMIT_TRIP --> CLOSE
    TRIP -->|no| UNKNOWN
    UNKNOWN -->|yes| EMIT_ERR --> CLOSE
    UNKNOWN -->|no| STREAM
```

---

## Stream Format Boundary

This is the most important invariant in the entire streaming system.

**`RunStreamEvent` items (produced by `Runner.run()`) are used inside the library. The HTTP transport emits a custom named-event SSE protocol for external clients.**

The TUI consumes the library stream directly (no SSE). Tests against the agent layer use `RunStreamEvent` items directly. The HTTP handler maps those events to custom SSE events for network transport. This means:

- Adding a new stream processor updates one internal stream path, while external clients still receive stable named SSE events.
- The TUI and the HTTP path share the same processor chain, so behavior is identical.
- Internal `RunStreamEvent` items and external SSE event names are intentionally separated by the transport boundary.

```mermaid
flowchart LR
    subgraph INTERNAL["Library internals (RunStreamEvent format throughout)"]
        AGENT["Runner.run(agent)\nAsyncIterable of RunStreamEvent"]
        GP["Guardrail\nProcessor"]
        CP["CTA\nProcessor"]
        TUI_C["TUI\nconsumer"]
    end

    subgraph WIRE["SSE Wire"]
        SSE["Custom named SSE events\nsession-meta | text-delta | trace-step | cta | citation | location | tripwire | done | error"]
    end

    subgraph CLIENTS["External Clients"]
        WEB["Web / Mobile\n@safeagent/client"]
    end

    AGENT --> GP --> CP
    CP --> TUI_C
    CP --> SSE
    SSE --> WEB
```

The wire protocol is custom SSE named events, not any framework-specific data format. `@safeagent/client` parses these events directly.

Location enrichment events are emitted during the live stream, never batched at the end. A single response can emit multiple `location` events, typically one per detected place. The underlying `search_locations` tool-call and tool-result chunks are suppressed from the outbound SSE stream using a `createLocationStreamProcessor` that mirrors the CTA suppression pattern, and only the clean `location` event payload is emitted to clients. When no image provider is configured, each `location` event still includes `lat` and `lng`, and `images` is emitted as an empty array.

---

## Session Metadata Delivery

Every stream starts with a `session-meta` event before any text delta arrives. This event carries the identifiers the client needs to correlate the stream with backend records.

```mermaid
sequenceDiagram
    participant HANDLER as createStreamHandler
    participant CLIENT as @safeagent/client

    Handler->>Client: event: session-meta\ndata: { traceId, threadId, agentId }
    Note over Client: client stores traceId for feedback submission
    Handler->>Client: event: text-delta\ndata: { delta: "Hello" }
    Handler->>Client: event: text-delta\ndata: { delta: " there" }
    Handler->>Client: event: cta\ndata: { cta: [...] }
    Handler->>Client: event: done
```

The `traceId` is a server-generated UUID created before the agent run begins and passed to Langfuse as the trace identifier. The `threadId` is the conversation thread ID for the conversation. The `agentId` is optional and identifies which agent handled the request when the server runs multiple agents.

The client SDK stores `traceId` automatically so feedback submissions (thumbs up/down) can attach it without the application code tracking it manually.

### trace_owners Table Schema

The `trace_owners` Postgres table (managed by Drizzle ORM) enables feedback ownership verification. `createStreamHandler` inserts a row before emitting the first SSE event; the feedback endpoint queries it to confirm the requesting user owns the trace. The Drizzle schema for this table is owned by SSE_STREAMING — it is defined and migrated as part of that task.

| Column | Type | Constraints |
|--------|------|-------------|
| traceId | text | PRIMARY KEY |
| userId | text | NOT NULL |
| createdAt | timestamp | NOT NULL, DEFAULT now() |

No additional indexes beyond the primary key — lookups are always by `traceId`.

---

## Trace-Step Events

Trace-step events provide real-time visibility into the engine pipeline for developer debugging and tracing. They expose what the engine is doing internally — intent classification, memory recall, guardrail evaluation, retrieval, tool execution, and context assembly — as structured SSE events. These events are emitted only when the verbosity level is `full`, ensuring end users never see internal pipeline details.

This is the primary mechanism for the frontend trace and debug experience described in [18 — Frontend SDK](./18-frontend-sdk.md). When verbosity is `standard` (the default), only user-facing events are emitted. When verbosity is `full`, trace-step events are interleaved with regular events to show pipeline activity in real time.

### Trace-Step Event Envelope

All trace-step events share a common envelope: `type: "trace-step"`, a `step` discriminator field, a step-specific `data` payload, a `latencyMs` timing field for visualization, and a `timestamp` for ordering.

```mermaid
classDiagram
    class SSE_TRACE_STEP_EVENT {
        type: "trace-step"
        step: TraceStepType
        data: TraceStepData
        latencyMs: number
        timestamp: number
    }
```

### Step Types

| Step | Emitted When | Data Summary |
|------|-------------|-------------|
| `intent-detected` | Embedding router and LLM validator complete | Intent name, confidence score, detection source (embedding or LLM override), whether LLM agreed with or corrected the embedding classification |
| `memory-recall` | Long-term memory recall returns | Number of facts recalled, top relevance score, temporal distribution summary |
| `guardrail-input` | Input guardrails aggregate | Aggregate verdict (pass, block, or flag), individual guardrail result summaries, highest severity encountered |
| `guardrail-output` | Output guardrail scan runs | Scan result (pass or flag), concept identifier if flagged, sliding window position |
| `retrieval` | RAG or document search completes | Source name, result count, retrieval mode (hybrid, direct, or external) |
| `tool-call-start` | Agent invokes a tool | Tool name, tool category |
| `tool-call-end` | Tool execution completes | Tool name, success or error indicator, result summary |
| `context-budget` | Context assembly completes | Used tokens, total budget, breakdown by layer (thread history, user short-term, long-term recall, source content) |
| `source-fetch` | Source router dispatches or completes a source fetch | Source name, status (started, completed, or failed), result count on completion |
| `rewrite` | Query rewrite applied | Strategy name (HyDE, EntityExtraction, DenseKeywords), trigger reason |

### Emission Ordering

Trace-step events are interleaved with regular events in the order they occur in the pipeline. The engine always produces trace-step data internally (it needs this data for Langfuse tracing regardless). The verbosity filter at the HTTP boundary decides whether to emit them to the SSE stream.

```mermaid
sequenceDiagram
    participant ENGINE as Engine Pipeline
    participant HANDLER as createStreamHandler
    participant CLIENT as Client (full verbosity)

    ENGINE->>HANDLER: session-meta
    HANDLER->>CLIENT: event: session-meta

    ENGINE->>HANDLER: trace-step (intent-detected)
    HANDLER->>CLIENT: event: trace-step

    ENGINE->>HANDLER: trace-step (guardrail-input)
    HANDLER->>CLIENT: event: trace-step

    ENGINE->>HANDLER: trace-step (memory-recall)
    HANDLER->>CLIENT: event: trace-step

    ENGINE->>HANDLER: trace-step (context-budget)
    HANDLER->>CLIENT: event: trace-step

    ENGINE->>HANDLER: trace-step (source-fetch started)
    HANDLER->>CLIENT: event: trace-step

    ENGINE->>HANDLER: trace-step (retrieval)
    HANDLER->>CLIENT: event: trace-step

    ENGINE->>HANDLER: text-delta
    HANDLER->>CLIENT: event: text-delta

    ENGINE->>HANDLER: text-delta
    HANDLER->>CLIENT: event: text-delta

    ENGINE->>HANDLER: trace-step (tool-call-start)
    HANDLER->>CLIENT: event: trace-step

    ENGINE->>HANDLER: trace-step (tool-call-end)
    HANDLER->>CLIENT: event: trace-step

    ENGINE->>HANDLER: trace-step (guardrail-output)
    HANDLER->>CLIENT: event: trace-step

    ENGINE->>HANDLER: cta
    HANDLER->>CLIENT: event: cta

    ENGINE->>HANDLER: done
    HANDLER->>CLIENT: event: done
```

### Relationship to Langfuse

Trace-step events are a real-time subset of the data that Langfuse captures asynchronously. They share the same `traceId` (from `session-meta`). The key differences:

| Aspect | Trace-Step Events | Langfuse Traces |
|--------|------------------|----------------|
| Delivery | Real-time SSE during stream | Async batch export after stream completes |
| Audience | Developers debugging via frontend UI | Analytics and observability dashboards |
| Detail level | Summary per pipeline step | Full span tree with nested children and metadata |
| Persistence | Ephemeral (client memory only) | Permanent (Langfuse storage) |
| Correlation | Same `traceId` from `session-meta` | Same `traceId` |

A developer can watch trace-step events in real-time during a conversation, then switch to Langfuse for deep post-hoc analysis of the same request using the shared `traceId`. See [14 — Observability](./14-observability.md) for the full Langfuse tracing architecture.

### Trace-Step Collector

The trace-step collector is a stream processor that sits after the location processor in the chain. It intercepts pipeline milestone signals — timing data, classification results, guardrail verdicts — and packages them as `trace-step` data events. Unlike the CTA and location processors which suppress tool-call events, the trace-step collector emits additional events alongside the normal stream without suppressing anything.

The collector is always active regardless of verbosity. The verbosity filter at the HTTP boundary decides whether trace-step events reach the SSE wire. The TUI consumes trace-step data directly from the library stream with no filtering.

```mermaid
flowchart LR
    AGENT["Agent\nstream"]
    GP["Guardrail\nProcessor"]
    CP["CTA\nProcessor"]
    LP["Location\nProcessor"]
    TC["Trace-Step\nCollector"]
    TUI_OUT["TUI\n(all events)"]
    HTTP_OUT["HTTP Handler\n(verbosity filter)"]

    AGENT --> GP --> CP --> LP --> TC
    TC --> TUI_OUT
    TC --> HTTP_OUT
```

---

## Verbosity Levels

The chat streaming endpoint accepts a `verbosity` parameter that controls which events are emitted on the SSE stream. This parameter is passed from the server route to `createStreamHandler`.

| Level | Events Emitted | Target Audience |
|-------|---------------|----------------|
| `standard` (default) | `session-meta`, `text-delta`, `cta`, `citation`, `location`, `tripwire`, `done`, `error` | End users, production applications |
| `full` | All `standard` events plus `trace-step` events interleaved at their natural pipeline positions | Developers debugging pipeline behavior via [18 — Frontend SDK](./18-frontend-sdk.md) verbosity toggle |

### Verbosity Filtering

```mermaid
flowchart LR
    HANDLER["createStreamHandler"]
    VERBOSITY{"verbosity\nparameter"}
    STANDARD_FILTER["Emit user-facing\nevents only\n(8 event types)"]
    FULL_FILTER["Emit all events\nincluding trace-step\n(9 event types)"]
    SSE["SSE Response"]

    HANDLER --> VERBOSITY
    VERBOSITY -->|standard| STANDARD_FILTER --> SSE
    VERBOSITY -->|full| FULL_FILTER --> SSE
```

The filter is applied at the HTTP boundary, not inside the engine. The engine always produces trace-step data (it needs it for Langfuse regardless). `createStreamHandler` decides whether to emit trace-step events to the SSE stream based on the verbosity parameter.

This design means:

- Zero performance cost when verbosity is `standard` — trace-step events are simply not written to the response
- The engine pipeline is identical regardless of verbosity — no conditional behavior in agent logic
- The TUI can consume trace-step data directly from the library stream (no verbosity filtering needed since TUI is not HTTP)
- Frontend applications control verbosity per-request, allowing a developer to toggle between standard and full modes without server restart

### Verbosity and Security

Trace-step data may contain internal pipeline details (intent names, guardrail concept IDs, tool names, token counts). When verbosity is `full`, the server trusts that the requesting client is a developer who should see this information. The server should enforce that `full` verbosity requires an authenticated user with developer-level permissions. This is an authorization concern owned by the server, not the library — the library simply respects the `verbosity` parameter it receives.

---

## CTA Streaming (CTA_STREAMING)

### What CTAs Are

Call-to-action suggestions are structured UI hints the agent can emit alongside its text response. They're not hardcoded responses — the LLM decides when to suggest them based on conversation context. A CTA might be a deeplink to a relevant screen, a callback action the app handles, or a dismiss button.

Each CTA has: `id`, `label`, `action` (one of `deeplink`, `callback`, or `dismiss`), optional `url`, and optional `icon`. A response carries at most three CTAs.

### createCTATool

The server defines a CTA catalog in its config: a list of available CTAs with their IDs, labels, and actions. The library's `createCTATool` factory takes that catalog and returns a framework-compatible tool the agent can call. The tool's input schema is derived from the catalog so the LLM can only suggest CTAs that actually exist.

The server owns the catalog. The library owns the tool mechanics. This separation means adding a new CTA to the server config automatically makes it available to the agent without touching library code.

### CTA Tool Flow

The key behavior: tool-call events are suppressed from the SSE stream. The client never sees a raw tool call. Instead, the stream processor intercepts the tool invocation, extracts the CTA data, and emits a clean `cta` event.

```mermaid
flowchart TB
    LLM["LLM decides to\nsuggest CTAs"]
    TOOL_CALL["suggest_cta tool call\n(RunStreamEvent)"]
    PROC["createCTAStreamProcessor\n(stream processor)"]

    subgraph DECISION["Processor decision"]
        IS_CTA{"is suggest_cta\ntool call?"}
        SUPPRESS["suppress tool-call\nevent from stream"]
        EMIT_CTA["emit cta event\nevent: cta\ndata: {\"cta\":[...]}"]
        PASS["pass chunk\nthrough unchanged"]
    end

    CLIENT["Client receives\nclean cta event"]

    LLM --> TOOL_CALL --> PROC
    PROC --> IS_CTA
    IS_CTA -->|yes| SUPPRESS --> EMIT_CTA --> CLIENT
    IS_CTA -->|no| PASS --> CLIENT
```

### CTA Event Format

The CTA event is emitted as a custom named SSE event. The handler writes `event: cta` and a JSON payload in the `data:` line with a `cta` key.

```mermaid
flowchart LR
    subgraph PAYLOAD["CTA Event Payload"]
        direction TB
        ID["id: string"]
        LABEL["label: string"]
        ACTION["action: 'deeplink' | 'callback' | 'dismiss'"]
        URL["url?: string"]
        ICON["icon?: string"]
    end

    subgraph CONSTRAINTS["Constraints"]
        MAX["max 3 per response"]
        CATALOG["must exist in server catalog"]
        LLM_DEC["LLM-decided (not hardcoded)"]
    end

    PAYLOAD --> CONSTRAINTS_CHECK["validated against catalog\nat tool call time"]
    CONSTRAINTS --> CONSTRAINTS_CHECK
```

### CTA Stream Processor Placement

The CTA processor sits after the guardrail processor in the chain. This ordering matters: guardrails run first, so a p0 violation aborts the stream before any CTA events are emitted. A response that gets blocked never leaks CTAs.

```mermaid
flowchart LR
    AGENT["Agent\nstream()"]
    GP["Guardrail\nProcessor"]
    CP["CTA\nProcessor"]
    OUT["Output\n(TUI or HTTP handler)"]

    AGENT -->|"all chunks"| GP
    GP -->|"safe chunks"| CP
    CP -->|"text + cta events\n(no raw tool calls)"| OUT
```

---

## Client SDK (CLIENT_SDK)

### Design Principles

`@safeagent/client` is a framework-agnostic TypeScript package with zero runtime dependencies. It works in browsers and React Native. It doesn't import React, Vue, or any UI framework. Applications wrap it in whatever state management they use.

Zero dependencies is a hard constraint. Every feature — SSE parsing, reconnection, offline queuing, file uploads — is implemented from scratch using platform APIs.

For TypeScript consumers who want server-inferred route types, Eden Treaty (`@elysiajs/eden`) is available as an optional alternative client path. The primary SDK remains `@safeagent/client` because it is zero-dependency and framework-agnostic, while Eden Treaty can provide auto-inferred types from `type App = typeof app` on Elysia servers.

### SSE Parsing and Typed Events

The client opens an SSE connection using the Fetch API with a streaming response body. It parses the event stream incrementally, dispatching typed events to registered callbacks as they arrive.

```mermaid
flowchart TB
    subgraph CONNECTION["Connection Layer"]
        FETCH["fetch() with\nReadableStream body"]
        READER["ReadableStreamDefaultReader"]
        DECODER["TextDecoder\n(UTF-8 incremental)"]
        PARSER["SSE line parser\n(event / data / id fields)"]
    end

    subgraph DISPATCH["Event Dispatch"]
        ROUTE{"event type?"}
        ON_META["onSessionMeta(event)"]
        ON_TEXT["onTextDelta(event)"]
        ON_CTA["onCTA(event)"]
        ON_CIT["onCitation(event)"]
        ON_LOC["onLocation(event)"]
        ON_TRIP["onTripwire(event)"]
        ON_DONE["onDone(event)"]
        ON_ERR["onError(event)"]
    end

    FETCH --> READER --> DECODER --> PARSER --> ROUTE
    ROUTE -->|session-meta| ON_META
    ROUTE -->|text-delta| ON_TEXT
    ROUTE -->|cta| ON_CTA
    ROUTE -->|citation| ON_CIT
    ROUTE -->|location| ON_LOC
    ROUTE -->|tripwire| ON_TRIP
    ROUTE -->|done| ON_DONE
    ROUTE -->|error| ON_ERR
```

### Auto-Reconnection

The client reconnects automatically when the connection drops. It uses exponential backoff with jitter to avoid thundering herd on server restarts. The last SSE `id` field is sent as `Last-Event-ID` on reconnect so the server can resume from where it left off (if the server supports it).

```mermaid
stateDiagram-v2
    [*] --> CONNECTING
    CONNECTING --> CONNECTED: stream opened
    CONNECTING --> BACKOFF: connection failed
    CONNECTED --> BACKOFF: connection dropped
    CONNECTED --> CLOSED: done event received
    CONNECTED --> CLOSED: client.close() called
    BACKOFF --> CONNECTING: backoff timer expires
    BACKOFF --> CLOSED: max retries exceeded
    CLOSED --> [*]

    note right of BACKOFF
        delay = min(base * 2^attempt + jitter, maxDelay)
    end note
```

### Offline Message Queue

When the network is unavailable, outbound messages (chat sends, feedback submissions) are held in an in-memory queue rather than failing immediately. When connectivity returns and the connection re-establishes, the queue drains in FIFO order.

The queue is bounded. When it reaches capacity, an `overflow` event fires and the oldest message is dropped. Applications can listen for overflow events to show a warning.

```mermaid
flowchart TB
    subgraph NETWORK_CHECK["Network State"]
        ONLINE{"navigator.onLine\nor fetch probe?"}
    end

    subgraph QUEUE["Offline Queue"]
        ENQUEUE["enqueue message"]
        BOUNDED["bounded capacity\n(configurable)"]
        OVERFLOW["emit overflow event\ndrop oldest"]
        DRAIN["drain FIFO\non reconnect"]
    end

    subgraph SEND["Send Path"]
        DIRECT["send immediately"]
    end

    MSG["sendMessage() called"]

    MSG --> ONLINE
    ONLINE -->|online| DIRECT
    ONLINE -->|offline| ENQUEUE
    ENQUEUE --> BOUNDED
    BOUNDED -->|at capacity| OVERFLOW
    BOUNDED -->|has space| QUEUE_WAIT["wait in queue"]
    QUEUE_WAIT --> DRAIN
    DRAIN --> DIRECT
```

### File Upload Support

The client exposes a file upload method that posts files to the server's upload endpoint with the JWT bearer token attached. Upload progress is reported via a callback. The returned file reference (an ID or URL) can then be included in a subsequent chat message.

### Feedback Submission

The client stores the `traceId` from the most recent `session-meta` event. When the application calls `submitFeedback`, the client attaches the stored `traceId` automatically. The application doesn't need to track trace IDs.

```mermaid
sequenceDiagram
    participant APP
    participant CLIENT as @safeagent/client
    participant SERVER

    SERVER-->>Client: session-meta { traceId: "abc123" }
    Note over Client: stores traceId internally
    App->>Client: submitFeedback({ score: 1 })
    Client->>Server: Submit feedback\n{ traceId, score: 1 }
    SERVER-->>Client: 200 OK
```

### JWT Auth

Every request the client makes — chat, upload, feedback — includes `Authorization: Bearer <token>`. The token is provided at construction time or via a refresh callback. When a refresh callback is provided, the client calls it before each request to get a fresh token, supporting short-lived JWTs without requiring the application to manage token lifecycle.

---

## SSE Event Type Reference

All event types are shared between the server (emitter) and the client (consumer). They're defined in safeagent and imported by `@safeagent/client` at compile time.

| Event Type | Payload | Description |
|---|---|---|
| `session-meta` | `SSESessionMetaEvent` | First event on every stream. Carries trace and thread IDs. |
| `text-delta` | `SSETextDeltaEvent` | Incremental text chunk from the LLM. |
| `trace-step` | `SSETraceStepEvent` | Pipeline step visibility event. Only emitted when verbosity is `full`. Carries step type, step-specific data, and timing. |
| `cta` | `SSECTAEvent` | Call-to-action suggestions from the CTA tool. |
| `citation` | `SSECitationEvent` | Source citation metadata for grounded output. |
| `location` | `SSELocationEvent` | Location enrichment data for a place mentioned by the agent. Emitted progressively as places are geocoded and enriched. Client renders map pins and inline image galleries. |
| `tripwire` | `SSETripwireEvent` | Guardrail p0 violation. Stream ends after this. |
| `done` | `SSEDoneEvent` | Stream completed normally. |
| `error` | `SSEErrorEvent` | Unexpected error. Stream ends after this. |

For `location`, `type` is one of `city`, `neighborhood`, `restaurant`, `landmark`, `region`, or `country`. `ImageResult` has the shape `{ url: string, thumbnail: string, attribution?: string, source?: string }`.

### SSESessionMetaEvent

```mermaid
classDiagram
    class SSE_SESSION_META_EVENT {
        type: "session-meta"
        traceId: string
        threadId: string
        agentId?: string
    }
```

### SSETextDeltaEvent

```mermaid
classDiagram
    class SSE_TEXT_DELTA_EVENT {
        type: "text-delta"
        delta: string
    }
```

### SSETraceStepEvent

```mermaid
classDiagram
    class SSE_TRACE_STEP_EVENT {
        type: "trace-step"
        step: TraceStepType
        data: TraceStepData
        latencyMs: number
        timestamp: number
    }

    class TRACE_STEP_TYPE {
        <<enumeration>>
        intent-detected
        memory-recall
        guardrail-input
        guardrail-output
        retrieval
        tool-call-start
        tool-call-end
        context-budget
        source-fetch
        rewrite
    }

    SSE_TRACE_STEP_EVENT --> TRACE_STEP_TYPE
```

`TraceStepData` is a discriminated union keyed by `step`. Each step type carries a step-specific payload described in the [Trace-Step Events](#trace-step-events) section.

### SSECTAEvent

```mermaid
classDiagram
    class SSECTA_EVENT {
        type: "cta"
        cta: CTA[]
    }
    class CTA {
        id: string
        label: string
        action: "deeplink" | "callback" | "dismiss"
        url?: string
        icon?: string
    }
    SSECTA_EVENT --> CTA
```

### SSECitationEvent

```mermaid
classDiagram
    class SSE_CITATION_EVENT {
        type: "citation"
        citation: Citation
    }
    class CITATION {
        source: string
        fileId?: string
        page?: number
        quote: string
        scope?: "thread" | "global"
        images?: ImageRef[]
    }
    class IMAGE_REF {
        url: string
        description: string
        imageIndex: number
        apiPath: string
    }
    CITATION --> IMAGE_REF
    SSE_CITATION_EVENT --> CITATION
```

### SSELocationEvent

```mermaid
classDiagram
    class SSE_LOCATION_EVENT {
        type: "location"
        location: LocationResult
    }
    SSE_LOCATION_EVENT --> LOCATION_RESULT
```

### SSETripwireEvent

```mermaid
classDiagram
    class SSE_TRIPWIRE_EVENT {
        type: "tripwire"
        conceptId: string
        reason: string
        fallbackMessage: string
    }
```

`conceptId` carries the canonical guardrail concept identifier. `reason` is a human-readable explanation mapped from that concept for client display.

### SSEDoneEvent

```mermaid
classDiagram
    class SSE_DONE_EVENT {
        type: "done"
    }
```

### SSEErrorEvent

```mermaid
classDiagram
    class SSE_ERROR_EVENT {
        type: "error"
        code: string
        message: string
    }
```

---

## Cross-References

| Document | Relationship |
|----------|-------------|
| **Requirements** ([01 — Requirements & Constraints](./01-requirements.md)) | Defines platform requirements, transport expectations, and frontend SDK requirements that this SSE boundary and client SDK must satisfy. |
| **Foundation** ([04 — Foundation](./04-foundation.md)) | Defines the SSE event type contracts (including SSETraceStepEvent and TraceStepType) consumed by this transport layer and the client SDK. |
| **Agents** ([06 — Agents & Orchestration](./06-agents.md)) | Defines orchestrator and processor-chain behavior, including location tool orchestration, that produces the `RunStreamEvent` stream consumed by this SSE transport layer. |
| **Guardrails & Safety** ([10 — Guardrails & Safety](./10-guardrails.md)) | Defines language drift detection in output sliding windows and p0 enforcement behavior used by this transport layer. |
| **Server Implementation** ([12 — Server Implementation](./12-server.md)) | Owns the Elysia route wiring, HTTP boundary, and verbosity parameter where this document's `createStreamHandler` and SSE event protocol are applied. |
| **Observability** ([14 — Observability](./14-observability.md)) | Trace-step events share `traceId` with Langfuse traces, providing real-time visibility that complements async post-hoc analysis. |
| **Frontend SDK** ([18 — Frontend SDK](./18-frontend-sdk.md)) | Consumes `@safeagent/client` events (including `trace-step`) and builds React hooks, web components, and React Native components on top of this transport layer. |
| **Demos** ([19 — Demos](./19-demos.md)) | Demo applications that exercise the full SSE protocol including trace-step events and verbosity toggle. |

---

## Task Specifications

---

### Task SSE_STREAMING: SSE Streaming Layer

**What to do**:

Build `createStreamHandler`, the Elysia handler factory that turns `Runner.run()` output into a well-formed SSE response. This includes:

- Accepting an `errorMessageMap` parameter from the server — a plain object mapping error codes to user-facing messages (string or function). Used by the error boundary to populate the `message` field in `error` SSE events.
- Extracting `userId` from the JWT auth context and combining it with `threadId` from the request body to form `requestContext`
- Inserting a `{ traceId, userId }` row into the `trace_owners` Postgres table before emitting the first SSE event (enables feedback ownership verification — see FEEDBACK_ENDPOINT)
- Calling the `contextProvider` DI hook to inject file context before invoking the agent
- Calling `Runner.run(agent, input, { stream: true })` to get `AsyncIterable<RunStreamEvent>`
- Accepting a `verbosity` parameter (`standard` or `full`) from the route handler. When `standard`, only user-facing events are emitted. When `full`, `trace-step` events are also emitted alongside user-facing events.
- Running the trace-step collector in the processor chain to capture pipeline milestone data (intent classification, guardrail verdicts, memory recall, retrieval results, tool execution, context budget) as `trace-step` data events
- Iterating `RunStreamEvent` items and mapping them to the nine SSE event types:
  - `raw_model_stream_event` (text delta) → `text-delta` SSE event
  - `run_item_stream_event` with tool call for `suggest_cta` → `cta` SSE event (tool chunks suppressed)
  - `run_item_stream_event` with tool call for `search_locations` → `location` SSE event (tool chunks suppressed)
  - Citation metadata from tool results → `citation` SSE event
  - Pipeline milestones (from trace-step collector) → `trace-step` SSE events (only when verbosity is `full`)
  - `agent_updated_stream_event` → logged for tracing (handoff routing)
  - Stream start → `session-meta` SSE event (first event, carries traceId + threadId)
  - Stream end → `done` SSE event
- Writing the stream through Elysia's `sse()` generator response path
- Running a keepalive interval that writes SSE comment lines to prevent proxy timeouts
- Catching `InputGuardrailTripwireTriggered` / `OutputGuardrailTripwireTriggered` exceptions (from framework guardrails) and emitting a `tripwire` SSE event with `conceptId`, reason, and fallback message
- Catching all other errors, looking up the error code in `errorMessageMap` to get the user-facing message, and emitting an `error` SSE event with `{ code, message }`
- Closing the stream cleanly in all exit paths

**Depends on**:

- AGENT_FACTORY (Agent Factory — `Runner.run()` must return `AsyncIterable<RunStreamEvent>`)
- INPUT_GUARD, OUTPUT_GUARD (Guardrail Processors — processor chain must be composable)
- SCAFFOLD_LIB (library scaffolding — `@openai/agents` + `@openai/agents-extensions` bridge packages installed)

**Acceptance Criteria**:

- The chat streaming endpoint returns `Content-Type: text/event-stream`
- The first event on every stream is `session-meta` with non-empty `traceId` and `threadId`
- A `trace_owners` row is inserted before the first SSE event (traceId + userId)
- Text deltas arrive as `text-delta` events in order
- A `done` event closes the stream after normal completion
- A p0 guardrail violation produces a `tripwire` event and no further text deltas
- An unexpected thrown error produces an `error` event with a `message` from the `errorMessageMap` and closes the stream
- Error codes not in the map produce a generic fallback message
- Keepalive comment lines appear at the configured interval during long responses
- The TUI can consume the same processor chain output without going through the HTTP handler
- When verbosity is `full`, `trace-step` events are interleaved with user-facing events at their natural pipeline positions
- When verbosity is `standard`, zero `trace-step` events appear in the stream
- `userId` extracted from JWT is present in the agent's `requestContext` for every call

**QA Scenarios**:
- Normal chat response → `session-meta` → N × `text-delta` → `done`
- p0 guardrail on input → `session-meta` → `tripwire` (no text deltas)
- p0 guardrail mid-stream → `session-meta` → some `text-delta` → `tripwire` → stream closes
- Agent throws unexpected error → `session-meta` → `error` → stream closes
- Response takes 45 seconds → keepalive comments appear and connection stays open
- `contextProvider` returns file content → agent receives file content prepended to messages
- Missing or invalid JWT → 401 before stream opens (auth middleware, not handler)
- Verbosity `full` → `trace-step` events interleaved with user-facing events showing pipeline activity
- Verbosity `standard` → zero `trace-step` events in stream, identical to pre-trace behavior
- Verbosity `full` with p0 guardrail → `trace-step` for `guardrail-input` appears, then `tripwire`, then stream closes

---

### Task CTA_STREAMING: CTA Streaming

**What to do**:

Build the CTA tool and stream processor pair:

- The `createCTATool` factory — takes a `CTACatalog` (array of `CTA` definitions) and returns a framework-compatible tool. The tool's input schema is derived from the catalog so the LLM can only reference valid CTA IDs. The tool's execute function returns the selected CTAs.
- The `createCTAStreamProcessor` factory — returns a stream processor that intercepts `suggest_cta` tool-call events, suppresses them from the output stream, and emits a `cta` data event in their place. All other chunks pass through unchanged.
- Define the `CTA` type: `{ id, label, action: 'deeplink' | 'callback' | 'dismiss', url?, icon? }`
- Enforce the max-3-CTAs constraint at the tool schema level
- Document the catalog configuration shape for server authors

**Depends on**:

- AGENT_FACTORY (Agent Factory — agent must accept custom tools)
- CORE_TYPES (tool definition and stream processor type interfaces)

**Acceptance Criteria**:

- The `createCTATool` factory returns a valid framework-compatible tool
- The LLM cannot suggest a CTA ID that isn't in the catalog (schema validation rejects it)
- A response with CTAs produces exactly one `cta` SSE event
- No `tool-call` or `tool-result` events for `suggest_cta` appear in the SSE stream
- A response without CTAs produces no `cta` event
- The processor passes all non-CTA chunks through without modification
- Guardrail processor runs before CTA processor — a blocked response emits no CTA events
- The catalog is defined in server config; the library has no hardcoded CTAs

**QA Scenarios**:
- LLM calls `suggest_cta` with 2 valid CTAs → `cta` event has 2 items and no raw tool-call event appears
- LLM calls `suggest_cta` with an invalid CTA ID → tool schema validation error and no `cta` event
- LLM calls `suggest_cta` with 4 CTAs → schema rejects the call (max-3 constraint)
- Response has no CTA tool call → no `cta` event appears in the stream
- p0 guardrail fires before CTA tool call → `tripwire` event with no `cta` event
- Empty catalog passed to `createCTATool` → tool is created but the LLM has no valid CTA options to suggest

---

### Task CLIENT_SDK: Client SDK

**What to do**:

Build `@safeagent/client`, a zero-dependency TypeScript package:

- SSE connection management using Fetch API streaming
- Incremental SSE line parser (handles chunked delivery, multi-line data fields)
- Typed event dispatch: register callbacks for each of the nine event types (including `trace-step`)
- `trace-step` event dispatch with step-type discrimination: the `onTraceStep` callback receives a discriminated union payload typed by `step` field, allowing consumers to handle each pipeline step differently
- Auto-reconnection with exponential backoff and jitter; respect `Last-Event-ID`
- Offline message queue: enabled via `offline: { enabled: true, maxQueueSize: N }` config; queues `sendMessage` calls when offline; drains FIFO on reconnect; emits a client-local `overflow` callback (NOT an SSE event) when queue is full and drops oldest
- File upload: multipart POST with progress callback; returns file reference
- Feedback submission: `submitFeedback` — attaches stored `traceId` automatically
- JWT auth: accept token at construction or via async refresh callback
- Full TypeScript types for all events, config, and public methods
- No runtime dependencies — zero production dependencies

**Depends on**:

- SSE_STREAMING (SSE Streaming Layer — server must emit the event types the client parses)
- CTA_STREAMING (CTA Streaming — `cta` event type must be defined before client handles it)
- CORE_TYPES (shared SSE event type definitions)

**Acceptance Criteria**:

- Package installs with zero production dependencies
- All nine event types are handled with typed callbacks (including `trace-step`)
- `onSessionMeta` fires before `onTextDelta` on every stream
- `traceId` from `session-meta` is stored and attached to `submitFeedback` automatically
- Connection drops trigger reconnection with backoff; `Last-Event-ID` is sent
- `sendMessage` while offline enqueues the message; it sends after reconnect
- Queue overflow fires `onOverflow` and drops the oldest message
- File upload reports progress via callback and returns a file reference
- JWT refresh callback is called before each request
- Works in browser (Bun-compatible web APIs only in the main bundle)
- TypeScript strict mode passes with no errors

**QA Scenarios**:
- Normal stream → `onSessionMeta` → N × `onTextDelta` → `onDone`
- Server sends `tripwire` → `onTripwire` fires with `conceptId`, reason, and `fallbackMessage`
- Server sends `cta` → `onCTA` fires with a typed CTA array
- Server sends `citation` → `onCitation` fires with typed source citation data
- Server sends `location` → `onLocation` fires with typed place, coordinates, and image metadata
- Server sends `trace-step` (verbosity full) → `onTraceStep` fires with discriminated union payload; step type and step-specific data are fully typed
- Connection drops mid-stream → client reconnects with backoff and sends `Last-Event-ID`
- Max retries exceeded → `onError` fires and no further reconnect attempts are made
- `sendMessage` while offline → message is queued and sent after reconnect
- Queue reaches `maxQueueSize` → `onOverflow` fires and oldest message is dropped
- `submitFeedback` after stream → POST includes `traceId` from the last `session-meta`
- JWT refresh callback provided → callback runs before each request and fresh token is used
- File upload → progress callback fires and a file reference is returned on completion

---

*Previous: [10 — Guardrails & Safety](./10-guardrails.md) | Next: [12 — Server Implementation](./12-server.md)*

