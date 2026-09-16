# Middesk

An OpenAI plugin package for ChatGPT and Codex that connects to Middesk's hosted MCP server.

This package is configuration and documentation only. It ships no server code, API wrappers, or
credentials. The remote MCP server is submitted separately as part of the OpenAI plugin setup.

## Requirements

- An OpenAI surface that supports remote MCP plugins
- A Middesk account with live API access

Trial accounts issue test API keys only, while Middesk's MCP server runs against the production
environment. A test key is rejected by every call, so contact Middesk about API access before
installing the plugin.

## Submit the plugin

1. Submit this directory through OpenAI's plugin submission flow.
2. Choose the **With MCP** path.
3. Provide Middesk's remote MCP server URL:

   ```text
   https://mcp.middesk.com/mcp
   ```

The MCP server is not declared in this package. Do not add a `.mcp.json` file or an
`mcpServers` declaration here; the server URL belongs in the submission flow.

## Heads up: `create_business` places orders

Creating a business places orders and you are billed for them. You control which ones.

Pass an explicit `orders` array to `create_business` and Middesk places only those, skipping the
packages your account is configured to run automatically. Omit it and creation falls through to
inference: a `business_verification_verify` order at minimum, plus any automatic package configured
on the account. That automatic set is not predictable from the request, which is why naming the
orders is essential.

Two limits:
- An empty array is not an opt-out — `orders: []` is treated as omitted and falls back to inference,
  so there is no way to create a business without placing an order.
- A `website` order may still be appended when submitted data implies one.

If you only want a quick risk assessment without placing orders, use `create_signal` instead.

## Authenticate

Use the authentication flow offered by the OpenAI surface when you connect the plugin. The first
connection may open a browser prompt for Middesk sign-in and consent. Approve the requested access
to make Middesk tools available.

Keep Middesk credentials out of this repository. This package does not store API keys, OAuth
tokens, or other secrets.

## Skills

This package includes two skills tuned for compliance, credit, and operations workflows:

- `lookup`: Search and retrieve businesses from Middesk, inspect registration and review findings, navigate pagination, and disambiguate near-duplicate records.
- `verify`: End-to-end business verification and due diligence workflows, including package scope selection, billable order confirmation, duplicate entity detection, order dependency sequencing, and decision-focused findings readouts.

## Local validation

This submitted artifact intentionally omits `.app.json`; the remote MCP connection is configured
through OpenAI's submission flow. Installing the package from a local marketplace alone therefore
does not wire the MCP connection. End-to-end validation should use a gitignored local fixture or an
external marketplace entry. Do not commit credentials or local connection files.

## UI

Version 1 ships without UI components. Verification results remain conversational for this
submission; a richer verification-result component can be evaluated for a later version.
