# middesk

A Claude Code plugin that connects Claude to [Middesk](https://middesk.com)'s hosted MCP server.

This plugin is **configuration only**. It ships no server code, no API wrappers, and no credentials
— it points Claude Code at `https://mcp.middesk.com/mcp` and gets out of the way. You authenticate
with your own Middesk account.

## Requirements

- Claude Code
- A Middesk account with live API access

Trial accounts issue test API keys only, and the MCP server runs against Middesk's production
environment — so a test key is rejected on every call. The tools will appear and then fail. If you
are on a trial, talk to Middesk about API access before installing.

## Install

This plugin lives in the [`middesk/plugins`](https://github.com/middesk/plugins) monorepo, under
`plugins/claude/middesk`. Clone the repo, then point Claude Code at that directory.

```bash
git clone git@github.com:middesk/plugins.git middesk-plugins
```

For a single session:

```bash
claude --plugin-dir middesk-plugins/plugins/claude/middesk
```

To have it load automatically in every session, copy it into your skills directory:

```bash
cp -r middesk-plugins/plugins/claude/middesk ~/.claude/skills/middesk
```

It loads as `middesk@skills-dir` on your next session. For a single project instead of everywhere,
use `.claude/skills/middesk` in the project root; it loads once you trust the workspace.

## Authenticate

Pick one of the two paths below. You only need one.

### OAuth (recommended)

Install the plugin and start Claude Code. The first time, Claude Code asks you to approve the
MCP server the plugin declares — until you do, it shows as `Pending approval` and no Middesk tools
are available. Approve it, then run:

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

> **Note:** this registers a Middesk server *directly* with Claude Code, separate from the one
> this plugin declares in its `.mcp.json`. Claude Code does not merge or de-duplicate the two —
> both will appear, pointing at the same URL, and Claude sees the tools twice. Pick one path:
> either use the API key without enabling the plugin's server, or use OAuth and skip this section.
> If you do register your own, give it a distinct name (`--transport http middesk-api ...`) so it
> does not collide with the plugin's `middesk`.

Keep your key out of version control. `claude mcp add` writes to your local Claude Code config, not
to this repo.

## Tools

The server exposes nine tools. The seven below are available to every account; two further tools
require account-level enablement and are omitted here.

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

## Skills

Three skills ship with the plugin. Each is invocable directly, and Claude also reaches for them on
its own when a request matches.

| Skill | What it does |
| --- | --- |
| `/middesk:order` | Orders a package against a business, creating the business first if it does not exist. Checks the prerequisite chain and warns before placing billable orders. |
| `/middesk:retrieve` | Full detail on one business. Resolves a name to an ID, and asks which you meant when several match. |
| `/middesk:search` | Finds businesses in your account by name, tag, or external ID. |

You do not have to use them — the tools above are available to Claude directly once the server is
connected, and plain language works. The skills add the parts the tool descriptions leave out: which
package slugs are real, which orders require a verification order first, and when a call is about to
bill you.

## Support

- [Middesk documentation](https://docs.middesk.com)
- [Middesk MCP server](https://docs.middesk.com/build/mcp-server)
