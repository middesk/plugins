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

The server exposes eleven tools. The nine below cover everyday use; `search_registrations` and
`retrieve_registration_search` need `mcp_registration_search` enabled on the account and are omitted
here.

### Businesses

| Tool | What it does |
| --- | --- |
| `list_businesses` | Paginated list of your businesses. Search with `q`; filter by `external_id` or `tags`. |
| `retrieve_business` | Full detail for a single business by Middesk ID. |
| `create_business` | Create a business for verification and monitoring. Requires `name` and at least one address. Optionally takes `orders`, `tin`, `people`, `external_id`, `unique_external_id`, and `tags`. |
| `list_connections` | The businesses connected to a business, strongest first, with a confidence score and what links each one. Goes behind the `Connections` review task, which only reports Found or Not Found. Requires the `related_businesses` entitlement. |

### Orders

| Tool | What it does |
| --- | --- |
| `create_order` | Order a product package against an existing `business_id`. |
| `list_orders` | Paginated list of orders for a given business. |
| `list_enabled_packages` | The package slugs your account is entitled to order. Accounts differ; a package missing from this list is rejected at order creation. |

Common packages, for reference — `list_enabled_packages` is the authoritative list for your account:

| Package | Covers |
| --- | --- |
| `business_verification_verify` | Comprehensive business verification, including Secretary of State registrations and TIN verification. The usual choice. |
| `documents` | Business documents. |
| `ucc_liens` | UCC lien filings. |
| `litigations` | Litigation records. |
| `adverse_media` | Adverse media screening. |

### Signals

| Tool | What it does |
| --- | --- |
| `create_signal` | Instant, lightweight risk assessment. Requires `name` and at least one address. Faster than a full order — reach for it when you need a quick read rather than a deep one. |
| `list_signals` | Paginated list of past signals. Filter by date range, `model_slug`, `reason_codes`, `external_id`, or `batch_id`. Signals do not carry `batch_id` back, so bring it from whatever created the batch. |

## Heads up: `create_business` places orders

Creating a business places orders and you are billed for them. You control which ones.

Pass an `orders` array to `create_business` and Middesk places only those, skipping the packages
your account is configured to run automatically. Omit it and creation falls through to inference: a
`business_verification_verify` order at minimum, plus `website` or `kyc` when the submitted data
implies them, plus every automatic package on the account. That last part is not predictable from
the request, which is why naming the orders is worth the extra breath.

Two limits. An empty array is not an opt-out — `orders: []` is treated as if the field were omitted
and falls back to inference, so there is no way to create a business without placing any order. And
a `website` order may still be appended when the data implies one.

If you only want a quick risk read without placing an order at all, use `create_signal` instead.

## Skills

Two skills ship with the plugin. Each is invocable directly, and Claude also reaches for them on its
own when a request matches.

| Skill | What it does |
| --- | --- |
| `/middesk:verify` | Runs a verification end to end: works out which checks your decision calls for, finds or creates the business, places the orders in dependency order, and reports the result against the question you asked. |
| `/middesk:lookup` | Finds businesses by name, tag, or external ID, and gives full detail on the one you mean. Asks which you meant when several match. |

Each covers a whole task rather than wrapping a single API call, and each stops to ask when a
choice is yours to make — which business you meant, which checks the decision needs, whether to
accept the orders that creating a business will place.

You do not have to use them: the tools above are available to Claude directly once the server is
connected, and plain language works. The skills add the parts the tool descriptions leave out —
which package slugs are real, which orders depend on a verification order, and when a call is about
to bill you.

## Support

- [Middesk documentation](https://docs.middesk.com)
- [Middesk MCP server](https://docs.middesk.com/build/mcp-server)
