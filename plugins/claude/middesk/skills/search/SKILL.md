---
name: search
description: Search and filter the businesses in the user's Middesk account by name, tag, or external ID. Use when asked to find, search, list, browse, or filter businesses, or to show businesses matching some criteria. For full detail on one specific business, use the retrieve skill instead.
---

# Search Middesk businesses

Lists and filters businesses already in the user's Middesk account. `$ARGUMENTS` is the search
criteria.

## This is not registration search

`search_registrations` and `retrieve_registration_search` search **state Secretary of State
registration records**. They are a different capability from this skill, and they are gated behind
a per-account feature flag (`features.mcp_registration_search`) that **most accounts do not have**.

Both tools are enumerated to every account regardless, so they look available and then fail on call
with `MCP registration search is not enabled`. Do not reach for them from this skill. If the user
explicitly asks for state registration search and the call fails that way, explain that their
account does not have the feature enabled — do not present it as an outage or retry it.

This skill searches businesses in the account. That is `list_businesses`.

## Filters

| Filter | Use it for |
| --- | --- |
| `q` | Free-text search across name and other text fields. The default for a name. |
| `external_id` | Exact match on the user's own identifier. Use this instead of `q` when they give you an ID from their system. |
| `tags` | One or more tags. Good for grouping — test data, a customer segment, a batch. |

## Pagination and output volume

`first` defaults to 10, and the tool description asks you to keep it there for context reasons.
Nothing enforces that — the parameter has no maximum and a larger value is honored — so do not raise
it casually, but do raise it when someone explicitly wants a bigger page and accepts the cost.
To go further instead, pass `after` set to the previous response's `pageInfo.endCursor`, and check
`pageInfo.hasNextPage` before offering more.

Each result carries every address, review task, and associated person — a page of ten is a lot of
JSON. Render a compact table instead of pasting it:

| Name | ID | Status | Primary address |
| --- | --- | --- | --- |

Mention the total returned and whether more pages exist. Then offer the **retrieve** skill for full
detail on any one of them rather than expanding all of them.

## Related

- The **retrieve** skill, for one business in full
- The **order** skill, to place an order against a business you found
