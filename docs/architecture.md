# Architecture and Design

This document explains how the major subsystems fit together.

## System Overview

At a high level, Dexter is a tool-using agent runtime wrapped in multiple delivery surfaces.

```text
                +----------------------+
                |  User / External     |
                |  Caller              |
                +----------+-----------+
                           |
              +------------+-------------+
              |                          |
   +----------v----------+    +----------v----------+
   | CLI (`src/cli.ts`)  |    | Gateway (`src/     |
   | `src/index.tsx`     |    | gateway/`)          |
   +----------+----------+    +----------+----------+
              |                          |
              +------------+-------------+
                           |
                 +---------v---------+
                 | Agent Loop        |
                 | `src/agent/`      |
                 +---------+---------+
                           |
         +-----------------+------------------+
         |                 |                  |
 +-------v------+  +-------v------+  +--------v--------+
 | Tool Registry |  | Memory Layer |  | Chat / Scratch  |
 | `src/tools/`  |  | `src/memory/`|  | State           |
 +-------+------+  +-------+------+  +--------+--------+
         |                 |                  |
         +-----------------+------------------+
                           |
                 +---------v---------+
                 | External Systems  |
                 | LLMs, finance,    |
                 | search, WhatsApp  |
                 +-------------------+
```

Core layers:

1. Interface layer: CLI and gateway entry points
2. Agent layer: iterative LLM and tool loop
3. Tool layer: finance, search, browser, filesystem, memory, cron, heartbeat, skills
4. State layer: chat history, scratchpad, persistent memory, cron state, gateway sessions
5. Integration layer: model providers, APIs, WhatsApp, LangSmith

## Entry Points

Primary entry points:

- `src/index.tsx`: CLI executable entry
- `src/cli.ts`: terminal application wiring
- `src/gateway/index.ts`: gateway command entry
- `src/evals/run.ts`: evaluation runner

The CLI and gateway both ultimately drive the same core agent.

## Agent Loop

The main agent implementation lives in `src/agent/agent.ts`.

```text
User query
   |
   v
Build system prompt + prior history + memory context
   |
   v
Call model (streaming first)
   |
   +--> no tool calls --> final answer --> done
   |
   +--> tool calls present
           |
           v
      Execute tools
           |
           v
      Add ToolMessages
           |
           v
      Apply budget / persistence / compaction safeguards
           |
           v
      Next iteration
```

The loop is designed around a growing message array:

1. Build the initial message set from system prompt, prior history, and the user query.
2. Call the configured model with streaming when available.
3. If the model returns tool calls, execute them.
4. Add tool results back into the conversation as `ToolMessage`s.
5. Manage context growth with compaction and truncation safeguards.
6. Repeat until the model returns a direct final answer or the iteration limit is reached.

Important design choices:

- Streaming first, blocking fallback
- Read-only tool concurrency where safe
- Strong bias toward keeping full reasoning continuity in the message array
- Explicit handling for context overflow
- A separate final answer path when no more tools are needed

Related files:

- `src/agent/agent.ts`
- `src/agent/tool-executor.ts`
- `src/agent/run-context.ts`
- `src/agent/types.ts`

## Prompt Construction

Prompt assembly is centralized in `src/agent/prompts.ts`.

The system prompt is composed from:

- Current date
- Channel profile, such as CLI or WhatsApp behavior/formatting
- Compact tool descriptions from the registry
- Tool usage policy
- Available skill metadata
- Memory instructions and optional user context
- Optional user research rules from `.dexter/RULES.md`
- Optional identity content from `SOUL.md`
- Optional group-chat context for WhatsApp group responses

This design keeps prompt policy in one place and lets different delivery surfaces reuse the same agent core.

Related files:

- `src/agent/prompts.ts`
- `src/agent/channels.ts`
- `SOUL.md`

## Tool Architecture

The tool registry in `src/tools/registry.ts` is the central catalog for agent tools.

```text
`src/tools/registry.ts`
        |
        +--> finance tools
        +--> search tools
        +--> fetch/browser tools
        +--> filesystem tools
        +--> memory tools
        +--> heartbeat / cron tools
        +--> skill tool
        |
        +--> rich descriptions for prompting
        +--> compact descriptions for token efficiency
        +--> concurrency metadata for executor policy
```

