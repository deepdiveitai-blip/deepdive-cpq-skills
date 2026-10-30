# Intake: a messy ask into a priced quote (`/intake`)

The rep pastes what the customer sent: an email, a Slack thread, an RFP
table, a spreadsheet. You turn it into one quote plan, one yes, one build.
Every price comes from the engine; you never invent a product, a quantity or
a price.

## 1. Read the ask into a table

Show this table first, before any lookup, so the rep can correct it:

| #   | They asked for | Qty | When | Where (site, phase) | Notes |
| --- | -------------- | --- | ---- | ------------------- | ----- |

One row per thing asked for. Keep the customer's words in "They asked for".
Budget words ("we have $9k a month"), competitor words ("we're on X today")
and must-haves go under the table, not into rows. A quantity, date or site
that is not in the text stays blank: ask, never guess.

## 2. Find the account, opportunity and products

- `dd_cpq_find_accounts`, `dd_cpq_find_opportunities` (the opportunity must
  exist; never create one), `dd_cpq_find_quote` for an open Draft to add to.
- `dd_cpq_resolve_products` with every row at once (code or name per row).
  - exact: use the productId.
  - ambiguous: ask, listing the candidates with their codes. Never pick one,
    even when there is only one candidate.
  - missing: the row becomes a question ("we don't sell 'fax gateway'; did
    they mean …?"). Never create a product; that is an admin's job.
- `dd_cpq_quote_prechecks` on the opportunity and
  `dd_cpq_quote_creation_options` (which header fields show, which are
  required).
- `dd_cpq_check_compatibility` for the resolved products together;
  `dd_cpq_get_bundle_structure` for any bundle.

## 3. Plan

- One quote (`dd_cpq_create_quote` with `lines` for the standalone products
  and `details` for start and term), or the open Draft the rep names.
- Sites or phases: one Deal Block per site. Build the first site, then copy
  it for the later ones with `dd_cpq_clone_deal_block({startOffsetMonths})`;
  products only some sites get go in through `dd_cpq_apply_change_plan`
  (`block.save`, `block.move`) after the create.
- Bundles: `dd_cpq_configure_bundle` after the quote exists.
- After the quote exists, `dd_cpq_list_products({quoteId})` tells you what
  this customer may buy. A row whose product is not listed is "not quoted"
  with the eligibility rule's words; do not add it.

Show the plan with the ask table's row number on each step, the pre-check
result, list prices (`dd_cpq_pricing_view`), "net prices and ARR come with
the build", and "Go?". Budget words: say you will offer `/fit` after the
build; never discount to fit before the rep says so.

## 4. One yes, build, verify, report — "Intake: <account>"

1. **What they asked** (the table).
2. **What I built**: one row per ask row, mapped to its quote line, with the
   engine's net price and ARR.
3. **Not quoted, and why** (missing, ineligible, ambiguous).
4. **Questions for the customer** (blanks in the table, ambiguous rows).
5. **Totals**, the quote link, "say _undo_".

Then `dd_cpq_add_agent_note` on the quote: who you acted for and what was
built from which message.

## Example

Pasted: "We need 60 seats for HQ from Jan 1, the two clinics come 6 months
later with 20 seats each, add analytics for HQ only, we have $9k a month,
and we're on Competitor X today."

Table: 1 seats · 60 · Jan 1 · HQ; 2 seats · 20 · Jul 1 · Clinic 1; 3 seats ·
20 · Jul 1 · Clinic 2; 4 analytics · ? · Jan 1 · HQ. Budget $9k/month and
competitor under the table. Question: analytics quantity (per seat or one?).
Plan: quote with an HQ block (seats 60, analytics), clinic blocks cloned
6 months later with seats 20 each, then offer `/fit` to $9k.
