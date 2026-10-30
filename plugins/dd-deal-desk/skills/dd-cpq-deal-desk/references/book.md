# Many quotes and contracts at once

Jobs over a book of business. Reads first; anything that writes is one plan,
one yes, at most 10 records per run.

## The renewal book (`/renewal-book`)

What a renewals manager builds in a spreadsheet each month: every contract
ending in the window, the ARR at stake, who has a renewal quote, and what
the rules' uplift will bring. Then, after one yes, Draft renewal quotes for
the ones nobody started.

1. **The book.** `dd_cpq_renewals_due({days, scope, limit: 200})`, paging
   with `offset` until `page.total` is read. Window: what the person said,
   else 90 days. Scope: `org` for a manager who asks for everyone's (the
   answer's `canSeeAll` says whether they may; if not, say so and use their
   own book), `account` otherwise, or `userId` for one rep's book.
2. **Expected after uplift**, for the uncovered contracts with the most ARR
   at stake, at most 10: `dd_cpq_start_renewal({contractIds, dryRun: true})`
   gives what renews, what expires and why, and today's MRR. The uplift
   itself is priced only once a renewal quote exists: say "uplift priced
   when drafted" and never estimate it.
3. **Report, "Renewal book: next <days> days"**:
   1. Totals per currency (never added across currencies): contracts, ARR
      ending, covered, not started, ARR at risk.
   2. A table by end date: account, contract, ends, days left, ARR,
      renewal quote (name and status, or "not started"), auto-draft date.
   3. Needs a person: contracts with nothing renewing (all evergreen or
      one-time), a pending change still open (`dd_cpq_install_base` pending
      changes), assets ending that are on no contract
      (`unassignedAssetsEnding`).
4. **Offer**, never run unasked: "Draft renewal quotes for the N not
   started (up to 10)". After one yes, `dd_cpq_start_renewal({contractIds:
[one contract]})` for each, one at a time, without holds or discounts.
   Stop at the first refusal and say which were drafted. Nothing is
   accepted. Report each new quote as a link with its ARR after uplift
   (`dd_cpq_quote_snapshot`), and leave a note on each.
5. **Optional, when asked**: the price-increase note for each account, in
   plain words, from the drafted quote's numbers ("your renewal on 1 July
   2027 is $1,575 a month, up from $1,500: the 5% the contract allows").
   Never send it; the rep does.

## Price change impact (`/price-change-impact`)

Before or right after a price changes (a rate plan from $100 to $110, a new
pricing rule), every open quote that carries the product or plan, what each
would cost today, and a reprice of the ones the person picks.

1. **The change.** `dd_cpq_product_360` (the product's plans, price and
   rules) or the rate plan the person names. Say what changed, in their
   words and the plan's numbers.
2. **Who it reaches.** `dd_cpq_list_quotes({productId | ratePlanId})`: open
   quotes (Draft, Needs Review, In Review, Approved, Presented) with owner,
   status, expiry, lock and the matching lines' ARR. Page with `limit` when
   `truncated`.
3. **What each would move.** `dd_cpq_reprice_preview({quoteId})` for each
   quote, at most 10 at a time, biggest ARR first: lines changed, ARR
   before and after. It saves nothing and works on locked quotes too.
4. **Report, "Price change impact: <product or plan>"**:
   1. The change.
   2. Open quotes reached: quote (link), owner, status, ARR before → after,
      lines changed, locked or accepted.
   3. Contracts it reaches at renewal, for information only
      (`dd_cpq_renewals_due`): their renewal is priced from today's catalog
      or their uplift rule.
   4. Left alone, and why: locked, Accepted, no change.
5. **Offer**, never run unasked: "Reprice these K quotes to today", the
   person picks which, at most 10. After one yes,
   `dd_cpq_reprice_quote_to_today` for each, one at a time, skipping any
   that is locked or Accepted (say so). Each reprice gives the quote a new
   expiration date and brings an Expired one back to Draft: say that in the
   plan. Report each from its own `changes`, and leave a note on each.
