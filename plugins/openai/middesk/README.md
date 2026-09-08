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

## Authenticate

Use the authentication flow offered by the OpenAI surface when you connect the plugin. The first
connection may open a browser prompt for Middesk sign-in and consent. Approve the requested access
to make Middesk tools available.

Keep Middesk credentials out of this repository. This package does not store API keys, OAuth
tokens, or other secrets.

## Skills

The manifest reserves `./skills/` for the OpenAI skill package. Skill content is delivered in a
separate task and is intentionally not duplicated here.

## UI

Version 1 ships without UI components. Verification results remain conversational for this
submission; a richer verification-result component can be evaluated for a later version.
