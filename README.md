# middesk

A Claude Code plugin that connects Claude to [Middesk](https://middesk.com)'s hosted MCP server.

This plugin is **configuration only**. It ships no server code, no API wrappers, and no credentials
— it points Claude Code at `https://mcp.middesk.com/mcp` and gets out of the way. You authenticate
with your own Middesk account.

## Requirements

- Claude Code 2.0+
- A Middesk account

## Install

Load it directly from a checkout:

```bash
claude --plugin-dir /path/to/claude-code-plugin
```

Or add it to a project's `.claude/settings.json`:

```json
{
  "plugins": ["./path/to/claude-code-plugin"]
}
```

## Authenticate

Pick one of the two paths below. You only need one.

### OAuth (recommended)

Install the plugin, start Claude Code, then:

```
/mcp
```

Select `middesk` and approve the browser prompt. That's the whole flow — the server supports
dynamic client registration, so Claude Code handles the redirect on its own and there is nothing
extra to configure.

### API key

If you would rather use a Middesk API key than sign in, register the server with an
`Authorization` header:

```bash
claude mcp add --transport http middesk https://mcp.middesk.com/mcp \
  --header "Authorization: Bearer mk_live_..."
```

> **Note:** this registers the Middesk server *directly* with Claude Code, separately from the
> server this plugin declares in its `.mcp.json`. It is an alternative to the OAuth path above, not
> an addition to it. If you do both, Claude Code notices the two point at the same server and
> suppresses the plugin's copy in favour of the one you configured by hand — so your API key stays
> in effect.

Keep your key out of version control. `claude mcp add` writes to your local Claude Code config, not
to this repo.

## Tools

Nine tools are exposed by the server; the seven generally available ones are documented here.

### Businesses

| Tool | What it does |
| --- | --- |
| `list_businesses` | Paginated list of your businesses. Search with `q`; filter by `external_id` or `tags`. |
| `retrieve_business` | Full detail for a single business by Middesk ID. |
| `create_business` | Create a business for verification and monitoring. Requires `name` and at least one address. Optionally takes `tin`, `people`, `external_id`, `unique_external_id`, and `tags`. |

### Orders

| Tool | What it does |
| --- | --- |
| `create_order` | Order a product package against an existing `business_id`. |
| `list_orders` | Paginated list of orders for a given business. |

Packages available to `create_order`:

| Package | Covers |
| --- | --- |
| `business_verification_verify` | Comprehensive business verification, including Secretary of State registrations. The usual choice. |
| `tin` | Tax identification number verification. |
| `documents` | Business documents. |
| `ucc_liens` | UCC lien filings. |
| `litigations` | Litigation records. |
| `adverse_media` | Adverse media screening. |

### Signals

| Tool | What it does |
| --- | --- |
| `create_signal` | Instant, lightweight risk assessment. Requires `name` and at least one address. Faster than a full order — reach for it when you need a quick read rather than a deep one. |
| `list_signals` | Paginated list of past signals. Filter by date range, `model_slug`, `reason_codes`, `external_id`, or `batch_id`. |

## Heads up: `create_business` places orders

Creating a business **always** triggers order inference. At minimum that means a
`business_verification_verify` order, plus any other product your account is configured to run
automatically.

There is no way to opt out of this over MCP.

This is intended Middesk behavior rather than a quirk of the plugin, but it is worth knowing before
you ask Claude to create businesses in bulk — you will be billed for the orders those calls
generate. If you only want a quick risk read without placing an order, use `create_signal` instead.

## What this plugin does not include

Skills and slash commands are not part of this release. The tools above are available to Claude
directly once the server is connected — just describe what you want in plain language.

## Support

- [Middesk documentation](https://docs.middesk.com)
- [Middesk MCP server](https://docs.middesk.com/build/mcp-server)
