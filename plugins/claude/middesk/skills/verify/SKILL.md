---
name: verify
description: Run a Middesk verification end to end — work out which checks the decision calls for, create or find the business, place the orders in the right sequence, and report the result against the question being asked. Use when asked to verify, diligence, check out, screen, or onboard a business, to run KYB, to assess a business before extending credit, or to order any Middesk package or report.
---

# Verify a business

This skill owns a whole verification, not a single order: decide what to check, get the business
into Middesk, place the orders, and interpret what comes back. `$ARGUMENTS` may name the business,
the checks, both, or neither.

Work through the steps in order. Each one can be skipped when the user has already answered it —
except the gate immediately below, which always applies.

## Before you create anything — this bills the user

**`create_business` always places orders, and the user is billed for them.** The MCP tool exposes no
`orders` parameter, so every creation runs Middesk's order inference path:

- `business_verification_verify` always
- `website` and/or `kyc` when the submitted data implies them
- anything the account is configured to run automatically on creation

There is no way to opt out over MCP. This is intended Middesk behavior, not a bug — but someone who
asks to "verify this merchant" is not expecting an invoice, so **say what it will cost and get
confirmation before calling `create_business`**. Do not treat "verify them" or "onboard them" as
that confirmation: the user is asking for a result and does not necessarily know that getting one
creates billable orders.

The full set is not predictable from the request, because the automatic packages are configured per
account. A bare name and address has been observed to place `bankruptcies` alongside
`business_verification_verify`. Name `business_verification_verify` as certain, say others may
follow depending on account configuration, and do not present a specific list as exhaustive.

Ordering against a business that **already exists** does not hit this gate — only creation does.
Still say which orders you are about to place, and what they are for.

### Re-running a verification clears an existing approval

One exception, and it is not obvious: **placing a `business_verification_verify` order on a business
whose status is `approved` resets that status to `in_review`.** The analyst decision behind the
approval stops standing, and someone has to make it again. Verified live — a business approved in
May went back to `in_review` the moment a fresh verification completed.

So before reordering a verification on an `approved` business, say that it will clear the approval
and confirm. Anything downstream keyed on `approved` is affected. This matters most when the
refresh is unlikely to tell the user anything new: a state registration that was inactive four
months ago is probably still inactive, and confirming that costs both an order and a standing
approval.

If the user wants a quick read rather than a decision, `create_signal` is the cheaper path — it
places no order and creates no business. Offer it when the request sounds like a lookup rather than
a decision.

## Step 1 — Establish what the verification is for

**Different decisions call for different checks**, and the user usually states a business name
rather than a policy. Do not default to a package list. Ask what the verification is supporting,
and offer these as a real choice the user can pick from rather than a paragraph of prose:
<!-- harness: in Claude Code this is the AskUserQuestion tool; a port swaps this line for the
equivalent prompt mechanism. -->

- **Onboarding / KYB** — is this a real, active business at the address it claims?
- **Extending credit** — the above, plus what is already claimed against it and what it is fighting
- **Enhanced / high-risk review** — the above, plus reputational and public-record exposure
- **Something else** — let them describe the decision in their own words

Skip this step when:

- The user already named the packages, or described the decision.
- The business already has **completed** orders that answer the question. Report what they say and
  offer the deeper checks, rather than interviewing someone about work that is already done. Check
  `list_orders` before asking.

### Mapping the decision to packages

| Decision | Packages | Why |
| --- | --- | --- |
| Onboarding / KYB | `business_verification_verify` | Establishes the business exists, is registered, and matches the submitted name, address, and TIN |
| Extending credit | `business_verification_verify`, `ucc_liens`, `litigations` | Adds existing secured claims against the assets, and active legal exposure |
| Enhanced / high-risk | `business_verification_verify`, `ucc_liens`, `litigations`, `adverse_media`, `documents` | Adds negative-news screening and the formation documents themselves |

**This mapping is reasoning, not Middesk policy.** State which packages you are proposing and why
before ordering, and let the user adjust. A user with their own compliance policy should be able to
override it outright — take the list they give you.

## Step 2 — Identify or create the business

Ordering requires a `business_id`. Prefer an existing business over a duplicate.

1. Given a Middesk ID (a UUID), use it.
2. Given a name, call `list_businesses` with `q`. On several plausible matches, offer them as a
   choice the user picks from — name, ID, address, status — rather than guessing. Businesses in
   Middesk are routinely near-duplicates differing only by address or formation state.
3. Only when it genuinely does not exist, create it.

**Duplicates are common and worth naming.** Accounts accumulate many records for one business —
sometimes a dozen, sometimes created minutes apart with identical results. When you see that, say
so: it tells the user their next `create_business` would add another, and it affects which record
they should act on. Never quietly pick one duplicate over another, even when the difference looks
cosmetic; if two records are equally plausible, that is the user's call and `pageInfo.hasNextPage`
may be hiding more.

