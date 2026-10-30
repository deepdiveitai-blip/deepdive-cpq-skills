# Lifecycle: what a customer already owns

For the account manager: mid-term add-ons, upgrades and swaps, renewals with
uplift, signing, go-live, and contract restructures. Every change here is
written to the customer's install-base ledger when it is accepted, and that
ledger is never edited afterwards. Plan carefully, preview everything.

## Read what they own first

- `dd_cpq_list_contracts({accountId})`: contracts with dates, MRR and lines
  (bundle components under their header).
- `dd_cpq_install_base({accountId})`: everything they own, MRR, next renewal,
  changes still pending, recent changes. Each line carries the attribute
  answers they own (`attributeSummary`, e.g. "Deployment: Cloud · Warranty:
  36 months") and `attributeCharge`, what those added per unit as accepted:
  use them to explain a price or plan an upgrade.
- `dd_cpq_list_pending_changes({accountId, status?})`: the change ledger,
  newest first.
- `dd_cpq_contract_lineage({contractId})`: where a contract came from and
  where its lines went (merges, splits, transfers, rewrites).

## Mid-term change: an amendment

1. `dd_cpq_amendment_dates({accountId, contractId?})`: the latest date an
   amendment can start. After it, the change is a renewal.
2. `dd_cpq_swap_options({assetId})` if the rep wants to swap something (a
   bundle option for another option of its feature; another plan of the same
   product; another product).
3. `dd_cpq_start_amendment({accountId, contractId?, assetIds?, effectiveDate,
timing?, swaps?, opportunityId?})`:
   - scope: the account, one contract (`contractId`), or chosen assets
     (`assetIds`, each widened to its bundle). Several contracts ending on
     different dates need `contractId`.
   - `timing`: Immediate (prorates the first period), NextPeriod (starts at
     the next billing boundary) or AtRenewal.
   - `swaps: [{fromAssetId, toProductId, toRatePlanId?, priceBasis?, qty?}]`:
     the old line goes to 0, and the new one replaces it as a linked Swap
     pair (upgrade or downgrade). `priceBasis: "CarryDiscount"` keeps the old
     line's % off list (over 15% needs approval). A downgrade needs approval.
   - The quote opens with what the customer has today, every line tied to
     what they own.
4. Change the **future** position: `dd_cpq_update_line({quantity: 120})` means
   they will have 120, **not +120**. New products: `dd_cpq_add_line_item`.
   Removing a line ends that product, with the notice the lifecycle rules
   require.
5. `dd_cpq_amendment_removals({quoteId})`: what is coming off and when it
   ends. Put one back with `dd_cpq_restore_removed_line({quoteId,
baselineLineId})`, which keeps it tied to what they own. Never re-add the
   product with `add_line_item`: that sells it a second time.
6. Accept: **never** `status: Accepted` on an amendment. Use
   `dd_cpq_accept_amendment({quoteId})`. If the contract changed since the
   amendment started (drift), nothing is written: show `drift.lines`, and
   call again with `acceptDrift: true` only when the rep confirms the quote
   is still right.

### Several contracts at once (`/amend` v2)

"Harbor: +20 seats on contract A from Feb 1, Standard → Premium support on
contract B, drop the fax gateway." One plan, one amendment quote per
contract, one yes.

1. **What they own, per contract.** `dd_cpq_list_contracts({accountId})` and
   `dd_cpq_install_base({accountId})`. Map every asked change to one
   (contract, asset, action): add, change quantity, swap, remove. A change
   that matches no owned asset is a new product on the contract the rep
   names; when they name none, ask.
2. **Dates per contract.** `dd_cpq_amendment_dates({accountId, contractId})`
   for each contract touched. A change dated after that contract's
   `latestEnd` is a renewal: take it out of the plan and say "that one is a
   renewal (`/renew`)".
3. **Swap targets.** `dd_cpq_swap_options({assetId})` for each swap; the
   rep's words pick the target. Several candidates: ask.
4. **The plan**, one table, a row per change:

   | #   | Contract | Product | Now | After | From | Timing | Price basis | Approval |
   | --- | -------- | ------- | --- | ----- | ---- | ------ | ----------- | -------- |

   Timing is asked once for the whole plan when the rep did not say it
   (Immediate, NextPeriod or AtRenewal). Approvals the plan will hit: a
   downgrade, carrying a discount over 15%, a decrease over the lifecycle
   rule's limit, an early-termination fee. Then "Go?".

5. **One yes**, then per contract, in order:
   `dd_cpq_start_amendment({accountId, contractId, effectiveDate, timing,
swaps, opportunityId})`, then quantities as the future position with
   `dd_cpq_update_line`, new products with `dd_cpq_add_line_item`, removals
   by setting the quantity to 0, then `dd_cpq_amendment_removals` and
   `dd_cpq_validate_quote`. Stop at the first refusal: say which contracts
   are done and which are not.
6. **Report, "Amendment plan: <account>"**: what they own today per
   contract; the changes table; per contract the quote link, its lines with
   the quantity change, the monthly change and the first part-period charge
   (`dd_cpq_quote_snapshot` lines' `change`); approvals needed; and how to
   undo (each amendment quote is a Draft: leave it unaccepted, or the rep
   deletes it). A note on each quote.
7. **Accept later, one by one**, each with its own second yes, through
   `dd_cpq_accept_amendment`; on drift show what moved and ask again.

## Renewal

1. `dd_cpq_renewals_due` (rep-desk) or `dd_cpq_list_contracts`: which contract.
2. `dd_cpq_start_renewal({contractIds, exclude?, include?, quantities?,
termMonths?, opportunityId?, dryRun: true})` first: what renews, starting
   the day after the contract ends, priced from what the customer pays today.
   Evergreen and one-time lines never renew; a bundle renews whole.
3. Renewal uplift rules apply automatically (a RenewalUplift stage in
   `dd_cpq_explain_line`, e.g. +3% capped at 7%). "Hold the price on seats":
   `dd_cpq_update_lines({target: {productIds}, patch: {salesPriceOverride:
<the price they pay today>}})`, and say that it overrides the uplift.
   A concession on top of the held renewal price: `manualDiscountPct` or
   `manualDiscountAmount` — only those apply below it; system, promotion,
   channel and volume discounts stay aside (last term's price has them in).
   Read every `notKept` entry in the answer to the rep: that part did not
   save, and the entry says why.
4. Run it without `dryRun` after the yes. Accept with
   `dd_cpq_accept_amendment`: renewed lines go on a new contract, and lines
   not renewed expire.

### Renewal options (`/renew` v2)

The account manager wants priced options, not one renewal. The uplifted
prices exist only once the renewal quote does, so it takes two yeses: one to
draft the renewal, one for the option the rep picks.

1. `dd_cpq_start_renewal({contractIds, dryRun: true})`: what renews, from
   when, what expires or is skipped and why, today's MRR. Show it. Ask
   "Draft the renewal so I can price the options? Nothing is accepted."
2. **Yes 1.** The same call without `dryRun` (with any `exclude`, `include`
   or `quantities` the rep already asked for). The Draft carries the uplift
   the rules apply.
3. **The options**, each a `dd_cpq_apply_change_plan` preview on that Draft
   (saves nothing), at most four:
   - **A. As drafted**: the uplift as the rules apply it (read the Draft
     with `dd_cpq_quote_snapshot`; `dd_cpq_explain_line` names the
     RenewalUplift stage).
   - **B. Longer term**: the header term at 24 or 36 months.
     `dd_cpq_term_curves` says which terms earn a discount.
   - **C. Right-size**: lower quantities or remove lines the customer does
     not use (the rep says which; `dd_cpq_install_base` shows what they
     own).
   - **D. Hold the price**: `salesPriceOverride` at today's price on the
     lines the rep names; say it overrides the uplift.
     An option the engine refuses is shown as "not available" with its reason.
4. **The ladder**, one table:

   | Option | Year-1 ARR | Exit ARR | Contract value | What changes | Approvals |
   | ------ | ---------- | -------- | -------------- | ------------ | --------- |

   Engine numbers from each preview only. Then "Which one?".

5. **Yes 2.** Apply the chosen option's operations with
   `dd_cpq_apply_change_plan` (`mode: "apply"`, its `expectState`); A needs
   nothing. Verify from the save, report "Renewal options: <contract>"
   (what renews from when, the ladder, the chosen option built with ARR
   before and after against the last term, approvals, undo), and leave a
   note.
6. Accept later, after the customer signs, with `dd_cpq_accept_amendment`.

## Signed → live

- **Signing a new-business quote:** `status: "Accepted"` after two yeses
  (core skill). It creates the customer's assets and contract.
- **What waits to bill:** `dd_cpq_billing_start_status({quoteId})` (or
  `{accountId}` for every accepted quote with a line still waiting): each
  line's trigger, service start, billing start, long-stop date and state
  (PendingEvent, Scheduled, Billing).
- **Go-live / provisioning / customer acceptance happened:**
  `dd_cpq_record_billing_event({quoteId, kind, occurredOn, idempotencyKey,
lineIds?, productIds?})` after a yes. Every line waiting for that event
  starts billing on that date (never before its service start). The same key
  twice records once.
- **An event was missed or happened elsewhere:** `dd_cpq_set_billing_start({
quoteId, lines})`. The first billing start is final; a line already
  billing is reported, not moved. Before signature, change the line itself
  (`dd_cpq_update_line`) instead.

## Restructure: merge, split, transfer, rewrite

Needs the DD CPQ Engine Admin permission ("Administer Pricing"). A rep gets
403: hand it to the deal desk with the plan.

1. `dd_cpq_restructure_preview({preset: Merge|Split|Transfer|Rewrite,
contractIds, …})`: writes nothing. It shows every move, credits, charges,
   any early-termination fee, net new ARR, churn and bookings, and whether
   approval is needed. `REWRITE_IS_RENEWAL` means the date is past the
   contract end: renew instead.
2. `dd_cpq_restructure_start` with the same inputs after the yes, and after
   any approval is granted.
3. Accept with `dd_cpq_accept_amendment`. Undo with
   `dd_cpq_restructure_reverse({quoteId, dryRun: true})` first. It is refused
   once anything later touched those lines.

## Example

**Rep:** "Harbor wants 20 more seats from 1 February and to move from
Standard to Premium support, on the same contract."

Plan:

1. Amendment dates: the latest start is 2027-06-30, so 2027-02-01 is fine.
2. Swap options for the Standard Support asset: Premium is an option.
3. Start the amendment on contract A, effective 2027-02-01, Immediate, with
   the swap Standard → Premium (CarryDiscount).
4. Seats 50 → 70.
5. Validate.

Show the ARR change and the prorated first period. One yes, run, verify in
the cart (Swap pair, Increase on seats), then offer to accept (the drift
check runs then).
