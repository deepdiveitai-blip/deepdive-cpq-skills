# Desk: explain, check, fix, fit

For the deal desk and the sales engineer, and for any rep the day a quote
won't commit, a customer asks "why $X", the quarter ends, or the budget is
fixed.

## "Why is this line $X?"

- `dd_cpq_explain_line({quoteId, qliId})`: one line's waterfall (PricebookList
  → … → Net), the rules that moved it, a plain-English narrative. Prefer it on
  a big quote.
- `dd_cpq_explain_price({quoteId, promoCode?, partnerTier?})`: every line at
  once, so it is large; use it for "what if this code / tier were on the
  quote". For one line, `dd_cpq_explain_line`.
- `dd_cpq_rules_fired({quoteId})`: every rule that applied, blocked, warned or
  lost to a higher one, with the sentence Rules Studio shows.
- Name only the stages that moved the price, one line each, in order, with
  the rule's name: `Seats ×100: list $100 → Term −10% (36 months) → Volume
−15% (100+) → net $76.50`. Never reconstruct it yourself.
- Product questions: `dd_cpq_product_360` (one product: plans, rules,
  bundles, sellability), `dd_cpq_pricing_view({productId})` (its plans and
  prices). Admin-wide questions (`dd_cpq_browse_products`) are for admins.

## "Will this commit?"

- `dd_cpq_validate_quote({quoteId})`: everything commit would refuse or warn
  about: required companions, exclusions, Requires rules, attribute findings.
  Blocking ones first, each with its fix.
- `dd_cpq_quote_snapshot({quoteId})`: what the screen shows. Compare
  quantities with the quantity rules and parentage with the bundle. A difference between engine and screen is a finding: report it.
- `dd_cpq_quote_expiration({quoteId})`: is it past its date, and does that
  block a commit (enforcement Block)?
- `dd_cpq_reprice_quote_to_today({quoteId})` (a write, after a yes): reprices
  with today's rules, gives a fresh expiry, and brings an Expired quote back
  to Draft. It returns exactly what moved, per line and stage, and why.

## The end-of-quarter sweep (`/sweep`)

1. `dd_cpq_my_day`: the rep's quotes with `reasons` (expired, expiring,
   awaitingApproval, flaggedLines).
2. For each one (at most 5 quotes per run):
   - `dd_cpq_quote_expiration`
   - `dd_cpq_validate_quote`
   - if expired or stale, `dd_cpq_reprice_quote_to_today` as a planned step.
3. Plan: one row per quote. Show what you will do, ARR before and after, and
   what still needs a human (an approval, a floor reason, a missing
   long-stop, a required product).
4. One yes for the routine, then run, verify each with what its save
   returned, and report the table.

Never touch an Accepted quote. Never add a required product or a floor
reason without the rep: list it.

## Fit to a budget (`/fit`)

"Their budget is $4,000 a month: get this under it without touching the
floor."

The engine stays the only calculator. Search over engine previews in this
fixed order, and stop at the first candidate under budget:

1. **Term:** a longer term, if a term curve rewards it (preview the header
   term).
2. **Quantity:** quantities at a volume-tier boundary, only if the rep says
   the quantity can move.
3. **Promotion:** an eligible one (`dd_cpq_list_available_promotions
onlyEligible`).
4. **Optional:** make the services/training block optional (left out of the
   total).
5. **Alternatives:** the cheaper alternative of a pick-one block
   (`dd_cpq_compare_alternatives`).
6. **Discount:** a manual discount, down to the guidance target, never below
   the floor without the rep's reason.

At most 4 previews (`dd_cpq_update_lines dryRun`, `dd_cpq_preview_line_change`,
`dd_cpq_compare_alternatives`). Show the ladder: each candidate, its monthly
total from the engine, what the customer gives up. The rep picks. Build it on
a scenario copy if the quote was presented.

## Approvals and errors

- A line flagged for approval (guidance, margin floor, billing start): say
  which and why, then offer to submit (build reference).
- An error reference DD-XXXXXX: give it to the rep and stop. Admins can read
  it with `dd_cpq_get_error` / `dd_cpq_issue_packet`; a rep gets 403.

## Preferences (the rep's own screen)

- `dd_cpq_get_header_preferences` / `dd_cpq_set_header_preferences`: which
  Quote fields the rep sees in the cart header.
- `dd_cpq_get_picker_preferences` / `dd_cpq_set_picker_preferences`: which
  filters show in the Add Products window.

Both affect only the rep.
