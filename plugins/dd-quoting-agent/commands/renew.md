---
description: Renew a contract with the changes the customer wants — e.g. /renew Harbor 24 months, drop the fax gateway, hold the seat price
argument-hint: <account or contract> <what changes at renewal>
---

Follow the DD Quoting Agent rules and use the dd-cpq-quoting-agent skill
(references/lifecycle.md, "Renewal"). For: $ARGUMENTS

Resolve the contract. Run `dd_cpq_start_renewal` with `dryRun: true` (exclude,
quantities, term) and show what renews, from when, and the uplift the rules
apply. Plan any holds or discounts on top, with ARR before and after. One yes,
run, verify in the cart, report. Accepting it later is
`dd_cpq_accept_amendment`, after the customer signs.
