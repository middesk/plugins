# Middesk plugins

Middesk integrations for AI assistants and coding harnesses. Every plugin here is a thin wrapper over the same
backend — Middesk's hosted MCP server at `https://mcp.middesk.com/mcp` — so a harness gets support
by adding a directory, not by growing new API surface.

## Layout

```
plugins/
  claude/middesk/     Claude Code plugin
  openai/middesk/      ChatGPT and Codex plugin
.claude-plugin/
  marketplace.json    Claude Code marketplace manifest
```

One directory per integration surface, each following that surface's own conventions: Claude Code
expects `.claude-plugin/plugin.json`, while the OpenAI package uses `.codex-plugin/plugin.json` and
is shared by ChatGPT and Codex. Adding an integration surface means adding
`plugins/<surface>/middesk/` and whatever manifest it looks for.

## Harnesses

| Harness | Path | Status |
| --- | --- | --- |
| Claude Code | [`plugins/claude/middesk`](plugins/claude/middesk) | In development |
| ChatGPT + Codex | [`plugins/openai/middesk`](plugins/openai/middesk) | In development |

## Using the Claude Code plugin

See [`plugins/claude/middesk/README.md`](plugins/claude/middesk/README.md) for install and
authentication. For a single session:

```bash
claude --plugin-dir plugins/claude/middesk
```

## Sharing content across harnesses

Skills are duplicated per harness rather than symlinked or generated, because each harness tunes
the parts that drive invocation — a skill's `description` frontmatter especially — even when the
body is identical. Write skill bodies to be harness-agnostic so copying one across costs only a
retuned description. Nothing enforces this; keeping them in step is manual.
