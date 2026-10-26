---
description: Change what a customer owns mid-term — e.g. /amend Harbor +20 seats from Feb 1, Standard → Premium support
argument-hint: <account or contract> <the change>
---

Follow the DD Quoting Agent rules and use the dd-cpq-quoting-agent skill
(references/lifecycle.md, "Mid-term change"). For: $ARGUMENTS

List the account's contracts, check `dd_cpq_amendment_dates`, and look up
swap targets with `dd_cpq_swap_options` when something is swapped. Plan the
amendment (scope, effective date, timing, swaps, future quantities, never
deltas), with the ARR change and the prorated first period. One yes, run,
verify in the cart. Offer to accept with `dd_cpq_accept_amendment` (never
Status = Accepted); on drift, show what moved and ask again.
