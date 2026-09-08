# Middesk plugins

Middesk integrations for AI assistants and coding harnesses. Every plugin here is a thin wrapper over the same
backend — Middesk's hosted MCP server at `https://mcp.middesk.com/mcp` — so a harness gets support
by adding a directory, not by growing new API surface.

## Layout

```
plugins/
  claude/middesk/     Claude Code plugin
  openai/middesk/     ChatGPT and Codex plugin
.claude-plugin/
  marketplace.json    Claude Code marketplace manifest
```

One directory per surface, each following that surface's own conventions: Claude Code
expects `.claude-plugin/plugin.json`, while the OpenAI package uses `.codex-plugin/plugin.json` and
is shared by ChatGPT and Codex. Adding a surface means adding
`plugins/<surface>/middesk/` and whatever manifest it looks for.

## Surfaces

| Surface | Path | Status |
| --- | --- | --- |
| Claude Code | [`plugins/claude/middesk`](plugins/claude/middesk) | In development |
| ChatGPT + Codex | [`plugins/openai/middesk`](plugins/openai/middesk) | In development |

## Using the Claude Code plugin

See [`plugins/claude/middesk/README.md`](plugins/claude/middesk/README.md) for install and
authentication. For a single session:

```bash
claude --plugin-dir plugins/claude/middesk
```

## Sharing content across surfaces

Skills are surface-specific deliverables rather than automatically shared files. Keep the underlying
product behavior aligned, but rewrite the skill for each surface's audience, compliance requirements,
and invocation model. Copying an existing skill can be a starting point, not a requirement; review
every copied instruction before shipping it. Nothing enforces this; keeping related skills aligned
is manual.