Each registered tool carries:

- A stable tool name
- The actual LangChain tool instance
- A rich description for prompts
- A compact description for token-efficient prompts
- A `concurrencySafe` flag used by the executor

This gives Dexter a single place to control:

- Which tools exist
- Which tools are conditionally enabled based on environment variables
- Which tools can run concurrently
- What the model is told about the tools

Conditional tool inclusion is important. For example:

- `web_search` is added only when Exa, Perplexity, or Tavily credentials exist
- `x_search` is added only when `X_BEARER_TOKEN` exists
- `skill` is added only when skills are discoverable

## Tool Execution Model

Tool execution is separated from the core loop in `src/agent/tool-executor.ts`.

Design goals:

- Run safe, read-only tools concurrently when possible
- Serialize tools that mutate local state
- Support approval flows for sensitive tools such as file writes/edits
- Return tool events suitable for live UI updates

This separation keeps agent orchestration and execution policy from becoming tangled.

## Context Management

Dexter is designed to survive large research sessions.

```text
Normal turn
   |
   v
Microcompact old tool payloads
   |
   v
If a tool result is too large:
  persist to disk + keep preview in context
   |
   v
If total context nears threshold:
  run full compaction / summarization
   |
   v
If provider still overflows:
  trim older rounds and retry
```

Three mechanisms work together:

### Microcompaction

`src/agent/microcompact.ts` performs lightweight trimming each turn.

Purpose:

- Remove low-value bulk from old tool payloads
- Preserve recent reasoning continuity
- Reduce pressure before expensive compaction is necessary

### Full Compaction

`src/agent/compact.ts` performs heavier summarization when the token budget is threatened.

Purpose:

- Summarize earlier context into a smaller representation
- Preserve conclusions while shrinking raw research history

### Oversized Tool Result Persistence

Large tool outputs are persisted to disk and replaced in-context with a preview and file path.

Purpose:

- Prevent giant tool payloads from dominating the prompt
- Allow the model to retrieve exact sections later via `read_file`

Related files:

- `src/agent/microcompact.ts`
- `src/agent/compact.ts`
- `src/utils/tool-result-storage.ts`
- `src/utils/tool-result-budget.ts`

## Conversation State

Dexter uses multiple forms of state rather than a single transcript store.

### In-Memory Chat History

Short-term conversational continuity is handled by `src/utils/in-memory-chat-history.ts`.

Purpose:

- Carry recent turns into new agent runs
- Summarize older turns when needed

### Scratchpad

The scratchpad records the raw research process for each query.

Purpose:

- Persist tool calls and results
- Persist reasoning summaries
- Support debugging and post-run inspection

Relevant file:

- `src/agent/scratchpad.ts`

### Input and Session History

Separate controllers manage the terminal app's navigable prompt history and other UI state.

Relevant files:

- `src/controllers/input-history.ts`
- `src/controllers/model-selection.ts`

## Persistent Memory Design

The memory subsystem is its own architecture, not just a helper.

```text
Markdown memory files (.dexter/memory/*.md)
                |
                v
         MemoryStore reads/writes
                |
                v
      MemoryIndexer chunks + watches files
                |
                v
      MemoryDatabase (SQLite index)
                |
                v
Hybrid retrieval
vector search + text search
                |
                v
temporal decay + MMR reranking
                |
                v
memory_search / memory_get / memory_update
                |
                v
Agent prompt context and future recall
```

Main components:

- `MemoryStore`: markdown file storage under `.dexter/memory/`
- `MemoryDatabase`: SQLite-backed search/index storage
- `MemoryIndexer`: watches and synchronizes files into the database
- `hybridSearch`: combines vector and text search
- Temporal decay and MMR reranking to bias results toward useful, diverse recall

Runtime behavior:

- Memory can be disabled in settings
- Embedding provider selection is configurable
- Session transcripts can be indexed alongside explicit memory files
- Agent startup can inject memory-derived user context into the system prompt
- The agent can update memory via dedicated tools

Related files:

