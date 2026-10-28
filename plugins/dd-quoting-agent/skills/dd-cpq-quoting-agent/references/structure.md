# Structure: shaping a big deal

For the enterprise rep: several sites or phases, multi-year ramps, pick-one
options for procurement, add-ons that must end with the existing contract,
partners, other currencies, and versions for the exec review. These deals
take 20 to 200 lines; plan them as one job.

## Deal Blocks: sites, phases, sections, alternatives

A Deal Block is a named part of a quote. A block can have its own start date
and term, a discount on every line in it, an optional flag (priced and shown,
left out of the total), and pick-one sub-blocks (alternatives).

| Need                                                                              | Tool                                                                                                                                                                                                  |
| --------------------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| See the blocks and their subtotals (net, MRR, ARR, Exit ARR, contract value)      | `dd_cpq_list_deal_blocks({quoteId})`                                                                                                                                                                  |
| Create or change a block                                                          | `dd_cpq_save_deal_block({quoteId, blockId?, name, kind: Section/Phase/Site/Scenario/Other, parentBlockId?, startDate?, termMonths?, isOptional?, selectionMode?, additionalDiscountPct?, location?})` |
| Put lines in a block (a bundle, all ramp years and sibling charges move together) | `dd_cpq_move_lines_to_block({quoteId, qliIds, blockId})` (null = ungrouped)                                                                                                                           |
| Add products straight into a block                                                | `dd_cpq_add_products_to_quote({…, blockId})` / `dd_cpq_add_line_item({…, blockId})`                                                                                                                   |
| Copy a site to make another, later                                                | `dd_cpq_clone_deal_block({quoteId, blockId, name, startOffsetMonths})`                                                                                                                                |
| Let DD CPQ propose a split (by window, location, charge, family; fixed rules)     | `dd_cpq_suggest_deal_blocks` → `dd_cpq_materialize_deal_blocks({by})`                                                                                                                                 |
| A reusable layout ("3 phases, 6 months apart")                                    | `dd_cpq_list_block_templates` → `dd_cpq_apply_block_template({quoteId, templateId})`, or `startFrom: template` on create                                                                              |
| Save this quote's layout for next time (rep-owned)                                | `dd_cpq_save_block_template({fromQuoteId, previewOnly: true})` first                                                                                                                                  |
| Remove a block                                                                    | `dd_cpq_delete_deal_block({quoteId, blockId, deleteLines?})` (lines stay unless deleteLines)                                                                                                          |