### Creating one

**Stop and confirm first — see the gate at the top of this skill.** Creating a business bills the
user for orders they did not ask for, so that confirmation is not optional, even when the request
sounds like a clear instruction to go ahead.

`create_business` requires `name` and at least one address; `tin`, `people`, `external_id`,
`unique_external_id`, and `tags` are optional and worth collecting, since richer input produces a
better verification result.

## Step 3 — Place the orders

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

<!--
TODO(FDE-84): delete this hardcoded package list once per-account order-type availability is
queryable. https://linear.app/middesk/issue/FDE-84

There is no API for it today: GraphqlTypes::QueryType exposes no packages/order-options field, and
PackageSetting (account_id + package_type + enabled) is not reachable via GraphQL.

When that ships, drop this table and the "illustrative" caveat above, and call the real tool.
Keep this list in step with ORDER_PACKAGES in middesk/mcp at src/worker/order-packages.ts until then.
-->

### The prerequisite chain

**Most packages depend on a `business_verification_verify` order.** No tool description says so, so
it has to be handled here.

Dependent packages include `documents`, `ucc_liens`, `litigations`, `adverse_media`, `bankruptcies`,
`tax_liens`, `liens`, `kyc`, `website`, `email_risk`, `enhanced_screenings`, and the `people_*`
products.

Before ordering any of them:

1. Call `list_orders` for the `business_id`.
2. Find the `business_verification_verify` order.
   - **Absent** → place it first, then place the requested packages.
   - **`created` or `pending`** → you can still place the orders. Middesk queues them and starts
     them automatically once verification completes. Tell the user they are queued and results will
     follow, rather than making them come back.
   - **`completed`** → go ahead as normal.
3. Place the requested orders.

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

So "onboard Acme and check its liens" will usually run against a pending verification. That is fine,
and it is one pass: create the business, place the orders, and tell the user they are queued and
results will follow.

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

## Step 4 — Answer the question that was asked

This is the step that makes the skill worth invoking. Do not stop at "orders placed."

Orders move `created` → `pending` → `completed`. Three further statuses exist but are not guaranteed
stops: `audited` appears only when an analyst reviews the order, and `approved` / `rejected` only
when a review decision follows completion. Do not wait for a status that may never arrive — treat
`completed` as the end state unless you see otherwise.

**When results are in**, call `retrieve_business` and read them against the decision from Step 1.

### Shape of the readout

Work through these four, then stop:

1. **The verdict** — one or two sentences answering the Step 1 question. Not "orders placed."
2. **What drove it** — only the findings that change the answer. A check that came back clean and
   was never in doubt does not need a line.
3. **What you could not establish** — gaps that bear on the decision: checks not run, results still
   pending, data the business does not have.
4. **One next step** — the single most useful thing to do next, not a menu of options.

**Aim for around 200 words** and treat that as a real budget, not an aspiration. A verification can
turn up a great deal that is true, interesting, and irrelevant to the decision in front of the
user; the discipline is leaving it out.

**Offer detail rather than including it.** "There are ten other records for this business — worth a
look?" beats three paragraphs about them. A finding that is interesting but does not change the
decision gets one line at most, or an offer.

Signs the readout has gone wrong: more than one table, section headings for asides, a numbered menu
at the end, or the user having to scroll before reaching the verdict.

### Reading the data

- Lead with the answer — is this business what it claims to be, is it safe to extend credit to —
  not with a data dump.
- `reviewTasks` with `status: failure` are the reasons a business is not verified. Name them in
  plain language: an unverified address and an SOS "Submitted Not Registered" mean something
  specific and worth saying.
- **`approved` is a review decision, not a clean result.** A business can sit at `approved` with
  failing review tasks, because a human approved it anyway. Read the tasks, not the status, and say
  so when the two disagree.
- **An inactive registration plus `Connections: Found` often means the wrong entity.** Businesses
  frequently trade under one entity while a sibling holds the active registration — an inactive
  `LLC` next to an active `L.P.` formed the same day, under the same people. When the SOS checks
  fail this way, search the account for related names before concluding the business is not in good
  standing. Onboarding the wrong entity of a real business is a different problem from onboarding a
  business that is not real, and the fix is different too.
- Flag what the checks could not establish, not just what they found. "No liens found" and "the
  liens search has not run yet" are very different answers to a credit question.
- If the policy called for checks the account could not run, say which, so the user knows the
  coverage they actually got.

**When results are still pending**, say exactly that in a couple of sentences — which orders are
placed, which are queued behind verification, and that `list_orders` shows progress. Do not imply a
clean result from an unfinished check, and do not pad a pending answer to look like a finished one.
