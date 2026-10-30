---
description: Change what a customer owns mid-term, across one or several contracts — e.g. /amend Harbor +20 seats on A from Feb 1, Standard → Premium support on B
argument-hint: <account or contract> <the changes>
---

Follow the DD Quoting Agent rules and use the dd-cpq-quoting-agent skill
(references/lifecycle.md, "Mid-term change", and "Several contracts at
once" when more than one contract is touched). For: $ARGUMENTS

Read what they own per contract and each contract's latest amendment date.
One plan table for every change (contract, product, now, after, from,
timing, price basis, approval), one yes, then one amendment quote per
contract with future quantities, never deltas. Report per contract with the
quantity and monthly change and the first part-period charge. Offer to
accept each with `dd_cpq_accept_amendment` (never Status = Accepted), one by
one; on drift, show what moved and ask again.
