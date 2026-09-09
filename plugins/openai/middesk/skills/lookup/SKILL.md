---
name: lookup
description: Find businesses in Middesk and report on identity, registration status, review tasks, associated people, and risk findings. Use when asked to find, search, list, look up, show, pull up, or get details on a business or businesses.
---

# Look up businesses

Use this skill to search, list, or inspect business entities in Middesk. Determine whether the request targets a specific entity or a broader group before making calls.

## Resolving a specific business

When the request targets a single business:

- **Given a UUID**: Call `retrieve_business` directly.
- **Given an external ID**: Call `list_businesses` with `external_id` for an exact match.
- **Given a business name**: Call `list_businesses` with `q`:
  - **Single clear match**: Call `retrieve_business` using its `id`.
  - **Multiple plausible matches**: Stop and prompt the user to choose. Present the candidates with distinguishing details (name, ID, formation state, status, and what differs between them). Accounts frequently contain near-duplicates created at different times; do not guess or pick on the user's behalf.

    **Duplicates can contradict each other.** Different packages are run on different records, so near-duplicates of the same business frequently hold conflicting findings — such as a watchlist or adverse media flag on one record and clean results on another. That is a real discrepancy about a real entity, not an averaging issue. Highlight conflicting findings explicitly, and do not let one record stand as the complete truth.
  - **No match found**: State clearly that no matching business exists in the account. Do not create a business in this skill.

## Filters

| Filter | Purpose |
| --- | --- |
| `q` | Free-text search matching name and related text fields. |
| `external_id` | Exact match against your internal system identifier. |
| `tags` | Filter businesses by customer segment, cohort, or batch tag. |

## Pagination and volume control

`first` is bounded to 1–10 and defaults to 10. Values above 10 are rejected by the server, so a page of ten is the maximum a single call can return. Keep `first: 10`.

Never paginate automatically through an entire list. A single page returns extensive metadata, and unconstrained pagination wastes context and performance.

- Render summary lists as a compact table:

| Name | ID | Status | Primary address |
| --- | --- | --- | --- |

- Report the match count from `totalCount`, which counts every business matching the filters independent of pagination. Never count by enumerating pages. `totalCount` is counted up to 1000, so a value of 1000 means **1000 or more**, not exactly 1000 — render it as "1000+" and never as an exact total. Below the cap the value is exact.
- State whether more results exist using `pageInfo.hasNextPage`.
- Only fetch the next page when the user explicitly requests it, passing `after` set to `pageInfo.endCursor`.

## Reporting business details

`retrieve_business` returns comprehensive entity details. Summarize key findings concisely (around 150 words) rather than outputting raw JSON:

1. **Entity identity**: Registered legal name, entity type, formation state, and registration date.
2. **Operating location**: Primary physical or mailing address.
3. **Verification status**: Current `status` and translated failure reasons for any failed `reviewTasks` (e.g., Secretary of State registration mismatch, address deliverability, or TIN discrepancy).
4. **Leadership**: Key officers or significant owners when present.
5. **Notable findings**: Outstanding review items or compliance flags.

Offer deeper details (such as full registration history or all addresses) upon request rather than presenting them unprompted.

### Related business connections

When `retrieve_business` shows `Connections: Found`, corporate linkages exist.

- The review task indicates presence only; it does not detail related entities.
- Call `list_connections` to inspect connected entities, relationship strength, and shared attributes.
- If `list_connections` returns an authorization error (HTTP 403), state that related business connections are not enabled for this account. Do not report this as an entity with zero connections.
- Follow `connected_business_id` with `retrieve_business` when you need to inspect a connected record in the account.

## Registration search distinction

`search_registrations` and `retrieve_registration_search` query live state Secretary of State scrapers. They are separate from account business records and are not enabled on most accounts. Do not call them for general account lookups. If a user asks for live state registry searches and the call fails because the feature is not enabled, explain the capability is not enabled on the account.