- `src/memory/index.ts`
- `src/memory/store.ts`
- `src/memory/database.ts`
- `src/memory/indexer.ts`
- `src/memory/search.ts`
- `src/memory/temporal-decay.ts`
- `src/memory/mmr.ts`
- `src/memory/flush.ts`

## Multi-Provider Model Layer

The LLM abstraction is split between provider metadata and invocation logic.

```text
Requested model name
      |
      v
`src/providers.ts` resolves provider by prefix
      |
      v
`src/model/llm.ts` builds provider-specific client
      |
      +--> retries / backoff
      +--> tool binding
      +--> structured output
      +--> streaming or blocking call path
      +--> provider-specific optimizations
```

### Provider Registry

`src/providers.ts` is the canonical provider registry.

It defines:

- Provider ID
- Display name
- Model prefix for routing
- API key environment variable
- Fast model for lightweight tasks
- Context window size for compaction-aware behavior

### Invocation Layer

`src/model/llm.ts` handles:

- Provider-specific chat model construction
- Retry logic with exponential backoff
- Structured output support
- Tool binding support
- Streaming and blocking variants
- Anthropic prompt caching annotations

This keeps provider-specific concerns out of the agent loop.

## Skills Architecture

Skills are discovered from `SKILL.md` files.

Design characteristics:

- Metadata is light enough to include in prompts
- Full skill content is loaded only when invoked
- Project-local skills can override built-ins with the same name
- Skills are treated as specialized workflows rather than generic prose prompts

Related files:

- `src/skills/registry.ts`
- `src/skills/loader.ts`
- `src/tools/skill.ts`

## Gateway Architecture

The gateway is a delivery/runtime layer built around channels, sessions, and routing.

```text
WhatsApp inbound message
         |
         v
Channel runtime / inbound parser
         |
         v
Route resolution
         |
         v
Session lookup / session state
         |
         v
Gateway agent runner
         |
         v
Shared Agent core
         |
         v
Outbound formatter / sender
         |
         v
WhatsApp reply
```

Main responsibilities:

- Start channels and keep them connected
- Resolve inbound messages to an agent route/session
- Preserve session context per route
- Deliver outbound responses back through the channel
- Start background services such as heartbeat and cron execution

The WhatsApp implementation includes:

- Authentication and credential storage
- Connection lifecycle management
- Inbound parsing and deduplication
- Outbound sending and typing indicators
- Mention detection for group usage

Related files:

- `src/gateway/gateway.ts`
- `src/gateway/channels/manager.ts`
- `src/gateway/routing/resolve-route.ts`
- `src/gateway/agent-runner.ts`
- `src/gateway/channels/whatsapp/`

## Scheduling and Heartbeat Design

The scheduling subsystem is separate from the gateway channel code even though the gateway starts it.

```text
Agent or config creates scheduled work
              |
              v
      `.dexter/cron/jobs.json`
              |
              v
          Cron runner
              |
              v
          Cron executor
              |
              v
      isolated agent execution
              |
              v
      route/channel delivery

Heartbeat is a specialized recurring workflow:

`.dexter/HEARTBEAT.md`
        |
        v
heartbeat tool <--> heartbeat scheduler/config
        |
        v
periodic monitoring runs
```

Cron design:

- Jobs are persisted to `.dexter/cron/jobs.json`
- The cron tool allows the agent to create and manage jobs
- A runner wakes and executes due work
- An executor handles isolated agent runs and delivery

Heartbeat design:

- Heartbeat state is backed by `.dexter/HEARTBEAT.md`
- A heartbeat tool exposes checklist read/update operations
- Gateway config defines interval, active hours, model, and iteration settings
- Heartbeat integrates with scheduling rather than being a one-off command

Related files:

- `src/tools/cron/cron-tool.ts`
- `src/cron/store.ts`
- `src/cron/runner.ts`
- `src/cron/executor.ts`
- `src/tools/heartbeat/heartbeat-tool.ts`
- `src/gateway/heartbeat/`

## Evaluation Design

The eval runner is intentionally simple and end-to-end.

It:

- Loads a CSV dataset
- Runs the real agent against each question
- Scores answers with an LLM-as-judge evaluator
- Streams progress through a dedicated TUI
- Logs runs to LangSmith for tracking

This makes it useful both as a quality check and as a regression signal for future changes.
