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
  can pick from rather than a paragraph of prose, one option per business.
  <!-- harness: in Claude Code this is the AskUserQuestion tool; a port swaps this line for the
  equivalent prompt mechanism. --> Do not pick for the user. Businesses in Middesk are routinely
  near-duplicates that differ only by address or formation state, so a confident guess is often
  wrong.

  **Show what differs, not just what each one is.** Name and address alone often do not distinguish
  duplicates. What makes the choice possible is the thing that is not the same about them — one has
  liens, another has litigation, a third is an empty stub. Lead each option with that.

  **Duplicates can contradict each other.** Different packages get run on different records, so
  near-duplicates of one business routinely hold different findings — and sometimes opposite ones,
  such as a watchlist hit on one record and no hits on the other. That is a real discrepancy about a
  real entity, not an averaging problem. Say so, and do not let the record the user happened to pick
  stand as the whole answer.
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

`first` is bounded to 1–10 and the server rejects anything larger, so a page of ten is the most a
call can return. Treat that page as the answer. `pageInfo.endCursor` is in the response, and
following it to walk the account is the thing not to do — on a large account that is thousands of
records and the slowest call in the toolset. Narrow with `q`, `tags`, or `external_id` instead of
paging. Pass `after` set to `pageInfo.endCursor` only when the user has explicitly asked for more
results.

Every result carries all of its addresses, review tasks, and associated people, so a page of ten is
a lot of JSON. Render a compact table rather than pasting it:

| Name | ID | Status | Primary address |
| --- | --- | --- | --- |

**"How many?" is `totalCount`, not the page.** The response carries `totalCount` — how many
businesses match the filters, independent of paging — so counting never means enumerating. It is
counted up to 1000, and that value means **1000 or more**, not exactly 1000. Render a capped count
as "1000+" and never as an exact total: an account with 40,000 businesses reports the same 1000, and
"you have 1000 businesses" is then simply wrong. Below the cap the number is exact and you can say
it plainly.

## Reporting one business

`retrieve_business` returns everything: every address, every review task, every associated person,
every known name, formation details, TIN. Dumping it raw buries the answer.

However you report review tasks, `Connections: Found` only says connections exist. Call
`list_connections` when the user wants to know who they are — it names them strongest first, with a
confidence score and what links each one. Do not name or count connections without calling it, and
do not assume two similarly named records in the account are connected to each other; that is a
question the tool answers and a name search does not.

If the call fails with a 403 naming support@middesk.com, the account lacks the `related_businesses`
entitlement. Say that, rather than reporting the business as having no connections.

### When the user asked something specific

Answer that and stop. "Is this business verified" leads with `status` and the failing review tasks.
"Who runs it" leads with `people`.

### When they only gave you a name

There is no question to answer, so do not answer all of them. Default to a short identity card:

- **What it is** — legal name as registered, entity type, formation state and date
- **Where it is** — the primary address
- **Whether it checks out** — `status`, plus any `reviewTasks` that failed, translated into what
  they actually mean. An SOS "Submitted Not Registered" is a specific finding, not a generic failure
- **One line** on anything genuinely odd

Then stop and offer the rest. The full registration list, every officer, every address, the complete
review-task roll-up — these are available on request and are almost never what someone wants from a
bare name lookup. A business with 45 state registrations and 10 officers is a footnote and an offer,
not two tables.

### Budget

**Around 150 words for one business.** Treat it as a real constraint. `retrieve_business` returns
far more than the user asked for, and reproducing it is not thoroughness.

Signs it has gone wrong: more than one table, section headings for asides, a menu of next steps at
the end, a full people or registration list nobody asked for, or the reader scrolling before
learning whether the business checks out.

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
