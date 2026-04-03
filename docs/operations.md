# Operations and Workflows

This document focuses on how the repository is intended to be run and operated.

## Local Development Workflow

Typical local loop:

```bash
bun install
cp env.example .env
bun run start
```

For iterative development:

```bash
bun run dev
```

Verification commands:

```bash
bun run typecheck
bun test
```

## CLI Usage Workflow

1. Start Dexter with `bun run start`.
2. Ask a research question.
3. Watch the live chat log show reasoning summaries and tool activity.
4. If needed, interrupt the run or queue a follow-up question.
5. Use slash commands for model switching, memory inspection, rules, history, and help.

The CLI is the simplest way to exercise the full agent stack.

## Financial Research Workflow

A typical financial question flows like this:

1. User asks a question such as a multi-company comparison or filing-driven analysis.
2. The agent calls `get_financials`, `get_market_data`, `read_filings`, or `stock_screener` depending on the request.
3. If structured data is not enough, Dexter may widen the research via search, fetch, browser, or X/Twitter tools.
4. The agent synthesizes the result into a concise final answer.

The prompt explicitly encourages broad, single-call use of finance tools for multi-company and multi-metric requests.

## Memory Workflow

Memory usage has two layers.

### Automatic Memory Use

On startup, the agent can load session context from the memory system and inject that into the system prompt.

During personalized financial advice, the prompt instructs the agent to search memory first.

### Explicit Memory Tools

Memory tools support:

- `memory_search`: search stored facts and indexed past sessions
- `memory_get`: read exact file sections
- `memory_update`: add, edit, or delete memory

Operationally, this means Dexter can both recall and curate durable user context over time.

## Large-Context Workflow

Longer research sessions are handled with a layered strategy:

1. Microcompact older low-value tool payloads
2. Persist very large tool outputs to disk and keep a preview in context
3. Run deeper compaction when the token threshold is crossed
4. Retry after trimming if the provider returns a context-overflow error

This is one of the most important architectural traits in the repo because it lets Dexter keep working through larger tasks.

## WhatsApp Workflow

### Initial Login

```bash
bun run gateway:login
```

This links a WhatsApp account by QR code and writes local auth/config state.

### Runtime

```bash
bun run gateway
```

Operational behavior:

- start the configured channel runtime
- accept inbound messages
- resolve sessions/routes
- run the shared agent core
- send responses back through WhatsApp
- keep background services such as heartbeat/cron active

### Direct Messages

Direct-message behavior depends on `dmPolicy` and `allowFrom` configuration.

### Group Chats

Group behavior depends on `groupPolicy` and mention detection.

Current design intent:

- Dexter stays quiet unless explicitly @-mentioned
- it can use recent group context when replying
- it is suitable for assistant-style participation rather than constant autonomous posting

## Scheduled Job Workflow

Dexter includes a cron-management surface exposed as a tool and backed by local storage.

Operational sequence:

1. A cron job is created or updated.
2. Job definitions are stored in `.dexter/cron/jobs.json`.
3. The runner waits for due execution windows.
4. The executor launches an isolated agent run for the scheduled task.
5. Results are delivered through the configured route/channel.

This makes the repository useful for recurring research or alerting workflows, not only ad hoc chat.

## Heartbeat Workflow

Heartbeat is effectively a recurring monitoring checklist.

Operational pieces:

- `.dexter/HEARTBEAT.md` stores the checklist state
- the heartbeat tool reads and updates that state
- gateway config defines interval, active hours, and model settings
- scheduling keeps the heartbeat loop running automatically

This is useful for periodic market checks or recurring review tasks during trading hours.

## Skills Workflow

Skills are meant for specialized tasks where a fixed workflow is more reliable than free-form reasoning.

Current workflow:

1. Skill metadata is exposed in the system prompt.
2. The agent chooses a skill when relevant.
3. The `skill` tool loads the full `SKILL.md` instructions.
4. The agent follows that specialized workflow within the same overall run.

Two concrete examples in the repo are DCF valuation and X/Twitter research.

## Evaluation Workflow

Run a sample evaluation:

```bash
bun run src/evals/run.ts --sample 10
```

Run the full dataset:

```bash
bun run src/evals/run.ts
```

The evaluation workflow:

1. Load the finance dataset CSV
2. Optionally sample questions
3. Run Dexter against each question
4. Score answers with a structured LLM evaluator
5. Send run metadata to LangSmith
6. Render progress in a terminal UI

## Operational Files Worth Knowing

Useful local files when operating or debugging Dexter:

- `.dexter/gateway.json`
- `.dexter/HEARTBEAT.md`
- `.dexter/cron/jobs.json`
- `.dexter/memory/`
- `.dexter/scratchpad/`
- `.dexter/gateway-debug.log`

## Testing and Validation Workflow

Recommended checks after changing behavior:

```bash
bun run typecheck
bun test
```

For changes affecting agent quality or tool orchestration, the eval runner is also useful.

## Where to Go Next

- For subsystem internals, read [Architecture and Design](./architecture.md)
- For available capabilities, read [Features](./features.md)
- For env vars and local state, read [Configuration and Environment](./configuration.md)
- For external systems, read [Dependencies and Integrations](./dependencies.md)
