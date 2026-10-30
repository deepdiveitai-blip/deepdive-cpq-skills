# Deal review (/deal-review)

A deal desk review of one quote before it is signed or sent: 30 numbered
checks, each **pass**, **warn**, **fail** or **n/a**, with the evidence and the
fix, and one verdict line. **Change nothing.** The only write is an offer the
person picks (a fix plan, submit for approval, a note), after a yes.

## 1. Find the quote

`dd_cpq_find_quote` (number, name or Id). Several matches: a numbered list.

## 2. Run the review in the engine

`dd_cpq_deal_review({quoteId})` runs all 30 checks below in DD CPQ and returns
each row (id, group, name, status, evidence, fix, blocking), the verdict
(`readyToSend`, `verdict`), the counts and the limits it used
(`thresholds`, from Settings › Deal Desk). Report it as it is: never re-judge
a row, never change a status, never add a check of your own. The Quote
page's Deal Review tab shows the same review.

Then read only what the report needs on top:

- `dd_cpq_quote_snapshot`, for the deal in one line (ARR, exit ARR, contract
  value).
- `dd_cpq_explain_line` for each line that failed check 9 or 10 (its
  discount stack).
- `dd_cpq_quote_changes_since` (30 days) for the five most recent changes.
- `dd_cpq_timeline_preview` invariants for check 22, which the engine leaves
  to the timeline: a failed invariant is a warning.

**Older package builds** answer that `dd_cpq_deal_review` is not available.
Then run the checks yourself from these reads, in parallel, once:
`dd_cpq_quote_snapshot`, `dd_cpq_get_quote_header`,
`dd_cpq_quote_expiration`, `dd_cpq_validate_quote`, `dd_cpq_price_guidance`,
`dd_cpq_list_deal_blocks`, `dd_cpq_get_ramp_segments`,
`dd_cpq_billing_start_status`, `dd_cpq_quote_changes_since`,
`dd_cpq_quote_prechecks`, `dd_cpq_list_available_promotions`,
`dd_cpq_list_contracts`, `dd_cpq_install_base`,
`dd_cpq_list_active_commitments`, and `dd_cpq_amendment_dates` for an
amendment. Use the shipped limits below and say "Policy: shipped defaults".

| Limit                                         | Shipped default                       |
| --------------------------------------------- | ------------------------------------- |
| Maximum manual discount on a line             | 20%                                   |
| Maximum total discount on a line (all stages) | 40%                                   |
| Term                                          | 12 to 60 months                       |
| Days left before the quote expires            | at least 7                            |
| Payment terms allowed                         | On receipt, Net 30, Net 45, Net 60    |
| Checks that block sending                     | 2, 5, 7, 8, 9, 11, 18, 19, 23, 25, 26 |

## 3. The checks (what the engine runs; the fallback follows the same rules)

**Group A — identity and state**

1. **Status reviewable.** Draft, Needs Review, In Review, Approved or
   Presented: pass. Accepted or Denied: every other check n/a; say so. —
   snapshot `quote.status`
2. **Quote Lock.** Locked: fail, name what is locked; the review is
   read-only either way. — snapshot `quote.lock`
3. **Right kind of quote.** A New Business quote on an account whose
   contract already covers the same product: warn, "should this be an
   amendment?". Amendment or renewal without a baseline: fail. — snapshot
   `transactionType`, `cartMode`, lines' `productId` vs `install_base` lines
4. **Header complete.** Start date, term, account, opportunity present;
   contact on the account. — `get_quote_header`
5. **Expiry.** Passed: fail. Fewer days left than the limit: warn. —
   `quote_expiration`
6. **Pre-checks.** Any failed Block pre-check on the opportunity: fail;
   Warn: warn. — `quote_prechecks`

**Group B — price and discount**

7. **Guidance.** A BelowFloor line without a reason: fail; with a reason:
   warn, quote the reason. BelowTarget lines: list with the win rate. —
   `price_guidance`, snapshot flag `guidance approval`
8. **Margin floor.** A line flagged `below floor`: fail. `floor approval`:
   warn, needs approval. — snapshot flags
9. **Manual discount.** A line whose manual discount is above the limit:
   fail. — snapshot `discounts` (the `manual` part); `dd_cpq_explain_line`
   on that line for the full stack
10. **Total discount.** All stages on a line above the limit: fail. —
    `dd_cpq_explain_line` for lines with two or more discounts in the
    snapshot. Never add percentages yourself: read each stage's value and
    the net price the engine gives against its list.
