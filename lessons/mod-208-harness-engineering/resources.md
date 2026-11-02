# Resources for mod-208-harness-engineering

Primary references for harness engineering. This area moves fast: check tool docs against your installed version.

## The idea

- **Anthropic — Effective harnesses for long-running agents** (Nov 2025) ([anthropic.com/engineering/effective-harnesses-for-long-running-agents](https://www.anthropic.com/engineering/effective-harnesses-for-long-running-agents)) — the initializer session, JSON feature list, progress file, one feature per session, and end-to-end testing. The basis for Chapter 4.
- **Birgitta Böckeler — Harness engineering for coding agent users** (martinfowler.com, Apr 2026) ([martinfowler.com/articles/exploring-gen-ai/harness-engineering.html](https://martinfowler.com/articles/exploring-gen-ai/harness-engineering.html)) — guides vs. sensors, computational vs. inferential controls, the steering loop, and harnessability. The basis for Chapter 1.
- **OpenAI — Harness engineering** (Feb 2026) ([openai.com/index/harness-engineering](https://openai.com/index/harness-engineering/)) — a team building a product with Codex writing the code; the post that made the term popular.
- **Anthropic — Building effective agents** ([anthropic.com/engineering/building-effective-agents](https://www.anthropic.com/engineering/building-effective-agents)) — the agent patterns the harness wraps.

## Configuring harnesses

- **AGENTS.md** ([agents.md](https://agents.md)) — the cross-tool instruction file convention.
- **Claude Code docs** ([code.claude.com/docs](https://code.claude.com/docs)) — memory and `CLAUDE.md`, skills, subagents, settings and permissions, sandboxing, hooks, and headless mode (`claude -p`).
- **Claude Code — Hooks reference** ([code.claude.com/docs/en/hooks](https://code.claude.com/docs/en/hooks)) — event names, input fields, exit codes, and JSON output. Read before exercise-02.
- **Codex CLI docs** ([developers.openai.com/codex](https://developers.openai.com/codex)) — `AGENTS.md`, sandbox modes and approval policies, `config.toml`, and `codex exec`.
- **Model Context Protocol** ([modelcontextprotocol.io](https://modelcontextprotocol.io)) — connecting tools and data sources to any harness.

## Building harnesses

- See [mod-209: Building a Custom Agent Harness](../mod-209-building-custom-harnesses/resources.md) for the Agent SDK, model API, isolation, MCP, and observability references.

## Companion reference

- **AI Agent Guidebook — Harness Engineering** ([ai-agent-guidebook/best-practices/harness-engineering.md](https://github.com/ai-infra-curriculum/ai-agent-guidebook/blob/main/best-practices/harness-engineering.md)) — a shorter, tool-by-tool reference with links to the guidebook's Claude Code, Codex, and other pages.

> The harness is the part you control. When an agent fails, ask which surface — guide, sensor, permission, hook, or state — would have prevented it, and change that.
