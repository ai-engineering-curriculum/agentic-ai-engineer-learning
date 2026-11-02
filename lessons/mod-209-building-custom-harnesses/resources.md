# Resources for mod-209-building-custom-harnesses

Primary references for building agent harnesses. APIs and conventions in this area change often: check against current docs.

## Harness design

- **Claude Agent SDK** ([docs.claude.com/en/api/agent-sdk/overview](https://docs.claude.com/en/api/agent-sdk/overview)) — the Claude Code harness as a library: loop, tools, permissions, hooks, subagents, compaction, and MCP. Compare each exercise with how the SDK does it.
- **Anthropic — Building effective agents** ([anthropic.com/engineering/building-effective-agents](https://www.anthropic.com/engineering/building-effective-agents)) — the agent patterns a harness implements.
- **Anthropic — Effective harnesses for long-running agents** ([anthropic.com/engineering/effective-harnesses-for-long-running-agents](https://www.anthropic.com/engineering/effective-harnesses-for-long-running-agents)) — the initializer session, feature list, and progress file behind Chapter 5 and exercise-04.
- **Anthropic — Effective context engineering for AI agents** ([anthropic.com/engineering/effective-context-engineering-for-ai-agents](https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents)) — context as a budget; clearing, compaction, and structured notes.
- **Anthropic — Writing effective tools for agents** ([anthropic.com/engineering/writing-tools-for-agents](https://www.anthropic.com/engineering/writing-tools-for-agents)) — tool naming, descriptions, output size, and evaluating tools. Read before Chapter 3.
- **OpenAI Codex CLI (source)** ([github.com/openai/codex](https://github.com/openai/codex)) — an open-source production harness: sandboxing, approvals, patch-based editing, and `codex exec`. Reading its code is a good companion to this module.

## Model API

- **Anthropic — Tool use** ([docs.claude.com/en/docs/agents-and-tools/tool-use/overview](https://docs.claude.com/en/docs/agents-and-tools/tool-use/overview)) — the `tool_use` / `tool_result` contract, parallel tool calls, and `is_error`.
- **Anthropic — Text editor tool** ([docs.claude.com/en/docs/agents-and-tools/tool-use/text-editor-tool](https://docs.claude.com/en/docs/agents-and-tools/tool-use/text-editor-tool)) — Anthropic's `view` / `str_replace` / `create` design, the basis of Chapter 3's edit tool.
- **Anthropic — Prompt caching** ([docs.claude.com/en/docs/build-with-claude/prompt-caching](https://docs.claude.com/en/docs/build-with-claude/prompt-caching)) — `cache_control`, prefix order, lifetimes, minimum lengths, and the usage fields.
- **Anthropic — Streaming** ([docs.claude.com/en/docs/build-with-claude/streaming](https://docs.claude.com/en/docs/build-with-claude/streaming)) — event types for text and tool-input deltas.
- **Anthropic — Errors** ([docs.claude.com/en/api/errors](https://docs.claude.com/en/api/errors)) — status codes, including 429 and 529, for the retry policy.
- **Anthropic — Context windows** ([docs.claude.com/en/docs/build-with-claude/context-windows](https://docs.claude.com/en/docs/build-with-claude/context-windows)) — limits and server-side context management, to compare with exercise-04.

## Isolation

- **Docker run reference** ([docs.docker.com/reference/cli/docker/container/run](https://docs.docker.com/reference/cli/docker/container/run/)) — `--network`, `--read-only`, `--tmpfs`, `--cpus`, `--memory`, `--pids-limit`, `--cap-drop`, `--security-opt`.
- **bubblewrap** ([github.com/containers/bubblewrap](https://github.com/containers/bubblewrap)) — unprivileged Linux sandboxing with mount namespaces.
- **Claude Code — Sandboxing** ([code.claude.com/docs/en/sandboxing](https://code.claude.com/docs/en/sandboxing)) — how a production harness uses OS sandboxes and network proxies.
- **git — plumbing** ([git-scm.com/book/en/v2/Git-Internals-Plumbing-and-Porcelain](https://git-scm.com/book/en/v2/Git-Internals-Plumbing-and-Porcelain)) — `write-tree`, `commit-tree`, `update-ref`, and `GIT_INDEX_FILE` for checkpoints.

## Extensibility

- **Model Context Protocol** ([modelcontextprotocol.io](https://modelcontextprotocol.io)) — the spec: lifecycle, tools, transports, and security considerations.
- **MCP Python SDK** ([github.com/modelcontextprotocol/python-sdk](https://github.com/modelcontextprotocol/python-sdk)) — `ClientSession`, the stdio client, and the high-level server class for test servers (`MCPServer` in 2.x, `FastMCP` in 1.x).
- **Claude Code — Subagents, skills, settings, hooks** ([code.claude.com/docs](https://code.claude.com/docs)) — a reference design for agent files, skill directories, layered settings, and the hook contract.

## Observability

- **OpenTelemetry — GenAI semantic conventions** ([opentelemetry.io/docs/specs/semconv/gen-ai](https://opentelemetry.io/docs/specs/semconv/gen-ai/)) — attribute names for model-call spans. Still evolving.
- **OpenTelemetry Python** ([opentelemetry.io/docs/languages/python](https://opentelemetry.io/docs/languages/python/)) — SDK setup and the in-memory exporter for tests.

> When your harness and a production one disagree on a design choice, find out why. Often the production harness learned something the hard way. Sometimes your use case really is different, and then you've found a reason to own a harness.