11. **Promotions.** A code on the quote that wins no line, or whose window
    ends before the quote expires: fail. — `list_available_promotions`
    (`isApplied`, `winsOnLines`, status)
12. **Partner tier.** n/a: the tier on the quote is reported; DD CPQ has no
    account field to compare it with.
13. **Currency.** A line flagged `currency fallback`: warn. — snapshot
14. **Trial lines.** A trial line without its paid line when the plan
    converts, or any discount on a trial line: fail. — snapshot `phase`,
    `trialStatus`, `discounts`
15. **Minimum commitment.** An active commitment on the account and the
    quote's MRR below it: warn. Compare the two engine numbers; never
    compute a new one. — `list_active_commitments`, snapshot `quote.mrr`

**Group C — term and time**

16. **Term.** Outside the limits: warn. — `get_quote_header.termMonths`
17. **Co-term.** New Business on an account with an active contract and no
    co-term mode set: warn, "independent by choice?". — `list_contracts`,
    `get_quote_header.coTermMode`
18. **Contract overlap.** A line selling a product the customer already owns
    while the owned line is still running (its end date after the quote
    line's start; no start date means owned now): fail. — `install_base`
    lines vs snapshot lines `start`/`end`
19. **Amendment dates.** An amendment line dated after `latestEnd`: fail
    (that is a renewal). — `amendment_dates`
20. **Ramps.** State each ramp: years, Year-1 vs exit ARR; a Manual ramp:
    warn. — `get_ramp_segments`
21. **Billing start.** A line waiting for an event with no expected go-live
    on the quote, or flagged `billing start approval`: warn. —
    `billing_start_status`, `get_quote_header.expectedGoLiveDate`, snapshot
22. **Invoice timeline.** Any failed invariant (TL-xx, SP-xx): warn, name
    it. — `timeline_preview.invariants`

**Group D — structure and validity**

23. **Commit checks.** `validate_quote` ok=false: fail, list each violation.
    Warning-only findings: warn.
24. **Bundles.** Option counts and required options: from the
    `validate_quote` findings on bundle lines.
25. **Blocks.** A pick-one block (`selectionMode` ExactlyOne) none of whose
    sub-blocks has `isSelected` true: fail, "no alternative chosen; the
    total counts <first alternative> by default". Optional blocks and lines
    flagged `optional`: list them as "not in the total". —
    `list_deal_blocks`, snapshot flags
26. **Approval coherence.** Any approval flag on a line while the status is
    Draft: fail ("needs submitting"). A Line or Save change after the last
    Approval in `quote_changes_since`: fail ("changed after approval").
27. **Changed since it was presented or approved.** List the changes after
    the last Approval (or the last 30 days): warn if any. —
    `quote_changes_since`

**Group E — downstream**

28. **Tax.** Calculated: pass. Out of date, failed or in progress: warn.
    Not calculated or not applicable: n/a. — Quote `TaxStatus__c`
29. **Payment terms.** A line whose rate plan's payment terms are not in the
    allowed list: warn. — the plan's `PaymentTerms__c`
30. **Opportunity sync.** Not the opportunity's syncing quote: warn.

A check whose tool is missing in this org's package build: n/a, "not
available in this build". A tool that fails: n/a with the error reference.

## 4. The report — "Deal review: <quote name> (<number>)"

1. **Verdict.** "Ready to send: **yes**" only when no check in the blocking
   list fails; else "Ready to send: **no**". Then "N fails · M warnings ·
   K passed · J n/a" and "Policy: shipped defaults" (or "Policy: Deal Desk
   settings").
2. **The deal in one line.** Account, type, start, term, ARR, exit ARR,
   contract value (engine numbers, one currency), status, approvals pending.
3. **Fails.** One row each: check number and name, what was found with the
   number, the fix, who does it.
4. **Warnings.** Same shape.
5. **Passed.** The check numbers only.
6. **Discount stack** of each line that failed 9 or 10, from
   `dd_cpq_explain_line`.
7. **What changed** (up to 5 most recent, from `quote_changes_since`).
8. **Offers**, never run unasked:
   - "Fix plan": a `dd_cpq_apply_change_plan` preview of the mechanical
     fixes only (remove a promotion that wins nothing, pick an option in a
     pick-one block, move a line to its block). Never lower a price or a
     discount to pass a check, never set a reason, never pick between real
     alternatives without asking.
   - `dd_cpq_submit_for_approval`, when an approval flag needs it.
   - `dd_cpq_add_agent_note` with the verdict.

Never a score. Never approve, reject or submit by yourself.
