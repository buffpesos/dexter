# Dexter Documentation

This directory documents what this fork of Dexter can do, how it is designed, and what it depends on.

Dexter is not just a terminal chat app. The repository contains:

- An interactive CLI research agent for financial analysis
- A multi-provider LLM runtime with tool calling
- Financial data, filings, search, browser, filesystem, memory, heartbeat, and cron tools
- A persistent memory system with indexing and retrieval
- A WhatsApp gateway with direct-message and group-chat support
- Built-in skills such as DCF valuation and X/Twitter research
- An evaluation runner with LangSmith logging

## Documentation Map

- [Features](./features.md)
- [Architecture and Design](./architecture.md)
- [Configuration and Environment](./configuration.md)
- [Dependencies and Integrations](./dependencies.md)
- [Operations and Workflows](./operations.md)

## Quick Orientation

Main entry points:

- CLI entry: `src/index.tsx`
- Terminal UI: `src/cli.ts`
- Agent loop: `src/agent/agent.ts`
- Tool registry: `src/tools/registry.ts`
- Model/provider routing: `src/model/llm.ts`, `src/providers.ts`
- Memory system: `src/memory/`
- WhatsApp gateway: `src/gateway/`
- Scheduled execution: `src/cron/`
- Skills: `src/skills/`
- Eval runner: `src/evals/run.ts`

## Primary Commands

```bash
bun install
bun run start
bun run dev
bun run gateway:login
bun run gateway
bun run typecheck
bun test
bun run src/evals/run.ts --sample 10
```

## High-Level Mental Model

1. A user asks Dexter a question in the CLI or through the gateway.
2. The agent builds a system prompt from channel rules, available tools, skills, memory, optional rules, and optional identity content.
3. Dexter calls an LLM, decides whether tools are needed, and executes tool calls.
4. Tool results are added back into the running context, with compaction and persistence safeguards when context grows.
5. Dexter produces a final answer and can preserve useful context in memory for future runs.

For subsystem details, start with [Architecture and Design](./architecture.md).