**Pick-one alternatives** ("Standard or Premium support, procurement
decides"): a parent block with `selectionMode: "ExactlyOne"` and one
sub-block per alternative. `dd_cpq_compare_alternatives({quoteId, blockId})`
prices each in full; `dd_cpq_select_alternative` picks the one that counts.
The others stay on the quote at zero in the totals.

**Discount a whole block**: either `additionalDiscountPct` on the block
(taken after line discounts), or `dd_cpq_update_lines({target: {blockId},
patch: {manualDiscountPct}})` on its lines. Say which you used; the net is
the same, but the waterfall shows a different stage.

**Bulk changes** (`dd_cpq_update_lines`): target by `qliIds`, `all`,
`productIds`, `blockId`, `family` or `parentQliId`; filters combine. Always
`dryRun: true` first. More than 25 lines: ask a second time, then send
`confirmManyLines: true`.

## Ramps: different quantities or prices per year

A multi-year deal with a ramp rule is saved as **one line per contract year**:
three lines for 36 months is correct, not a duplicate.

- `dd_cpq_get_ramp_segments({quoteId})`: each ramp's years, Rule or Manual,
  `totalArr` (Year 1) and `exitArr` (final year).
- `dd_cpq_preview_ramp_segments({quoteId, mode: propose|set|reset, lines})`:
  price years without saving.
- `dd_cpq_set_ramp_years({quoteId, qliId, mode: "set", years: [{year,
quantity?, price?, discountPct?}], dryRun})`: save edited years (the ramp
  becomes Manual); `mode: "reset"` goes back to the rule.
- Always quote three numbers: **ARR (Year 1)**, **Exit ARR** (final year),
  **contract value** (all years).

## Term length and the timeline

- A longer term can earn a term discount (a term curve, e.g. 36 months → 10%).
  Change `termMonths` on the header or the line and read the TermDiscount
  stage in `dd_cpq_explain_line`.
- `dd_cpq_timeline_preview({quoteId})`: what the customer is invoiced per
  period (first month, steady state, contract value, trial savings, spread
  fees). Show it whenever there is a trial, ramp, prepay, spread fee,
  proration or a billing start after the service start.

## Ending with the existing contract (co-term)

- `dd_cpq_list_contracts({accountId})`: the customer's contracts and end dates.
- Join a new-business quote to a contract:
  `dd_cpq_update_quote_header({contractId})`. The deal is co-termed to that
  contract's end.
- `coTermMode`: `AlignToDealEnd` shortens lines to end with the deal;
  `ExtendDealToLongest` only **proposes** stretching the deal to the longest
  line and never changes it by itself; `Independent` leaves every line on its
  own dates. One line can opt out with `independentTerm: true`.
- A co-termed line shorter than its plan's minimum term is raised to the
  minimum at commit: say so.
- A Deal Block with its own start date is its own window: its lines end at
  the block's start plus the block's term (the deal's term when the block has
  none), and `AlignToDealEnd` does not pull them back to the deal's end. To
  end a later site with the deal, give the block a term
  (`dd_cpq_save_deal_block({termMonths})`): Clinic East from 2027-07-01 in a
  deal ending 2029-12-31 needs 30 months.

## When billing starts

The term runs from the service start; billing can start later.
`dd_cpq_update_line({qliId, billingTrigger, …})`:

| Trigger                                    | Billing starts                        | Also send                                                                                |
| ------------------------------------------ | ------------------------------------- | ---------------------------------------------------------------------------------------- |
| ServiceStart (default)                     | at service start                      |                                                                                          |
| Signature                                  | later of service start and acceptance |                                                                                          |
| FixedDate                                  | on a date                             | `billingStartDate`                                                                       |
| Offset                                     | N days after service start            | `billingOffsetDays`                                                                      |
| Provisioning / GoLive / CustomerAcceptance | when that event is recorded           | `longStopDays` (billing starts anyway after N days), optional `expectedBillingStartDate` |

Set it on the leader line: bundle options, hybrid siblings and later ramp
years follow it. Header defaults: `expectedGoLiveDate`, `longStopDays`.
`billingStartOverridden: false` hands the line back to the defaults.

**Spread a one-time fee** (e.g. $12,000 implementation over 12 months):
`dd_cpq_update_line({qliId, amortization: {months: 12}})`; `null` stops it.
Show the timeline after.

## Partner (channel) deals

`dd_cpq_update_quote_header({partnerTier: "Gold" | "Silver" | "Bronze" |
"Distributor"})`: every price then applies the active Channel Discount rules
for that tier (after promotions, before volume). `null` is a direct sale. Read
the ChannelDiscount stage in `dd_cpq_explain_line`; the margin floor still
applies.

## Usage, prepaid credits and minimum commits

- A usage line has MRR $0 by design; its ARR is a projection from expected
  usage. Say so every time.
- Tiered and percent-of-basis plans price from their tiers or basis; read
  the waterfall's UsageTierPrice stage.
- `dd_cpq_list_prepaid_credit_pools`: the credit pools a usage plan can draw
  from. `dd_cpq_prepaid_credit_simulate({quoteId, deposits?,
hypotheticalUsage})`: how long the credits last at the customer's usage.
- `dd_cpq_list_active_commitments({accountId})` and
  `dd_cpq_evaluate_commit_floor({accountId, hypotheticalUsage})`: an existing
  minimum commit and the shortfall at a given usage. These are what-ifs:
  floors are a billing rule, never applied to the quote's price.

## Other currencies

The quote's currency follows the account. Each line uses a rate plan in that
currency; with none, the engine uses another currency's plan as is and flags
the line `currencyFallbackApplied` (amber badge in the cart). Never add
amounts in different currencies together; report each currency separately.
Missing plans are added by an admin (dd-cpq-rate-plan-authoring), not by
you.

## Versions: scenario quotes

"Show me a 36-month version", "what if they take Premium", "a cheaper
option for the CFO": build it on a copy (see the core skill), compare,
and let the rep pick. To compare, read both with `dd_cpq_quote_snapshot`
and show the lines and totals side by side (one currency). The winner gets `dd_cpq_sync_quote_to_opportunity`.

## Example

**Rep:** "Riverside expansion: three sites, HQ first, the clinics 6 and 12
months later, seats ramp 100/150/200, and end it with their current
contract."

Plan:

1. Apply the "three sites" template, or create Site HQ.
2. Add seats to HQ.
3. Ramp years: 100 / 150 / 200.
4. Clone HQ as Clinic North (+6 months) and Clinic South (+12 months).
5. Join the contract (contractId), with AlignToDealEnd.
6. Preview the timeline.

Report ARR (Year 1), Exit ARR and contract value per block and in total.
One yes, run, verify the cart's block totals.
