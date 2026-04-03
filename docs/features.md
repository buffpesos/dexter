# Features

This document describes the major capabilities present in the repository today.

## Core Product Surface

Dexter is a CLI-first AI agent for financial research.

Primary capabilities:

- Answer financial research questions interactively in a terminal UI
- Use live tools instead of relying only on model knowledge
- Stream progress while running, including tool activity and reasoning summaries
- Maintain conversation context across turns
- Personalize responses using persistent memory
- Operate through WhatsApp in addition to the CLI
- Run recurring jobs through a cron-style scheduling layer

Relevant files:

- `src/index.tsx`
- `src/cli.ts`
- `src/controllers/agent-runner.ts`
- `src/agent/agent.ts`

## Interactive CLI

The CLI is built on `@mariozechner/pi-tui` and provides an app-like terminal experience instead of a plain request/response shell.

What the CLI supports:

- A running chat log with incremental event rendering
- Tool lifecycle display for started, completed, errored, approved, and denied tool calls
- A working indicator while research is in progress
- Input history and prompt recall
- Model/provider switching
- Inline selection overlays and prompts
- Interruptible runs and follow-up message queueing

Relevant files:

- `src/cli.ts`
- `src/components/`
- `src/controllers/`

## Slash Commands

Available slash commands are defined in `src/commands/index.ts`.

Current commands:

- `/model`: switch LLM provider and model
- `/rules`: show research rules
- `/clear`: clear the conversation
- `/memory`: show remembered user context
- `/heartbeat`: show the heartbeat checklist
- `/history`: show recent conversation summaries
- `/help`: show keyboard shortcuts and tips

## Financial Research Features

Dexter includes finance-specific tooling rather than only general web search.

Structured financial capabilities include:

- Financial statements and fundamentals
- Key ratios and historical ratios
- Analyst estimates
- Revenue segment breakdowns
- Stock price snapshots and historical price data
- Crypto price snapshots and historical price data
- Insider trades
- Earnings data
- SEC filing search and section extraction
- Stock screening
- Financial news

Relevant files:

- `src/tools/finance/`
- `src/tools/finance/index.ts`
- `src/tools/finance/get-financials.ts`
- `src/tools/finance/get-market-data.ts`
- `src/tools/finance/read-filings.ts`
- `src/tools/finance/screen-stocks.ts`

## General Research Features

Beyond structured finance APIs, Dexter can also expand its research surface.

General-purpose capabilities include:

- Web search via Exa, Perplexity, or Tavily
- X/Twitter search when `X_BEARER_TOKEN` is available
- Fetching and extracting article/page content as markdown
- Browser automation for JavaScript-heavy or interactive pages
- Reading, writing, and editing local files

Relevant files:

- `src/tools/search/`
- `src/tools/fetch/`
- `src/tools/browser/`
- `src/tools/filesystem/`

## Persistent Memory

Dexter has a real memory subsystem, not just current-turn chat history.

Memory features include:

- Markdown-backed long-term and daily memory files under `.dexter/memory/`
- Search across stored memory and indexed prior sessions
- Section reads for exact recall
- Add, edit, and delete operations through memory-specific tools
- Session-context loading during agent startup
- Automatic indexing and retrieval scoring using hybrid search
- Memory flush behavior to preserve useful user facts when context pressure is high

Relevant files:

- `src/memory/index.ts`
- `src/memory/search.ts`
- `src/memory/database.ts`
- `src/tools/memory/`
- `src/memory/flush.ts`

## Skills

Dexter supports specialized workflows packaged as `SKILL.md` files.

Built-in skills currently present in the repository:

- `dcf`: discounted cash flow valuation workflow
- `x-research`: X/Twitter research workflow

Skill features:

- Metadata-only discovery for prompt injection
- Full skill loading only when invoked
- Built-in and project-local skills
- Override behavior where project skills can replace built-ins with the same name

Relevant files:

- `src/skills/registry.ts`
- `src/skills/loader.ts`
- `src/tools/skill.ts`
- `src/skills/dcf/SKILL.md`
- `src/skills/x-research/SKILL.md`

## WhatsApp Gateway

The repository contains a WhatsApp runtime so Dexter can run outside the terminal.

Supported gateway features include:

- QR-based WhatsApp login
- One or more WhatsApp accounts
- Direct-message access policies
- Group-chat participation policies
- Mention-based activation in groups
- Routing and per-session context
- Typing/read-receipt support
- Reconnect handling and deduplication
- Message delivery through the same agent core used by the CLI

Relevant files:

- `src/gateway/index.ts`
- `src/gateway/gateway.ts`
- `src/gateway/config.ts`
- `src/gateway/channels/whatsapp/`
- `src/gateway/channels/whatsapp/README.md`

## Scheduled Jobs and Heartbeat

Dexter can run in the background on a schedule.

Two related capabilities exist:

- `cron`: general scheduled jobs stored in `.dexter/cron/jobs.json`
- `heartbeat`: a checklist-driven periodic monitoring workflow backed by `.dexter/HEARTBEAT.md`

What this enables:

- Recurring agent runs
- Scheduled outbound updates
- Operational monitoring loops during configured hours
- Integration between heartbeat state and cron scheduling

Relevant files:

- `src/tools/cron/cron-tool.ts`
- `src/cron/`
- `src/tools/heartbeat/heartbeat-tool.ts`
- `src/gateway/heartbeat/`

## Evaluation Infrastructure

Dexter includes an evaluation runner for testing answer quality on a finance dataset.

Eval features include:

- CSV dataset loading
- Optional random sampling
- End-to-end agent execution per question
- LLM-as-judge correctness scoring
- LangSmith run logging
- Live terminal progress UI

Relevant files:

- `src/evals/run.ts`
- `src/evals/components/`

## Safeguards and Practical Design Features

The codebase includes several operational protections:

- Tool concurrency metadata so safe reads can run in parallel
- Approval gating for file writes and edits
- Tool result size caps with disk persistence for oversized outputs
- Per-turn microcompaction and threshold-triggered full compaction
- Context overflow recovery logic
- Iteration limits to prevent runaway loops
- Retry handling for model calls

Relevant files:

- `src/agent/tool-executor.ts`
- `src/utils/tool-result-budget.ts`
- `src/utils/tool-result-storage.ts`
- `src/agent/microcompact.ts`
- `src/agent/compact.ts`
- `src/model/llm.ts`
