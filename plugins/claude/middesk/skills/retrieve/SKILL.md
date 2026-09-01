---
name: retrieve
description: Look up one business in the user's Middesk account and report its details — status, addresses, formation, review tasks, people, TIN. Use when asked to retrieve, look up, show, pull up, or get details on a specific business, whether identified by Middesk ID or by name.
---

# Retrieve a Middesk business

Fetches full detail for a single business. `$ARGUMENTS` is the business — an ID or a name.

## Resolving which business

`retrieve_business` takes a Middesk ID, not a name. Bridge the gap before calling it.

**Given a UUID**, call `retrieve_business` with it directly.

**Given a name**, resolve it first:

1. Call `list_businesses` with `q` set to the name.
2. **Exactly one plausible match** — use it, and say which business you resolved to so the user can
   catch a wrong guess.
3. **Several plausible matches** — show them as a short table (name, ID, primary address, status)
   and ask which one. Do not pick for the user. Businesses in Middesk are routinely near-duplicates
   of each other, differing only by address or formation state, so a confident guess is often wrong.
4. **No matches** — say so plainly. Do not create the business. If they want it created, that is
   the **order** skill, which will warn them about the orders creation places.

If the user supplies an `external_id` from their own system rather than a name, pass it as
`external_id` to `list_businesses` instead of `q` — it is an exact filter and skips the ambiguity.

## Reporting the result

`retrieve_business` returns a lot: every address, every review task, every associated person, every
known name, formation details, TIN. Dumping it raw buries the answer.

Summarize against what was actually asked. If the question was "is this business verified", lead
with `status` and the failing review tasks. If it was "who runs it", lead with `people`. Offer the
rest rather than pasting it.

Useful things to surface by default:

- `status` — the overall verification state
- `reviewTasks` with `status: failure` — these are why a business is not verified
- `formation` — entity type, state, and date, when present
- Address count, when there are several — it is a common source of confusion

## Related

- The **search** skill, to find businesses when you do not have one in mind
- The **order** skill, to place an order against the business once identified
