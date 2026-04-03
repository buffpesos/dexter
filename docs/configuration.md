# Configuration and Environment

This document summarizes runtime commands, environment variables, and key on-disk configuration files.

## Runtime and Package Manager

Dexter uses Bun as the primary runtime.

Primary commands from `package.json`:

```bash
bun run start
bun run dev
bun run gateway
bun run gateway:login
bun run typecheck
bun test
```

## Environment Variables

The canonical example file is `env.example`.

### LLM Providers

- `OPENAI_API_KEY`
- `ANTHROPIC_API_KEY`
- `GOOGLE_API_KEY`
- `XAI_API_KEY`
- `OPENROUTER_API_KEY`
- `MOONSHOT_API_KEY`
- `DEEPSEEK_API_KEY`
- `OLLAMA_BASE_URL`

Provider routing is prefix-based. Examples:

- `gpt-5.4` -> OpenAI
- `claude-...` -> Anthropic
- `gemini-...` -> Google
- `grok-...` -> xAI
- `openrouter:...` -> OpenRouter
- `ollama:...` -> Ollama

Relevant files:

- `src/providers.ts`
- `src/model/llm.ts`

### Financial Data

- `FINANCIAL_DATASETS_API_KEY`

This powers Dexter's structured finance toolset.

### Search and External Research

- `EXASEARCH_API_KEY`
- `PERPLEXITY_API_KEY`
- `TAVILY_API_KEY`
- `X_BEARER_TOKEN`

Tool-selection behavior:

- `web_search` prefers Exa if available
- If Exa is unavailable, it falls back to Perplexity
- If neither exists, it falls back to Tavily
- `x_search` is only registered when `X_BEARER_TOKEN` exists

### Observability

- `LANGSMITH_API_KEY`
- `LANGSMITH_ENDPOINT`
- `LANGSMITH_PROJECT`
- `LANGSMITH_TRACING`

These are used primarily by the evaluation runner.

## On-Disk State

Dexter uses the `.dexter/` directory for local application state.

Important files and directories:

- `.dexter/settings.json`: persisted user settings such as model/provider choices
- `.dexter/RULES.md`: optional user-defined research rules
- `.dexter/SOUL.md`: optional identity override if present
- `.dexter/scratchpad/`: per-query research traces
- `.dexter/memory/`: persistent memory markdown files and memory index database
- `.dexter/cron/jobs.json`: scheduled jobs
- `.dexter/HEARTBEAT.md`: heartbeat checklist state
- `.dexter/gateway.json`: gateway configuration
- `.dexter/credentials/whatsapp/`: WhatsApp auth state

Note: `SOUL.md` may also be loaded from the bundled repository root when no user override exists.

## Model Selection

Model/provider selection appears in two places:

- Interactive CLI switching through `/model`
- Programmatic configuration passed into `Agent.create(...)`

The provider registry also defines a fast model per provider for lightweight summarization tasks.

## Memory Configuration

Memory defaults are defined in `src/memory/index.ts` and can be overridden from settings.

Notable runtime knobs:

- `enabled`
- `embeddingProvider`
- `embeddingModel`
- `maxSessionContextTokens`
- `chunkTokens`
- `chunkOverlapTokens`
- `maxResults`
- `minScore`
- `vectorWeight`
- `textWeight`
- `watchDebounceMs`
- `temporalDecay`
- `mmr`
- `indexSessions`

Behavioral summary:

- Memory can be fully disabled
- Embeddings can reuse existing model credentials
- Session transcripts can be included in indexing
- Retrieval mixes vector and keyword results with reranking

## Gateway Configuration

The gateway configuration schema is implemented in `src/gateway/config.ts`.

Default config location:

- `.dexter/gateway.json`

Main sections:

- `gateway`
- `channels.whatsapp`
- `bindings`

Supported gateway-level settings include:

- `accountId`
- `logLevel`
- `heartbeatSeconds`
- `reconnect`
- `heartbeat`

Supported WhatsApp account settings include:

- `enabled`
- `authDir`
- `allowFrom`
- `dmPolicy`
- `groupPolicy`
- `groupAllowFrom`
- `sendReadReceipts`

Supported access policies:

- DM policy: `pairing`, `allowlist`, `open`, `disabled`
- Group policy: `open`, `allowlist`, `disabled`

## Heartbeat Configuration

Heartbeat is configured under `gateway.heartbeat`.

Supported settings:

- `enabled`
- `intervalMinutes`
- `activeHours.start`
- `activeHours.end`
- `activeHours.timezone`
- `activeHours.daysOfWeek`
- `model`
- `modelProvider`
- `maxIterations`

The defaults are market-hours-oriented, using New York time and weekdays.

## Skills Configuration

Skills are discovered from:

- Built-ins under `src/skills/`
- Project-local skills under `.dexter/skills/`

Project-local skills override built-ins by name.

## Command Surfaces

Common commands:

```bash
bun run start
bun run dev
bun run gateway:login
bun run gateway
bun run src/evals/run.ts
bun run src/evals/run.ts --sample 10
```

## Practical Setup Profiles

### Minimal CLI Setup

Recommended minimum:

- `OPENAI_API_KEY`
- `FINANCIAL_DATASETS_API_KEY`

Optional but useful:

- `EXASEARCH_API_KEY`

### Local-Model Experimentation

Recommended:

- `OLLAMA_BASE_URL`
- finance and search API keys as needed

### WhatsApp Deployment

Recommended:

- normal CLI/API configuration
- `bun run gateway:login`
- a populated `.dexter/gateway.json`

### Evaluation Setup

Recommended:

- `OPENAI_API_KEY`
- `LANGSMITH_API_KEY`

The evaluator itself currently uses `gpt-5.4` as the judge model.
