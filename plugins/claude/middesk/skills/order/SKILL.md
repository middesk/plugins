---
name: order
description: Order a Middesk product package for a business — verification, TIN, documents, UCC liens, litigations, or adverse media. Use when asked to order, run, or pull a Middesk report or package on a business, to verify a business, or to add a business to Middesk. Handles creating the business first if it does not exist yet.
---

# Order a Middesk package

Places an order against a business in the user's Middesk account. `$ARGUMENTS` may name the
business, the package, both, or neither — ask for whatever is missing.

## Read this before creating anything

**`create_business` always places orders, and the user is billed for them.**

The MCP tool exposes no `orders` parameter, so every creation runs Middesk's order inference path:

- `business_verification_verify` always
- `website` and/or `kyc` when the submitted data implies them
- anything the account has configured to run automatically on creation

There is no way to opt out of this over MCP. This is intended Middesk behavior, not a bug — but
someone who asks to "add this business" is not expecting a bill, so **say so and get confirmation
before calling `create_business`**.

The full set is not predictable from the request, because the automatic-on-creation packages are
configured per account. A bare name and address has been observed to place `bankruptcies` alongside
`business_verification_verify`. So name `business_verification_verify` as certain, say that others
may follow depending on account configuration, and do not present a specific list as exhaustive.

If the user only wants a quick risk read and not a report, `create_signal` is the cheaper path —
it does not place an order. Offer it when the request sounds like a lookup rather than a purchase.

## Step 1 — Find the business, or create it

Ordering requires a `business_id`. Prefer an existing business over a duplicate.

1. If given a Middesk ID (a UUID), use it directly.
2. If given a name, call `list_businesses` with `q` set to the name. On several plausible matches,
   show them and ask which one — do not guess. See the **retrieve** skill for the fuller
   disambiguation flow.
3. Only when it genuinely does not exist, create it. `create_business` requires `name` and at least
   one address; `tin`, `people`, `external_id`, `unique_external_id`, and `tags` are optional and
   worth collecting, since richer input produces a better verification result.

Confirm the billing consequence above before this call, not after.

## Step 2 — Choose a package

Pass the exact slug to `create_order` as `package`. Use the full value —
`business_verification_verify`, never `verify`.

| Package | What it covers |
| --- | --- |
| `business_verification_verify` | Comprehensive business verification, including Secretary of State registrations and TIN verification. The usual choice, and a prerequisite for most of the rest. |
| `documents` | Articles of Incorporation, Certificates of Good Standing. |
| `ucc_liens` | UCC lien filings. |
| `litigations` | Business litigation records. |
| `adverse_media` | Negative news and media screening. |

**This list is illustrative, not a guarantee.** It reflects packages Middesk commonly offers, not
what this particular account has enabled. An order can fail because the account does not have that
package. Treat that as an expected outcome and tell the user their account may not have it enabled,
rather than surfacing a raw API error.

### `tin` is a conditional reorder, not a single-product option

Do not offer `tin` as a routine choice, even though it is a real slug that the API and the
`create_order` tool description both accept.

**TIN verification already runs inside `business_verification_verify`.** A business created with a
TIN comes back with `TIN Match`, `TIN Error`, and `TIN Issued` review tasks and a populated `tin`
object, with no separate TIN order on it. So for "verify this business's TIN", the answer is
`business_verification_verify` — or the one already on the business.

Ordering `tin` on its own succeeds only when the business has a TIN **and** its verification order
has completed. Before that, verified against the live API:

- No TIN on the business → `"Business is missing a TIN. Please update the business with a TIN
  before submitting a TIN order."`
- Verification not yet complete → `"Tin order is pending."` This message is misleading: it reports
  the status of the *verification* order, which the reorder validator treats as the predecessor,
  with the requested package name interpolated in. There is no hidden TIN order, and `list_orders`
  is not omitting one.

Reach for `tin` only as a deliberate re-run after verification has completed — for example when a
TIN was added or corrected after the fact. It is a reorder, and reordering is account-gated, so it
can still be refused.

### Do not offer `business_verification_qualify` or `identity`

Both are deprecated products. Nothing in the Middesk codebase marks them as deprecated — they still
have constants, scopes, and handler classes in `app/models/order.rb`, and the API will accept orders
for them. So their presence in `Order::PACKAGES`, in a tool description, or anywhere else is not
evidence that they are valid. Do not offer them, and do not add them back on the strength of having
seen them somewhere.

<!--
TODO(FDE-84): delete this hardcoded package list once per-account order-type availability is
queryable. https://linear.app/middesk/issue/FDE-84

There is no API for it today: GraphqlTypes::QueryType exposes no packages/order-options field, and
PackageSetting (account_id + package_type + enabled) is not reachable via GraphQL.

When that ships, drop this table and the "illustrative" caveat above, and call the real tool.
Keep this list in step with ORDER_PACKAGES in middesk/mcp at src/worker/order-packages.ts until then.
-->

## Step 3 — Respect the prerequisite chain

**Most packages depend on a `business_verification_verify` order.** No tool description says so,
so it has to be handled here.

Dependent packages include `documents`, `ucc_liens`, `litigations`, `adverse_media`, `bankruptcies`,
`tax_liens`, `liens`, `kyc`, `website`, `email_risk`, `enhanced_screenings`, and the `people_*`
products.

Before ordering any of them:

1. Call `list_orders` for the `business_id`.
2. Find the `business_verification_verify` order.
   - **Absent** → place it first, then place the requested package.
   - **`created` or `pending`** → you can still place the order. Middesk queues it and starts it
     automatically once verification completes. Tell the user it is queued and results will follow,
     rather than making them come back.
   - **`completed`** → go ahead as normal.
3. Place the requested order.

**Do not treat a pending verification as a blocker.** Middesk's docs say to *"wait for the business
verification to complete before proceeding"*
(https://middesk.docs.buildwithfern.com/assess-risk/search-liens), but that is the recommended
sequence in a REST walkthrough, not an enforced constraint. The docs are right that *results* depend
on verification; they are not right that you must wait to *place* the order.

Verified against the live API: `litigations` and `documents` both ordered against a `pending`
verification were accepted with no error. A queued order's `startedAt` lands on the same second its
verification completes — Middesk holds dependent orders and starts them automatically. Refusing to
place one blocks work that succeeds unattended.

(If an order is ever rejected outright for this reason, report the rejection rather than assuming
the whole chain is blocked.)

### New businesses are the common case

A business created through `create_business` gets a `business_verification_verify` order from the
inference path, starting at `created`/`pending`. It is not instant, and how long it takes varies a
lot — a few seconds for a business Middesk already knows, several minutes for one it does not. Do
not quote a duration; check `list_orders` for the actual status.

So "add Acme and pull its liens" will usually run against a pending verification. That is fine, and
it is one call: create the business, place the liens order, and tell the user it is queued and
results will follow.

### Order status

Most orders move `created` → `pending` → `completed`. Three further statuses exist but are not
guaranteed stops: `audited` appears only when an analyst reviews the order, and `approved` /
`rejected` only when a review decision follows completion. Do not wait for a status that may never
arrive — treat `completed` as the end state unless you see otherwise.

## Step 4 — Report back

Confirm what was ordered, against which business, and the order's current status. If the order is
still `pending`, say that results are not available yet and that `list_orders` will show progress.
