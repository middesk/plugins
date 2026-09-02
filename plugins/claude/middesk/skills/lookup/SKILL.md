---
name: lookup
description: Find businesses in the user's Middesk account and report on them — search by name, tag, or external ID, then give full detail on whichever one they mean, including status, addresses, formation, review tasks, people, and TIN. Use when asked to find, search, list, look up, show, pull up, or get details on a business or businesses.
---

# Look up businesses

Searching and retrieving are the same job at different depths, and which one the user wants is
usually clear from how specific they were. `$ARGUMENTS` is what they gave you — a name, a tag, an
ID, or a description.

## Work out which shape the request is

**Specific enough to mean one business** ("show me Acme Corp", "is Acme verified?") → resolve to a
single business and report it in full.

**Broad** ("find businesses tagged onboarding", "what do we have matching Sequoia") → list the
matches and stop there. Offer detail rather than expanding every result.

When it is genuinely unclear, resolve first and offer the list — a single answer is the more common
intent.

## Resolving to one business

`retrieve_business` takes a Middesk ID, not a name.

**Given a UUID**, call `retrieve_business` directly.

**Given a name**, call `list_businesses` with `q` first, then:

- **One plausible match** → use it, and say which business you resolved to so the user can catch a
  wrong guess.
- **Several plausible matches** → **stop and ask.** Offer the candidates as a real choice the user
  can pick from rather than a paragraph of prose, one option per business, each showing enough to
  tell them apart — name, primary address, status. <!-- harness: in Claude Code this is the
  AskUserQuestion tool; a port swaps this line for the equivalent prompt mechanism. --> Do not pick
  for the user. Businesses in Middesk are routinely near-duplicates that differ only by address or
  formation state, so a confident guess is often wrong.
- **No matches** → say so plainly. Do not create the business. If they want it created, that is the
  **verify** skill, which will warn them about the orders creation places.

**Given an `external_id`** from the user's own system, pass it as `external_id` rather than `q` — it
is an exact filter and skips the ambiguity entirely.

## Filters

| Filter | Use it for |
| --- | --- |
| `q` | Free-text search across name and other text fields. The default for a name. |
| `external_id` | Exact match on the user's own identifier. |
| `tags` | One or more tags. Good for grouping — a customer segment, a batch, a cohort. |

## Pagination and output volume

`first` defaults to 10, and the tool description asks you to keep it there for context reasons.
Nothing enforces that — the parameter has no maximum and a larger value is honored — so do not raise
it casually, but do raise it when someone explicitly wants a bigger page and accepts the cost.
To go further instead, pass `after` set to the previous response's `pageInfo.endCursor`, and check
`pageInfo.hasNextPage` before offering more.

Every result carries all of its addresses, review tasks, and associated people, so a page of ten is
a lot of JSON. Render a compact table rather than pasting it:

| Name | ID | Status | Primary address |
| --- | --- | --- | --- |

Say how many came back and whether more pages exist.

## Reporting one business

`retrieve_business` returns everything: every address, every review task, every associated person,
every known name, formation details, TIN. Dumping it raw buries the answer.

Summarize against what was actually asked. "Is this business verified" leads with `status` and the
failing review tasks. "Who runs it" leads with `people`. Offer the rest rather than pasting it.

Worth surfacing by default:

- `status` — the overall verification state
- `reviewTasks` with `status: failure` — these are why a business is not verified, and they are
  worth translating: an SOS "Submitted Not Registered" is a specific finding, not a generic failure
- `formation` — entity type, state, and date, when present
- Address count, when there are several — a common source of confusion

If the user is heading toward a decision rather than a fact — whether to onboard, whether to extend
credit — hand off to the **verify** skill, which picks checks against the decision and orders them.

## This is not registration search

`search_registrations` and `retrieve_registration_search` search **state Secretary of State
registration records**. They are a different capability, gated behind a per-account feature flag
(`features.mcp_registration_search`) that **most accounts do not have**.

Both tools are enumerated to every account regardless, so they look available and then fail on call
with `MCP registration search is not enabled`. Do not reach for them from this skill. If the user
explicitly asks for state registration search and the call fails that way, explain that their
account does not have the feature enabled — do not present it as an outage or retry it.

This skill searches businesses in the account. That is `list_businesses` and `retrieve_business`.
