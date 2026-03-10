# Agents & Orchestration

> **Scope**: Agent creation factory wrapping `@openai/agents`, SDK bridge helper for Gemini, orchestrator with handoff-based sub-agent routing, multi-intent handling, live synthesis streaming, provider fallback, and context shaping behaviors that make responses feel natural under real conversational conditions.
>
> **Existing tasks**: AGENT_FACTORY (Agent Factory), PROVIDER_FALLBACK (Provider Fallback), AGENT_ROUTER (Agent Router)
> **New components**: Orchestrator pattern using framework handoffs, sub-agent framework, live synthesis

---

## Table of Contents

- [Architecture Overview](#architecture-overview)
- [Agent Factory (Agent Creation Factory)](#agent-factory-agent-creation-factory)
- [Orchestrator Agent Pattern](#orchestrator-agent-pattern)
- [Dynamic Fan-Out & Map-Reduce Orchestration](#dynamic-fan-out--map-reduce-orchestration)
- [Context Assembly, Budgeting, and Response Calibration](#context-assembly-budgeting-and-response-calibration)
- [Live Synthesis Streaming](#live-synthesis-streaming)
- [Dependent Intent Handling](#dependent-intent-handling)
- [Tool Registry](#tool-registry)
- [Agent Router (Query Classification)](#agent-router-query-classification)
- [Scaling: Queue-Based Execution](#scaling-queue-based-execution)
- [Provider Fallback](#provider-fallback)
- [Cross-References](#cross-references)
- [Task Specifications](#task-specifications)
- [External References](#external-references)

---

## Architecture Overview

The agent layer has three tiers: the **factory** (wraps `@openai/agents` framework agent class with safe defaults), the **orchestrator** (supervises multi-intent execution via the framework's handoff mechanism), and **sub-agents** (handle individual intents with scoped tools).

```mermaid
graph TB
    subgraph FACTORY_LAYER["Agent Factory (Library)"]
        CREATE_AGENT["Agent creation factory\nwraps framework agent class\n+ SDK model bridge"]
        CREATE_ORCHESTRATOR["Orchestrator factory\nframework agent class with handoffs to sub-agents"]
        CREATE_SUB_AGENT["Sub-agent factory\nscoped framework agent class per intent"]
    end

    subgraph RUNTIME_LAYER["Runtime Execution (framework execution method)"]
        direction TB
        ORCHESTRATOR_AGENT["Orchestrator Agent<br/>(Supervisor)"]

        subgraph SUB_AGENT_GROUP["Sub-Agents (via Handoffs)"]
            SUB_AGENT_REFUND["Sub-Agent: refund<br/>Tools: ragflow, searchDocument"]
            SUB_AGENT_PRODUCT["Sub-Agent: product<br/>Tools: searchDocument"]
            SUB_AGENT_MEMORY["Sub-Agent: memory<br/>Tools: memory_recall"]
        end

        LIVE_SYNTHESIS["Live Synthesis<br/>Streaming"]
    end

    FACTORY_LAYER --> RUNTIME_LAYER
    ORCHESTRATOR_AGENT -->|handoff| SUB_AGENT_REFUND & SUB_AGENT_PRODUCT & SUB_AGENT_MEMORY
    SUB_AGENT_REFUND & SUB_AGENT_PRODUCT & SUB_AGENT_MEMORY -->|results stream| LIVE_SYNTHESIS
    LIVE_SYNTHESIS -->|SSE| CLIENT_APP["Client"]
```

---

## Agent Factory (Agent Creation Factory)

The core factory wraps `@openai/agents` framework agent class with safeagent-specific configuration. It uses the SDK model bridge from `@openai/agents-extensions` to connect Gemini to the framework's provider-agnostic model interface. It is the foundation for all agents in the system.

```mermaid
flowchart TB
    subgraph SAFE_AGENT_CONFIG["SafeAgentConfig"]
        CONFIG_ID["id + name"]
        CONFIG_INSTRUCTIONS["instructions (prompt)"]
        CONFIG_MODEL["model (PRIMARY_PROVIDER\nvia SDK model bridge)"]
        CONFIG_TOOLS["tools"]
        CONFIG_GUARDRAILS["inputGuardrails +\noutputGuardrails"]
        CONFIG_MEMORY["memory config"]
        CONFIG_THINKING["thinkingLevel"]
        CONFIG_GROUNDING["groundingMode"]
        CONFIG_GUARD_MODE["guardMode"]
        CONFIG_OUTPUT_SCHEMA["outputSchema (optional)"]
        CONFIG_HANDOFFS["handoffs (optional)"]
    end

    subgraph AGENT_FACTORY["agent creation factory"]
        FACTORY_RESOLVE_MODEL["SDK model bridge"]
        FACTORY_BUILD_GUARDRAILS["Build guardrails\n(guardrail arrays)"]
        FACTORY_WIRE_MEMORY["Wire memory\n(Conversation store)"]
        FACTORY_WIRE_TOOLS["Wire tools\n(MCP + custom)"]
        FACTORY_CONSTRUCT_AGENT["Construct framework agent class instance"]
    end

    subgraph FACTORY_OUTPUT["Agent + runtime"]
        OUTPUT_RUNNER["framework execution method\nAsyncIterable of RunStreamEvent"]
        OUTPUT_STRUCTURED["structured output generation (when outputSchema set)"]
    end

    SAFE_AGENT_CONFIG --> AGENT_FACTORY --> FACTORY_OUTPUT
```

### Agent Modes

The agent operates in distinct modes depending on use case:

```mermaid
stateDiagram-v2
    [*] --> CHAT_MODE: Default
    [*] --> GROUNDING_MODE: groundingMode set
    [*] --> TOOLS_MODE: tools provided
    [*] --> STRUCTURED_MODE: outputSchema set

    CHAT_MODE: Chat Mode
    CHAT_MODE: Pure conversation
    CHAT_MODE: Custom tools available

    GROUNDING_MODE: Grounding Mode
    GROUNDING_MODE: Google Search grounding
    GROUNDING_MODE: NO custom tools (mutual exclusion)

    TOOLS_MODE: Tools Mode
    TOOLS_MODE: Custom tools + MCP tools
    TOOLS_MODE: NO grounding (mutual exclusion)

    STRUCTURED_MODE: Structured Output Mode
    STRUCTURED_MODE: structured output generation with Zod v4 schema
    STRUCTURED_MODE: Returns typed JSON
```

**Critical constraint**: Gemini grounding and custom tools are mutually exclusive in the same agent call (AI SDK limitation). The factory creates separate agent instances for grounding and tool-use modes.

### Parallel Grounding

For queries that benefit from both grounding and tool responses, the library provides a parallel execution pattern:

```mermaid
sequenceDiagram
    participant USER
    participant STREAM_HANDLER as Stream Handler
    participant GROUNDING_AGENT
    participant TOOLS_AGENT
    participant RESPONSE_MERGER as Response Merger

    USER->>STREAM_HANDLER: "What's the weather and what does my doc say about climate?"

    par Parallel Execution
        STREAM_HANDLER->>GROUNDING_AGENT: Grounding query (Google Search)
        STREAM_HANDLER->>TOOLS_AGENT: Tool query (document search)
    end

    GROUNDING_AGENT-->>RESPONSE_MERGER: Grounding result + citations
    TOOLS_AGENT-->>RESPONSE_MERGER: Tool result + citations
    RESPONSE_MERGER-->>STREAM_HANDLER: Merged response
    STREAM_HANDLER->>USER: Unified SSE stream
```

## Orchestrator Agent Pattern

The orchestrator is a **supervisor agent** that handles multi-intent messages by routing to sub-agents via the framework handoff mechanism. Handoffs are implemented as tool calls under the hood: the model makes an agent transfer call to route control.

```mermaid
flowchart TB
    subgraph CLASSIFIED_INPUT["Classified Input"]
        USER_MESSAGE["User message"]
        DETECTED_INTENTS["Detected intents<br/>(from conversation pipeline)"]
        REWRITTEN_QUERIES["Rewritten queries<br/>(from conversation pipeline)"]
    end

    INTENT_DECISION{How many<br/>intents?}

    subgraph SINGLE_INTENT_PATH["Single Intent Path"]
        ORCHESTRATOR_DIRECT["Orchestrator works<br/>directly with tools"]
    end

    subgraph MULTI_INTENT_PATH["Multi-Intent Path"]
        SPLIT_SUB_QUERIES["Split into sub-queries"]

        subgraph PARALLEL_HANDOFFS["Sub-Agents (via Handoffs)"]
            HANDOFF_REFUND["agent transfer call<br/>Query: 'refund policy'"]
            HANDOFF_COMPANY["agent transfer call<br/>Query: 'CEO name'"]
        end

        MULTI_INTENT_SYNTHESIS["Live Synthesis<br/>(stream as handoffs complete)"]
    end

    CLASSIFIED_INPUT --> INTENT_DECISION
    INTENT_DECISION -->|1 intent| SINGLE_INTENT_PATH
    INTENT_DECISION -->|2+ intents| MULTI_INTENT_PATH
    SPLIT_SUB_QUERIES --> HANDOFF_REFUND & HANDOFF_COMPANY
    HANDOFF_REFUND & HANDOFF_COMPANY --> MULTI_INTENT_SYNTHESIS
```

### Sub-Agent Lifecycle

Each sub-agent is an independent framework agent class instance with its own tools, scoped to a single intent. The framework's handoff mechanism transfers control from orchestrator to sub-agent. Handoff input filtering scopes conversation history so the sub-agent focuses on its assigned intent. Handoff callbacks log routing decisions.

```mermaid
sequenceDiagram
    participant RUNNER as Framework execution method
    participant ORCHESTRATOR
    participant SUB_AGENT as Sub-Agent (via framework handoff mechanism)
    participant AGENT_TOOLS as Agent Tools
    participant DATA_SOURCES as Data Sources

    RUNNER->>ORCHESTRATOR: Start with user message
    ORCHESTRATOR->>ORCHESTRATOR: Detect intents, decide routing
    ORCHESTRATOR->>SUB_AGENT: agent transfer call handoff
    Note over SUB_AGENT: handoff context scoping limits to intent
    Note over SUB_AGENT: handoff callback logs routing decision

    SUB_AGENT->>SUB_AGENT: Analyze query

    opt Query Rewrite Needed
        SUB_AGENT->>AGENT_TOOLS: queryRewrite(strategy, query)
        AGENT_TOOLS-->>SUB_AGENT: Rewritten query
    end

    SUB_AGENT->>AGENT_TOOLS: searchDocument / ragflowSearch / groundingSearch
    AGENT_TOOLS->>DATA_SOURCES: Parallel source queries
    DATA_SOURCES-->>AGENT_TOOLS: Results with priority weights

    SUB_AGENT->>AGENT_TOOLS: evidenceGate(results, threshold)
    AGENT_TOOLS-->>SUB_AGENT: Sufficiency score

    alt Evidence sufficient
        SUB_AGENT->>SUB_AGENT: Generate response with citations
    else Evidence insufficient
        SUB_AGENT->>SUB_AGENT: Apply configured behavior (refuse/caveat/clarify)
    end

    SUB_AGENT-->>RUNNER: Stream result back
```

### Auto-Trigger Memory Recall on New Threads

The memory recall tool uses hybrid access: auto-triggered on the first turn of new threads, agent-initiated on later turns. This balances comprehensiveness (new threads always start with cross-thread context) and efficiency (established threads avoid unnecessary lookups).

```mermaid
flowchart TB
    INCOMING_MESSAGE["Incoming message"]

    FIRST_TURN_CHECK{First message<br/>in thread?}

    AUTO_RECALL["Auto-trigger memoryRecall<br/>before agent starts reasoning.<br/>Results injected as system context<br/>alongside cross-thread messages."]
    AGENT_INITIATED_RECALL["Agent decides whether to call<br/>memoryRecall based on query content.<br/>Tool description guides when to use it."]

    FIRST_TURN_CHECK -->|"Yes — new thread"| AUTO_RECALL
    FIRST_TURN_CHECK -->|"No — existing thread"| AGENT_INITIATED_RECALL
```

**Why auto-trigger on first turn**: In a new thread, the agent has zero local context. A vague message such as “the place I went yesterday” provides little signal for explicit tool planning. Auto-trigger ensures the first turn starts with recent user context available.

**Why agent-initiated after**: Once a thread has context, unconditional recall wastes budget. The agent can decide when memory is relevant based on conversation flow and intent confidence.

**Injection format**: Recalled context is injected as system context alongside thread short-term history and cross-thread messages (for young threads). The orchestrator receives all layers before reasoning begins.

---

## Context Assembly, Budgeting, and Response Calibration

Before reasoning starts, the engine assembles full context in strict priority order:

1. System prompt
2. Current message
3. Tool definitions
4. Last N thread turns
5. Rolling summary
6. Auto-recalled facts
7. User short-term context

After assembly, a token estimator (character count divided by four) computes budget usage and compares it against the context window budget.

If assembled context exceeds budget, truncation runs in reverse priority order:

1. Drop user short-term context first
2. Cap auto-recalled facts to the configured recall token ceiling
3. Compact rolling summary
4. Drop oldest thread turns

System prompt, current message, and tool definitions are never truncated.

At the same assembly stage, the orchestrator injects lightweight response and interpretation signals that shape generation without hard constraints:

- Implicit reference candidates are surfaced when language contains anaphora (for example: “the other one,” “do that again,” “yesterday’s thing”).
- Energy calibration hints are computed from message length, formality markers, and complexity.
- Resumption metadata is injected for long inactivity gaps.
- Clarification policy state is injected to support proactive disambiguation while avoiding clarification loops.

```mermaid
flowchart TB
    INCOMING_MESSAGE[Incoming user message]
    INTENT_SIGNALS[Intent detection signals\nconfidence + interpretation set]
    MEMORY_SIGNALS[Memory signals\nthread history + rolling summary + recall results]
    THREAD_GAP_DETECTOR[Thread gap detector\ntime delta]
    INPUT_CALIBRATOR[Input analyzer\nlength + formality + complexity]

    IMPLICIT_REF_PREP[IMPLICIT_REFERENCE_RESOLUTION\nContext preparation for referents]
    RESPONSE_ENERGY_HINT[RESPONSE_ENERGY_MATCHING\nSoft response energy hint]
    RESUMPTION_CONTEXT[CONVERSATION_RESUMPTION\nResumption context note]
    CLARIFICATION_POLICY[CLARIFICATION_PATIENCE_MODEL +\nPROACTIVE_CLARIFICATION\nClarification strategy state]

    FINAL_CONTEXT[Final agent context\nbehavior adjustments injected]

    INCOMING_MESSAGE --> INTENT_SIGNALS
    INCOMING_MESSAGE --> MEMORY_SIGNALS
    INCOMING_MESSAGE --> THREAD_GAP_DETECTOR
    INCOMING_MESSAGE --> INPUT_CALIBRATOR

    INTENT_SIGNALS --> IMPLICIT_REF_PREP
    MEMORY_SIGNALS --> IMPLICIT_REF_PREP
    INPUT_CALIBRATOR --> RESPONSE_ENERGY_HINT
    THREAD_GAP_DETECTOR --> RESUMPTION_CONTEXT
    INTENT_SIGNALS --> CLARIFICATION_POLICY
    MEMORY_SIGNALS --> CLARIFICATION_POLICY

    IMPLICIT_REF_PREP --> FINAL_CONTEXT
    RESPONSE_ENERGY_HINT --> FINAL_CONTEXT
    RESUMPTION_CONTEXT --> FINAL_CONTEXT
    CLARIFICATION_POLICY --> FINAL_CONTEXT
```

### Implicit Reference Resolution in Context Assembly

Implicit references are handled as a context assembly enhancement layered over recall and budget management. When user language contains anaphora, the engine injects both recent thread turns and relevant recalled candidates before model reasoning begins.

If likely referents are outside the active window, recall scope expands to include candidates from rolling summaries and longer-term strata. This remains retrieval-first rather than interpretation-first: the engine supplies candidates, and the model resolves the referent in normal reasoning.

### Response Energy Matching in Generation Strategy

A response calibration step computes input characteristics and injects a soft calibration hint into context. The signal includes approximate input length, formality markers, and question complexity.

Short casual messages bias toward concise responses; detailed prompts bias toward fuller responses. The hint is advisory. If safety, risk, or correctness requires depth, the agent overrides brevity.

### Thread Resurrection Handling and Conversation Resumption

When the time gap between current and previous thread message exceeds the configured resurrection gap threshold, the orchestrator treats the turn as resurrection.

On resurrection, the orchestrator auto-triggers memory recall using key entities from rolling summary and injects a staleness notice into context:

“This thread has been inactive for [N days]. Some previously discussed context may no longer be available.”

It also injects resumption metadata containing a readable time delta and last-topic summary so the model can acknowledge continuity naturally while still running fresh intent analysis for the new turn.

**Ordering guarantee**: Memory loading (all layers in parallel) completes before intent detection runs. Auto-triggered recall uses raw user message as the search query and does not depend on classified intent. This avoids circular dependency between memory loading and intent detection. See [Memory & Intelligence](./memory.md) for the two-phase memory and intent pipeline.

### Clarification Patience Model in Orchestrator Policy

The clarification policy combines proactive disambiguation and loop prevention.

For proactive clarification, when intent signals indicate genuine ambiguity (low confidence and multiple plausible interpretations), the agent asks one brief clarifying question offering top interpretations.

For patience control, the orchestrator tracks consecutive clarification rounds per thread. After the configured threshold, the agent stops re-asking and returns a best-effort response with explicit assumptions. This guarantees progress and avoids infinite clarification loops.

---

## Live Synthesis Streaming

When multiple sub-agents run in parallel, the orchestrator synthesizes responses in real time as each sub-agent completes:

```mermaid
sequenceDiagram
    participant CLIENT
    participant ORCHESTRATOR
    participant SUB_AGENT_FAST as Sub-Agent A (fast)
    participant SUB_AGENT_SLOW as Sub-Agent B (slow)

    Note over SUB_AGENT_FAST: Completes in 2s
    Note over SUB_AGENT_SLOW: Completes in 5s

    SUB_AGENT_FAST-->>ORCHESTRATOR: Result A ready (2s)
    ORCHESTRATOR->>CLIENT: Start streaming synthesis<br/>"Regarding your refund question..."

    Note over ORCHESTRATOR: Orchestrator writes about<br/>Sub-Agent A findings

    SUB_AGENT_SLOW-->>ORCHESTRATOR: Result B ready (5s)
    ORCHESTRATOR->>CLIENT: Continue streaming<br/>"As for the CEO, ..."

    Note over ORCHESTRATOR: Orchestrator weaves in<br/>Sub-Agent B findings

    ORCHESTRATOR->>CLIENT: Stream complete
```

How live synthesis works:

1. Streaming starts when the first sub-agent completes.
2. The orchestrator writes naturally about that first result.
3. Later completions are woven into the same ongoing stream.
4. Final output is coherent and unified rather than concatenated fragments.

Why this improves UX:

- Time to first token is bounded by fastest sub-agent, not slowest.
- User sees progress immediately.
- Output remains natural because synthesis happens at orchestration level.

---

## Dependent Intent Handling

When the intent validator detects dependency between intents (for example, feedback + constrained search), the orchestrator processes dependent segments sequentially rather than in parallel. This guarantees constraints from earlier results are available to later stages.

```mermaid
flowchart TB
    subgraph INDEPENDENT_INTENTS["Independent Intents (Parallel)"]
        INTENT_SEARCH_A["Intent A: search for restaurants"]
        INTENT_SEARCH_B["Intent B: search for hotels"]
        INTENT_SEARCH_A -->|parallel| SYNTH_INDEPENDENT["Live Synthesis"]
        INTENT_SEARCH_B -->|parallel| SYNTH_INDEPENDENT
    end

    subgraph DEPENDENT_INTENTS["Dependent Intents (Sequential)"]
        FEEDBACK_INTENT["Intent Feedback\n'Place X is terrible'"]
        CONSTRAINED_SEARCH_INTENT["Intent Constrained Search\n'Find me another one'"]
        PRODUCED_CONSTRAINT["Exclusion: Place X"]
        SYNTH_DEPENDENT["Live Synthesis"]

        FEEDBACK_INTENT -->|produces constraint| PRODUCED_CONSTRAINT
        PRODUCED_CONSTRAINT -->|passed to| CONSTRAINED_SEARCH_INTENT
        CONSTRAINED_SEARCH_INTENT -->|receives constraint| SYNTH_DEPENDENT
    end
```

Sequential flow:

1. Feedback intent runs first and produces a constraint.
2. Search intent runs second and receives the constraint.
3. Results are synthesized once both stages complete.

Detection: The intent validator detects dependent structures such as “the one I mentioned is bad, find another” or “I do not like Y, what about Z.”

Constraint passing: After first stage completes, the orchestrator extracts a structured constraint object containing type, entities, and metadata, then passes it through handoff input filtering so the dependent sub-agent must respect it during planning and tool usage.

| Constraint type | When produced | What it carries |
|----------------|---------------|-----------------|
| exclusion | Feedback identifies disliked entity | entities: names or IDs to exclude from later results |
| refinement | Context intent narrows scope | entities: scope qualifiers; metadata: additional filters |
| context | Context-establishing intent provides background | entities: key mentions; metadata: grounding facts |

---

## Dynamic Fan-Out & Map-Reduce Orchestration

For workloads where parallel width is unknown until runtime, the orchestrator uses dynamic fan-out and reduction. Instead of fixed handoff count, it computes `N` from incoming work (for example, many documents or records), dispatches one worker per item or shard, and reduces outputs into one coherent result.

```mermaid
flowchart TB
    ORCHESTRATOR[ORCHESTRATOR]
    WORK_ITEM_SET[WORK_ITEM_SET]
    FAN_OUT_POLICY{FAN_OUT_POLICY}
    WORKER_POOL[WORKER_POOL_N]
    REDUCER[REDUCER]
    FINAL_RESULT[FINAL_RESULT]

    ORCHESTRATOR --> WORK_ITEM_SET
    WORK_ITEM_SET --> FAN_OUT_POLICY
    FAN_OUT_POLICY --> WORKER_POOL
    WORKER_POOL --> REDUCER
    REDUCER --> FINAL_RESULT
```

Core orchestration behaviors:

- Runtime-determined parallelism: worker count is derived at execution time, not fixed during design.
- Fan-out dispatch: the orchestrator partitions work items and assigns each partition to dynamically created workers.
- Result aggregation: reducer strategies include merge, vote, concatenate, and summarize depending on task semantics.
- Partial failure handling: policy is explicit per run as fail-all, succeed-with-partial, or retry-failed-only.
- Concurrency limits: the orchestrator enforces a maximum active worker cap to avoid resource exhaustion.
- Progress tracking: completion is tracked as processed items versus total items, enabling percent-complete reporting.
- Backpressure: when worker output exceeds reducer throughput, the orchestrator applies queueing or throttling to stabilize flow.

Durability note: checkpoint-backed fan-out state and recovery behavior should align with durable execution guidance in [Durable Execution](./durable-execution.md), so incomplete worker sets can resume without reprocessing completed items.

---

## Tool Registry

Each agent (orchestrator or sub-agent) receives a role-specific tool set:

```mermaid
graph LR
    subgraph ORCHESTRATOR_TOOLS["Orchestrator Tools + Handoffs"]
        TOOL_HANDOFFS["Handoffs to sub-agents\n(agent transfer call)"]
        TOOL_QUERY_REWRITE_ORCH["queryRewrite"]
        TOOL_SOURCE_SET_ORCH["All source tools"]
    end

    subgraph SUB_AGENT_TOOLS["Sub-Agent Tools (per intent)"]
        TOOL_SEARCH_DOCUMENT["searchDocument(document_id)"]
        TOOL_RAGFLOW["ragflowSearch"]
        TOOL_GROUNDING["groundingSearch"]
        TOOL_MEMORY_RECALL["Memory Recall"]
        TOOL_QUERY_REWRITE_SUB["Query Rewrite"]
        TOOL_EVIDENCE_GATE["Evidence Gate"]
    end

    subgraph SPECIAL_TOOLS["Special Tools"]
        TOOL_SUGGEST_CTA["CTA Suggestion"]
    end
```

| Tool | Description | Used By |
|------|-------------|---------|
| Document search tool | Search uploaded documents by document ID | Sub-agents, orchestrator (single intent) |
| RAG flow search | Query external knowledge source | Sub-agents |
| Grounding search | Google Search grounding (separate agent call, not LLM tool) | Orchestrator (parallel agent spawn) |
| Memory recall | Long-term semantic recall with auto-trigger on new threads | Sub-agents, auto-trigger on first turn |
| Thread summary | Returns rolling summary for current thread | Orchestrator, sub-agents (on demand) |
| Memory inspector | Returns paginated user memories by category | User-initiated via agent |
| Memory deletion | Deletes specific memories after explicit user confirmation | User-initiated via agent |
| Ordinal resolver | Resolves ordinal references against recent structured result sets | Sub-agents |
| Query rewriter | Conditional query rewriting | Sub-agents, orchestrator |
| Evidence gate | Evidence sufficiency scoring | Sub-agents |
| CTA suggestion | Call-to-action suggestions | Orchestrator only (end of response) |

Additional tool capabilities including generative UI, MCP client, location enrichment, and computer use are defined in [Agent Capabilities & Integrations](./agent-capabilities.md).

### Tool Assignment Flow

```mermaid
flowchart TB
    CLASSIFIED_INTENT_TOPIC["Classified Intent + Topic"]
    TOPIC_CONFIG["Topic sourcesPriority[]"]

    CLASSIFIED_INTENT_TOPIC --> TOPIC_CONFIG
    TOPIC_CONFIG --> TOOL_MAPPING["Map source names to tools"]

    TOOL_MAPPING --> MAP_RAGFLOW["'ragflow' -> ragflowSearch tool<br/>with topic dataset IDs + rewrite strategy"]
    TOOL_MAPPING --> MAP_DOCUMENT_QA["'document_qa' -> searchDocument tool"]
    TOOL_MAPPING --> MAP_GROUNDING_SEARCH["'grounding_search' -> groundingSearch tool<br/>with rewrite strategy"]
    TOOL_MAPPING --> MAP_MEMORY_RECALL["'memory_recall' -> memoryRecall tool"]
    TOOL_MAPPING --> MAP_DIRECT_ANSWER["'direct_answer' -> no tool needed<br/>(agent answers from context)"]

    MAP_RAGFLOW & MAP_DOCUMENT_QA & MAP_GROUNDING_SEARCH & MAP_MEMORY_RECALL & MAP_DIRECT_ANSWER --> SCOPED_SUB_AGENT["Sub-Agent created with<br/>selected tools only"]
```

## Agent Router (Query Classification)

For deployments with multiple specialized agents, the router dispatches each query to the right agent:

```mermaid
flowchart LR
    USER_QUERY["User Query"]
    QUERY_CLASSIFIER["PRIMARY_MODEL<br/>structured output generation<br/>(enum mode)"]
    THREAD_CACHE["Per-thread cache<br/>(Valkey)"]

    USER_QUERY --> THREAD_CACHE
    THREAD_CACHE -->|Hit| AGENT_DISPATCH
    THREAD_CACHE -->|Miss| QUERY_CLASSIFIER
    QUERY_CLASSIFIER --> THREAD_CACHE
    QUERY_CLASSIFIER --> AGENT_DISPATCH

    subgraph DISPATCH_TARGETS["Agent Dispatch"]
        TARGET_MAIN_AGENT["Main Agent"]
        TARGET_GROUNDING_AGENT["Grounding Agent"]
        TARGET_SPECIALIZED_AGENT["Specialized Agent N"]
    end

    AGENT_DISPATCH --> TARGET_MAIN_AGENT & TARGET_GROUNDING_AGENT & TARGET_SPECIALIZED_AGENT
```

Router uses Gemini Flash Lite structured output generation with enum output mode for single-token classification latency. Results are cached per thread in Valkey so the same thread keeps consistent agent continuity.

---

## Scaling: Queue-Based Execution

For production scale, agent execution can run through Trigger.dev:

```mermaid
flowchart TB
    subgraph API_SERVER["API Server (Stateless)"]
        INCOMING_REQUEST["Incoming request"]
        QUICK_QUERY_DECISION{Simple query?}
    end

    subgraph IN_PROCESS_MODE["In-Process (Dev / Simple)"]
        DIRECT_AGENT_EXEC["Execute agent directly"]
    end

    subgraph QUEUED_MODE["Trigger.dev Queue (Production / Heavy)"]
        ENQUEUE_TASK["Enqueue agent task"]
        WORKER_PRIMARY["Primary worker"]
        WORKER_SECONDARY["Secondary worker"]
        WORKER_POOL_N["Worker N"]
    end

    INCOMING_REQUEST --> QUICK_QUERY_DECISION
    QUICK_QUERY_DECISION -->|Yes / Dev mode| DIRECT_AGENT_EXEC
    QUICK_QUERY_DECISION -->|No / Prod mode| ENQUEUE_TASK
    ENQUEUE_TASK --> WORKER_PRIMARY & WORKER_SECONDARY & WORKER_POOL_N
```

Configurable per deployment:

- Dev and testing: all execution in-process (no queue overhead)
- Production (simple): single-intent in-process, multi-intent queued
- Production (full): all requests through queue for uniform scaling and observability

---

## Provider Fallback

Simple sequential fallback for model failures:

```mermaid
flowchart LR
    MODEL_CALL["LLM Call"]
    PRIMARY_PROVIDER["PRIMARY_PROVIDER"]
    FALLBACK_PROVIDER["Fallback Provider<br/>(if configured)"]
    RESULT_SUCCESS["Result"]
    RESULT_ERROR["Error"]

    MODEL_CALL --> PRIMARY_PROVIDER
    PRIMARY_PROVIDER -->|Success| RESULT_SUCCESS
    PRIMARY_PROVIDER -->|Failure| FALLBACK_PROVIDER
    FALLBACK_PROVIDER -->|Success| RESULT_SUCCESS
    FALLBACK_PROVIDER -->|Failure| RESULT_ERROR
```

The fallback model wrapper wraps two providers. If primary fails, it tries fallback. No dynamic smart routing; only sequential try/catch for predictable behavior.

---

## Cross-References

| Component | Interaction |
|-----------|------------|
| **Requirements** ([Requirements & Constraints](./requirements.md)) | Defines quality, safety, and behavior constraints for all orchestrated agent runs |
| **Conversation Pipeline** ([Conversation Pipeline](./conversation.md)) | Provides intent signals, rewritten queries, ambiguity markers, and dependency hints used by orchestrator and router |
| **Memory & Intelligence** ([Memory & Intelligence](./memory.md)) | Supplies thread short-term, user short-term, long-term recall, rolling summaries, and recall policies used during context assembly |
| **Transport** ([Streaming & Transport](./transport.md)) | Carries live synthesis streams, location events, and structured event flow over SSE |
| **Server** ([Server Implementation](./server.md)) | Hosts orchestration runtime, queue integration boundaries, provider fallback wiring, and deployment controls |
| **Agent Capabilities** ([Agent Capabilities & Integrations](./agent-capabilities.md)) | Defines specialized capabilities for computer use, MCP integration, location enrichment, and generative UI components used by the agent runtime |

---

## Task Specifications

### Task AGENT_FACTORY: Agent Creation Factory + Framework Adapter

**What to do**: Build the core agent creation factory that wraps `@openai/agents` framework agent class with safeagent-specific configuration. Use the SDK model bridge from `@openai/agents-extensions` to connect Gemini into the framework model interface. Configure guardrail arrays, register tools (custom + MCP), and connect memory integration. Agent execution uses the framework's execution method to handle tool loop, maximum turn limits, retries, and streaming.

**Depends on**: CORE_TYPES (Types), ZOD_SCHEMAS (Schemas), CONFIG_DEFAULTS (Config), STORAGE_WRAPPER (Storage), PROVIDER_HELPERS (Provider)

**Acceptance Criteria**:

- Agent creation factory returns configured framework agent class instance wrapped with safe defaults
- SDK model bridge for the primary provider produces a valid framework model
- Agent modes (chat, grounding, tools, structured) each return non-empty response through the framework's execution method
- Guardrails are wired as guardrail arrays
- Memory integration includes conversation store with configurable window
- Tool binding includes custom tools and MCP tools
- Thinking level configuration is passed to provider
- Guard mode configuration controls active guardrails
- Request context propagation (user ID, thread ID) reaches tools via runner context
- Unit tests use the AI SDK test mock model from the test utilities package, wrapped via the SDK bridge helper

**QA Scenarios**:

- Create agent with defaults -> framework execution method streams events successfully
- Create agent with grounding -> grounding metadata present in response
- Create agent with tools -> tool calls execute in runner loop
- Create agent with output schema -> typed JSON returned through structured output path
- Two concurrent framework execution method calls with different thread IDs -> no state leakage

---

### Task PROVIDER_FALLBACK: Provider Fallback Helper

**What to do**: Build the fallback model wrapper that wraps two providers with sequential try/catch.

**Depends on**: PROVIDER_HELPERS (Provider helpers)

**Acceptance Criteria**:

- Primary succeeds -> primary result returned
- Primary fails -> fallback is attempted and returned when successful
- Both fail -> original primary error is thrown
- Unit tests with mocked providers

**QA Scenarios**:

- Primary responds normally -> fallback not called
- Primary throws -> fallback called and returned
- Both throw -> original primary error surfaces
- Primary timeout -> fallback engaged within total timeout budget

---

### Task AGENT_ROUTER: Agent Router — Query Classification + Dispatch

**What to do**: Build query classification that routes to the correct agent using structured output generation in enum mode with per-thread Valkey caching.

**Depends on**: AGENT_FACTORY (Agent Factory), SSE_STREAMING (Streaming)

**Acceptance Criteria**:

- Classification uses primary model with minimal thinking level
- Enum output mode is used for single-token classification
- Per-thread Valkey caching keeps same thread on same agent
- Cache invalidation available on explicit user request
- New threads with no cache handled gracefully
- Unit tests with mocked model and cache

**QA Scenarios**:

- New thread (no cache) -> classification runs and result cached
- Cached thread -> cached dispatch used and no classifier call
- User requests reclassification -> cache invalidated and fresh classification executed
- Two concurrent requests for same thread -> no duplicate classification work

---

### Task GEMINI_GROUNDING: Gemini Grounding Agent Mode

**What to do**: Implement Gemini grounding mode using Google Search grounding for real-time web queries. Configure grounding metadata extraction and citation formatting.

**Depends on**: AGENT_FACTORY (Agent Creation Factory)

**Acceptance Criteria**:

- Grounding mode creates an agent configured for Google Search grounding
- Grounding responses include structured metadata for used sources
- Citation formatter converts grounding metadata into consistent citation output
- Grounding mode works in streaming and non-streaming paths
- Grounding failures return typed errors without crashing non-grounding modes
- Unit tests validate grounding metadata extraction and citation formatting

**QA Scenarios**:

- Real-time query in grounding mode -> response includes grounded citations
- Same query in chat mode -> no grounding metadata
- Grounding metadata with multiple sources -> formatter outputs deterministic citation ordering
- Grounding provider error during request -> typed error surfaced and stream closes cleanly
- Streaming grounding response -> citations remain aligned with final answer content

---

### Task ORCHESTRATOR: Orchestrator Agent Framework

**What to do**: Build the supervisor orchestrator pattern using the framework's handoff mechanism. Orchestrator is a framework agent class instance with handoffs pointing to sub-agents. Each handoff uses history scoping to the assigned intent and routing callbacks for logging. For multi-intent messages, orchestrator spawns parallel sub-agent runs and synthesizes results while streaming. The runtime handles handoff execution automatically: when the model makes an agent transfer call, execution switches to the target agent.

**Depends on**: AGENT_FACTORY (Agent Factory), EMBED_ROUTER (Embedding Router), LLM_INTENT (LLM Intent Validator), SOURCE_ROUTER (Source Priority Router)

**Acceptance Criteria**:

- Orchestrator factory creates a framework agent class instance with sub-agent handoffs
- Each handoff has context scoping and routing logging
- Single intent -> orchestrator works directly with tools without handoff overhead
- Multiple independent intents -> parallel sub-agent runs via handoffs, one per intent
- Multiple dependent intents -> sequential runs with constraint passing through handoff-scoped context
- Sub-agents execute independently with their own tool sets
- Auto-trigger memory recall on first turn before reasoning
- Recalled context injected alongside thread short-term and user short-term context
- After first turn, recall returns to agent-initiated mode
- Live synthesis streams as handoffs complete
- First handoff completion starts stream immediately
- Later handoff completions weave into ongoing stream
- If all sub-agents fail, error propagates with context
- If one sub-agent fails, remaining results still synthesize (graceful degradation)
- Framework tracing captures each handoff as span
- Unit tests include mocked sub-agents and dependent intent scenarios

**QA Scenarios**:

- Single intent query -> no handoff and direct execution
- Dual intent query -> two handoffs run in parallel and merged
- Triple intent query -> three handoffs spawned and synthesized stream includes all
- One handoff timeout -> remaining results still delivered with partial-failure note
- All handoffs timeout -> user-facing error response

---

### Task SUBAGENT_FACTORY: Sub-Agent Factory

**What to do**: Build a sub-agent factory that creates intent-scoped framework agent class instances with proper tools from topic source-priority configuration. Each sub-agent is referenced by the orchestrator through the framework's handoff mechanism. The factory produces both a sub-agent instance and corresponding handoff configuration, including context scoping, routing logging, and optional conditional routing predicates.

**Depends on**: AGENT_FACTORY (Agent Factory), ORCHESTRATOR (Orchestrator), SOURCE_ROUTER (Source Priority Router)

**Acceptance Criteria**:

- Sub-agent factory creates a scoped framework agent class instance plus handoff configuration
- Tool set determined by topic source priority
- Each source maps to corresponding tool on agent
- Rewrite strategies assigned per tool based on topic config
- Evidence gate tool included with topic threshold configuration
- Sub-agent prompt includes intent context for focused generation
- Handoff context scoping limits conversation to assigned intent
- Unit tests verify tool assignment from intent config

**QA Scenarios**:

- Topic with three sources -> sub-agent created with three tools
- Topic with external knowledge source -> corresponding tool includes correct dataset IDs
- Topic with HyDE rewrite strategy -> tool configured with HyDE rewriter
- Missing evidence threshold config -> defaults applied
- Handoff context scoping removes irrelevant conversation turns

---

### Task DEPENDENT_INTENT: Dependent Intent Coordination

**What to do**: Build orchestrator support for detecting and processing dependent intents sequentially. When the intent validator reports dependent structure (for example, feedback + constrained search), the orchestrator executes feedback first to produce a constraint, then passes that constraint into dependent search through handoff context scoping, ensuring search respects feedback.

**Depends on**: ORCHESTRATOR (Orchestrator Agent Framework), LLM_INTENT (LLM Intent Validator)

**Acceptance Criteria**:

- Orchestrator detects dependent intents from validator output
- Dependent intents execute sequentially, not in parallel
- Feedback intent executes first and produces constraint
- Constraint passes to dependent intent through handoff-scoped context
- Dependent intent receives constraint and applies it during search and filtering
- Results from both intents synthesize into coherent response
- Independent intents continue to run in parallel without regression
- Unit tests verify sequential and parallel execution paths
- Constraint passing is verified through handoff-scoped context inspection

**QA Scenarios**:

- Two independent intents -> both execute in parallel and merge
- Two dependent intents (feedback + search) -> feedback first, constraint passed, search respects constraint
- Three intents (feedback + search + other) -> dependent chain sequential, independent intent parallelized appropriately
- Constraint prevents disliked item appearing in results -> exclusion works
- One dependent stage fails -> remaining results still delivered with partial-failure note
- Constraint context visible in sub-agent trace -> handoff-scoped context includes constraint

---

## External References

- OpenAI Agents SDK documentation: https://openai.github.io/openai-agents-js/
- OpenAI Agents SDK — Handoffs: https://openai.github.io/openai-agents-js/guides/handoffs
- OpenAI Agents SDK — Guardrails: https://openai.github.io/openai-agents-js/guides/guardrails
- OpenAI Agents SDK — AI SDK bridge: https://openai.github.io/openai-agents-js/extensions/ai-sdk
- AI SDK structured output generation: https://sdk.vercel.ai/docs/ai-sdk-core/generating-structured-data

---


## Test Specifications

**Agent factory**:

- Creation with sensible defaults and full customizability.
- System prompt wiring through Gemini's dedicated system instruction field.
- Provider bridge wiring and model resolution.
- No hardcoded business prompts in library.

**Orchestrator**:

- Multi-intent supervision via handoffs between sub-agents.
- Sub-agent lifecycle management: creation, execution, result collection, cleanup.
- Live synthesis: streaming merge of sub-agent results into unified response.
- Dependent intent handling: sequential processing with constraint passing.
- Independent intent handling: parallel processing with result merging.

**Agent behaviors**:

- Auto-trigger memory recall on first message in new thread.
- Context budgeting: token allocation across layers with priority-based truncation.
- Response energy matching: input characteristics determine output calibration hint.
- Clarification patience model: after threshold turns synthesize best-effort answer.
- Tool registration: dynamic scoping per agent, namespace isolation, no duplicate instances.
- Grounding mode: separate agent mode for web-grounded responses with grounding metadata.
- Provider-agnostic configuration: no model-specific branching in core agent logic.

**Agent factory — configuration, wiring, and mode guarantees**:

- Factory output is a configured agent instance with safe defaults and override support.
- Identity fields and instruction content are required and validated before agent creation.
- Provider bridge wiring produces valid model interface object for execution runtime.
- Guardrails are wired through input and output guard arrays for every created agent.
- Memory integration wiring includes conversation-store support with configurable history window.
- Tool wiring supports custom tools and external tool registrations in one agent configuration.
- Thinking-level input is forwarded to provider configuration when set.
- Guard mode controls active guardrail behavior according to configured mode.
- Request context fields such as user identity and thread identity propagate into tool runtime context.
- Default chat mode returns non-empty response output.
- Grounding mode returns non-empty response output and grounding metadata when applicable.
- Tools mode returns non-empty response output with working tool loop.
- Structured-output mode returns schema-conforming typed JSON result.
- Grounding mode and custom tool mode are mutually exclusive within one agent call.
- Factory creates separate agent instances when both grounding and tool use are needed.
- Concurrent execution calls with different thread identifiers do not leak stream or memory state.

**Agent factory — validation and constraints**:

- Missing required config fields fail fast with descriptive validation errors.
- Invalid guardrail config shape fails validation before runtime execution.
- Invalid memory config shape fails validation before runtime execution.
- Invalid tool definitions fail validation before runtime execution.
- Invalid structured output schema definition fails at setup path.
- Grounding plus custom tools in one config is rejected by mutual-exclusion validation.
- Structured-output and normal-chat execution return type contracts remain distinct and reliable.

**Orchestrator pattern — supervisor and handoff behavior**:

- Orchestrator is configured as supervisor agent with handoffs to intent-scoped sub-agents.
- Handoffs are exercised through agent-transfer control path with explicit ownership transfer.
- Single-intent requests follow direct tool path without handoff overhead.
- Multi-intent requests split into sub-queries and route to sub-agents.
- Independent multi-intent requests run sub-agent executions in parallel.
- Dependent multi-intent requests run in dependency order.
- Handoff context filtering limits each sub-agent to intent-relevant history.
- Routing callbacks record handoff decisions for observability.
- Sub-agent execution results stream back to orchestrator for synthesis.
- Sub-agent failures degrade gracefully when at least one sibling path succeeds.
- All-sub-agent failure path propagates explicit user-facing error.
- Handoff spans are emitted for tracing across orchestrated paths.

**Sub-agent lifecycle and scoped execution**:

- Sub-agent instances are created per intent with isolated tool scope.
- Sub-agent prompt context includes intent-specific framing for focused reasoning.
- Tool scope is derived from topic source-priority configuration.
- Source entries map to expected tool capabilities for retrieval and synthesis.
- Rewrite strategy assignment per source is passed into sub-agent tool config.
- Evidence sufficiency tool is attached with topic threshold configuration.
- Handoff-scoped context excludes irrelevant conversation turns.
- Sub-agent result collection includes partial completions when sibling sub-agents are still running.

**Context assembly and budgeting in orchestration layer**:

- Context assembly order is system prompt, current message, tool definitions, thread turns, rolling summary, recalled facts, then user short-term context.
- Token estimation uses character-count divided-by-four approximation.
- Estimated usage is compared against configured context window budget before reasoning begins.
- Budget overflow triggers truncation in reverse priority order.
- User short-term context is dropped first under budget pressure.
- Auto-recalled facts are capped at configured recall-token ceiling under budget pressure.
- Rolling summary is compacted before thread-turn truncation.
- Oldest thread turns are dropped after higher-truncation-priority layers are reduced.
- System prompt is non-truncatable.
- Current user message is non-truncatable.
- Tool definitions are non-truncatable.

**Implicit references, energy calibration, resumption, and clarification policy**:

- Anaphoric phrases trigger implicit-reference candidate injection before reasoning.
- Referent candidate set includes recent turns and recalled candidates.
- Referent candidates outside active window trigger expanded recall from summary and longer-term layers.
- Reference handling remains retrieval-first and does not force pre-resolution.
- Response-energy hint is computed from length, formality markers, and complexity.
- Short casual inputs bias toward concise response style.
- Detailed inputs bias toward fuller response style.
- Safety and correctness needs can override brevity-biased calibration.
- Resurrection gap detection marks long-inactive threads for resumption handling.
- Resurrection handling injects staleness notice with readable inactivity duration.
- Resumption metadata includes time delta and last-topic summary.
- Resurrection path still runs fresh intent analysis for current turn.
- Clarification policy asks one concise clarifying question for genuine ambiguity.
- Clarification loop counter tracks consecutive clarification rounds by thread.
- When clarification limit is reached, orchestrator returns best-effort response with explicit assumptions.

**Auto-trigger memory recall policy**:

- First turn in a new thread triggers memory recall before agent reasoning starts.
- Auto-trigger recall results are injected alongside thread short-term and user short-term context.
- Auto-trigger recall uses raw incoming message as recall query.
- Existing-thread turns use agent-initiated recall decisions rather than unconditional recall.
- First-turn auto-trigger improves vague-context handling for new-thread openings.
- Later-turn agent-initiated mode avoids unnecessary recall token spend.

**Live synthesis streaming behavior**:

- Multi-agent live synthesis starts streaming on first completed sub-agent result.
- First completion content is expressed naturally instead of as detached fragment.
- Later sub-agent completions are woven into ongoing stream.
- Final output is coherent unified narrative rather than concatenated outputs.
- Time to first token is bounded by fastest sub-agent completion.
- Progress visibility is preserved while slower sub-agents continue.
- Stream completion emits clean terminal event after synthesis finalization.

**Dependent intent coordination and constraint passing**:

- Dependent-intent structures are detected from validator output and trigger sequential orchestration path.
- Feedback-first stage produces structured constraints before dependent search stage starts.
- Constraint object includes constraint type, entities, and metadata.
- Dependent stage receives constraints through handoff-scoped context.
- Dependent search planning respects exclusion constraints from prior stage.
- Independent intents continue parallel execution and are not forced into sequential mode.
- Mixed sets with one dependent chain and one independent intent schedule appropriately.
- Constraint-aware filtering removes disliked entities from downstream result set.
- Partial failures in dependent chains still return available successful outputs with partial-failure signaling.

**Tool registry and assignment flow**:

- Orchestrator tool set includes handoff controls, rewrite capability, and source-execution tools for direct single-intent path.
- Sub-agent tool set includes source-specific retrieval tools, recall tools, rewrite capability, and evidence-scoring tool.
- CTA suggestion capability is restricted to orchestrator end-of-response scope.
- Topic source-priority entries map deterministically to tool assignments.
- External retrieval source maps to external retrieval tool with topic dataset configuration.
- Document retrieval source maps to document-search tool.
- Grounding-search source maps to grounding path with configured rewrite strategy.
- Memory-recall source maps to memory-recall tool.
- Direct-answer source maps to no-tool synthesis path.
- Sub-agent receives only selected scoped tools and no extra global leakage.


**Agent router behavior and caching**:

- Router classifies query to dispatch target for multi-agent deployments.
- Classification runs with minimal-thinking profile for low-latency routing.
- Enum output mode is used for compact routing output.
- Per-thread cache keeps agent continuity across turns.
- Cache miss triggers fresh classification and subsequent cache write.
- Cache hit bypasses classifier call and dispatches cached target.
- Explicit user reclassification request invalidates cached routing decision.
- New threads with no cache route correctly via fresh classification path.
- Concurrent requests on same thread avoid duplicate classifier work.
- Latency measurements cover classifier call and cache hit paths.
- Deployment profile differences in available agents are respected by router dispatch.

**Provider fallback behavior**:

- Primary provider success returns primary result with no fallback call.
- Primary provider failure triggers fallback attempt when fallback is configured.
- Fallback success returns fallback result.
- Primary and fallback failure rethrows original primary error for caller context.
- Primary timeout path can engage fallback within total timeout budget.
- Sequential fallback order is deterministic and test-covered.

**Queue-scaling orchestration modes**:

- Development mode runs orchestrated execution in-process without queue overhead.
- Production simple mode allows single-intent in-process and multi-intent queued execution.
- Production full mode routes all requests through queue for uniform scaling.
- Queue mode preserves orchestration semantics and result synthesis behavior.


### Extension: Dynamic Fan-Out

- Orchestrator spawns the correct number of workers for runtime-determined fan-out size.
- Result aggregation merges outputs from all workers correctly.
- Partial-failure handling follows configured policy for fail-all, succeed-with-partial, and retry-failed behaviors.
- Concurrency limits prevent spawning beyond the configured maximum worker count.
- Progress tracking reports accurate completion percentage throughout execution.
- Backpressure throttles workers when reducer capacity is overwhelmed.
- Worker timeout enforcement terminates individual workers that exceed configured duration.
- Worker state cleanup releases all resources after completion or failure.
- Result ordering matches the original input order regardless of worker completion order.
- Fan-out with zero qualifying inputs completes immediately with an empty result set.
