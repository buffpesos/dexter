# Dependencies and Integrations

This document groups the repository's major dependencies by responsibility.

## Runtime Foundation

Core platform choices:

- Bun for runtime and test execution
- TypeScript with ESM modules
- `tsx` for some TypeScript entry points outside the Bun entry script path

Relevant files:

- `package.json`
- `tsconfig.json`

## Terminal UI

Dexter's terminal UI is built primarily on:

- `@mariozechner/pi-tui`

This powers the app-style CLI with custom components, live rendering, selectors, and progress UI.

## LLM Stack

LangChain is the main abstraction layer for model integrations.

Primary packages:

- `@langchain/core`
- `@langchain/openai`
- `@langchain/anthropic`
- `@langchain/google-genai`
- `@langchain/ollama`

Provider/integration notes:

- OpenAI is the default provider
- Anthropic receives explicit prompt caching annotations for lower repeated-input cost
- OpenRouter is accessed through OpenAI-compatible configuration
- xAI, Moonshot, and DeepSeek are also wired through OpenAI-compatible clients
- Ollama is supported for local models

Relevant files:

- `src/model/llm.ts`
- `src/providers.ts`

## Financial Data

The finance subsystem depends on the Financial Datasets API.

What it powers:

- fundamentals
- market data
- filings
- estimates
- screening
- insider trades
- news

Relevant files:

- `src/tools/finance/api.ts`
- `src/tools/finance/`

## Web Search and External Content

Search providers:

- Exa
- Perplexity
- Tavily

Page/content extraction stack:

- `@mozilla/readability`
- `linkedom`

These support article extraction and readable markdown conversion for fetched web content.

Relevant files:

- `src/tools/search/`
- `src/tools/fetch/web-fetch.ts`
- `src/tools/fetch/web-fetch-utils.ts`

## Browser Automation

Dexter uses:

- `playwright`

This enables interaction with JavaScript-rendered pages and more dynamic browsing behavior than simple fetches.

Install note:

- `postinstall` runs `playwright install chromium`

## WhatsApp Integration

Gateway-specific dependencies:

- `@whiskeysockets/baileys`
- `qrcode-terminal`

These enable:

- WhatsApp authentication
- QR-code login in the terminal
- session persistence
- inbound/outbound messaging

Relevant files:

- `src/gateway/channels/whatsapp/`
- `src/gateway/index.ts`

## Persistent Memory and Local Storage

Persistence dependencies:

- `better-sqlite3`

The memory system uses SQLite-backed indexing plus markdown files on disk.

The repository also contains logic to use Bun SQLite where available with fallback behavior in the database layer.

Relevant files:

- `src/memory/database.ts`
- `src/memory/indexer.ts`
- `src/memory/store.ts`

## Scheduling

Cron-style scheduling uses:

- `croner`

This supports the scheduled job runner and heartbeat-related background execution.

Relevant files:

- `src/cron/`
- `src/tools/cron/cron-tool.ts`

## Validation and Parsing

Schema and parsing helpers include:

- `zod`
- `gray-matter`

Where they are used:

- tool schemas
- gateway config validation
- eval schema output
- skill metadata parsing from `SKILL.md`

Relevant files:

- `src/gateway/config.ts`
- `src/skills/loader.ts`
- `src/evals/run.ts`

## Observability and Evaluation

Evaluation and experiment tracking use:

- `langsmith`

This is used by the evaluation runner to create datasets, runs, and result metadata.

Relevant file:

- `src/evals/run.ts`

## Development and Testing

Development/test dependencies include:

- `typescript`
- `jest`
- `babel-jest`
- `ts-jest`
- `@types/*` packages

The repository's primary test command is still `bun test`, even though Jest compatibility packages remain in the dependency graph.

## Dependency Summary by Capability

If you want a quick mapping from capability to outside system:

- LLM reasoning: OpenAI, Anthropic, Google, xAI, OpenRouter, Moonshot, DeepSeek, Ollama
- Financial data: Financial Datasets API
- Web search: Exa, Perplexity, Tavily
- Social/search: X/Twitter bearer token integration
- Web pages: Readability, Linkedom, Playwright
- Messaging: Baileys for WhatsApp
- Memory/indexing: SQLite-backed storage
- Evaluation: LangSmith
