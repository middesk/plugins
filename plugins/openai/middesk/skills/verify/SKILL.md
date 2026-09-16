---
name: verify
description: Verify a business through Middesk end-to-end — determine appropriate diligence checks, check for existing records, place billable orders, and interpret verification and risk findings. Use when assessing businesses for KYB, onboarding, credit underwriting, or enhanced risk reviews.
---

# Verify a business

This skill guides a complete verification workflow: establishing the required diligence checks, checking for existing records, placing orders in the proper sequence, and interpreting findings against the compliance or credit decision.

## Confirmation gate: `create_business` places billable orders

Creating a business places orders that incur billing. You control which orders run:

- **Pass an explicit `orders` array**: Middesk places only the specified packages, skipping packages your account is configured to run automatically.
- **Omit `orders` or pass `orders: []`**: Creation falls through to automatic inference, ordering `business_verification_verify` at minimum plus any account-configured automatic packages. An empty array (`orders: []`) is treated as omitted.
- **Website orders**: A `website` order may still be appended when submitted data implies one.

Always state the exact packages to be ordered and confirm before calling `create_business`. Do not infer approval from an open-ended request like "verify this business."

Ordering against an existing business through `create_order` also incurs billing, but does not trigger account-automatic creation packages. State the packages to be ordered before calling `create_order`.

### Re-verifying approved entities

Placing a `business_verification_verify` order on a business whose status is `approved` resets that status to `in_review`. This clears the existing operational approval. Confirm before re-ordering verification on an approved business.

For quick risk assessments without creating records or placing orders, use `create_signal` instead.

## Step 1 — Establish diligence scope

Align the packages to the underlying decision:

| Decision | Recommended packages | Purpose |
| --- | --- | --- |
| Onboarding / KYB | `business_verification_verify` | Confirms legal existence, Secretary of State registration, address deliverability, and TIN matching. |
| Credit underwriting | `business_verification_verify`, `ucc_liens`, `litigations`, `bankruptcies` | Adds secured asset claims, active litigation exposure, and bankruptcy filings. |
| Enhanced diligence / high-risk | `business_verification_verify`, `ucc_liens`, `litigations`, `adverse_media`, `documents`, `enhanced_screenings` | Adds negative news screening, filed formation documents, and watchlist or sanctions screening. |

### Check account entitlements

Before proposing or placing orders, call `list_enabled_packages` to verify which packages the account can order:

- Propose only packages present in the enabled list. If a requested check is unavailable, inform the user factually that the account is not enabled for that package. State entitlement limitations strictly as neutral facts. Do not mention commercial tiers, rates, plan names, sales outreach, or additional package access options.
- **Do not propose `watchlist`**: The correct package slug for watchlist and sanctions screening is `enhanced_screenings`.
- **Do not propose deprecated packages**: `identity` and `business_verification_qualify` are deprecated products. `identity` may still appear in `list_enabled_packages` responses, but must not be used. Use `business_verification_verify` for comprehensive identity and registration verification.
- **TIN verification scope**: TIN verification is included automatically inside `business_verification_verify`. Ordering `tin` as a standalone package is only a conditional reorder after verification has completed on a business that has a TIN.

## Step 2 — Identify or create the business

Prefer existing business records over creating duplicates:

1. **Given a UUID**: Use `retrieve_business` directly.
2. **Given an external ID**: Query `list_businesses` with `external_id`.
3. **Given a business name**: Query `list_businesses` with `q` to search for existing entities.

### Guard against duplicate entities

Accounts frequently accumulate multiple records for the same business, often created minutes apart with differing order history.

- If multiple matching records exist, **stop and ask the user to clarify**.
- Present candidate records with key distinguishing details: legal name, business ID, formation state, primary address, status, existing orders, and differing review findings.
- Near-duplicates may hold conflicting findings (e.g., an adverse screening flag on one record and clean results on another). Highlight these discrepancies rather than averaging or choosing between them.
- Never quietly pick one duplicate over another, and never create a new record when a plausible match already exists without explicit user confirmation.

### Creating a new business

Only call `create_business` when no matching record exists and the user has confirmed the billable package scope:

- Supply `name` and at least one structured or full address.
- Include optional fields (`tin`, `people`, `external_id`, `tags`) when available to improve match accuracy.
- Pass the agreed package set directly in the `orders` array on `create_business`.

## Step 3 — Place orders and handle dependencies

When adding packages to an existing business, call `create_order` with the exact package slug (e.g., `ucc_liens`, `litigations`, `bankruptcies`, `adverse_media`, `documents`, `enhanced_screenings`).

### Prerequisite dependencies

Most supplemental packages depend on a completed or running `business_verification_verify` order:

1. Call `list_orders` for the `business_id`.
2. Inspect the `business_verification_verify` order:
   - **Absent**: Place `business_verification_verify` first, then place the supplemental packages.
   - **`created` or `pending`**: You can place dependent packages immediately. Middesk queues them and starts them automatically once verification finishes. Do not block or wait for verification to finish before placing orders.
   - **`completed`**: Place dependent packages as normal.

## Step 4 — Report findings against the decision

Deliver a clear readout tailored to the original compliance or credit question.

### When orders are still pending

Verification typically takes seconds to several minutes, and supplemental orders may queue behind it. When orders have not finished:

- State clearly which orders are currently in progress (`created` or `pending`) and which are queued behind business verification.
- Inform the user that `list_orders` tracks progress as checks complete.
- **Do not imply a clean result from unfinished checks**: Clearly distinguish between "no adverse findings" and "check is still running."

### When orders are completed

Call `retrieve_business` and structure the readout into four concise parts (around 200 words total):

1. **Verdict**: One or two sentences answering the core question (e.g., whether the business is verified and in good standing, or suitable for credit extension).
2. **Key drivers**: Findings that directly impact the decision:
   - Failed `reviewTasks` (e.g., SOS registration mismatch, inactive status, TIN failure).
   - Findings from supplemental orders (e.g., active UCC liens, open litigation, bankruptcy filings, adverse media hits).
   - Note that an `approved` status represents an analyst decision, not necessarily a clean automated result; always review underlying review tasks.
3. **Data gaps and caveats**: Checks that could not be run, unverified data points, or pending items.
4. **Next step**: The single most relevant operational recommendation.

### Investigating corporate connections

When `retrieve_business` shows `Connections: Found`:

- Call `list_connections` to see associated entities, link strength, and shared attributes.
- If `list_connections` returns HTTP 403, state that related business connections are not enabled for this account.
- When `connected_business_id` is present, use `retrieve_business` on that ID to inspect related corporate entities.
