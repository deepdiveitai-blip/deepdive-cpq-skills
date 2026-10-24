---
description: Contracts coming up for renewal on your book — e.g. /renewals 90 days
argument-hint: [days ahead, default 90]
---

Follow the DD Quoting Agent rules (in the dd-cpq-quoting-agent skill) and use the dd-cpq-rep-desk skill. Call
`dd_cpq_renewals_due` for the window the rep gave ($ARGUMENTS, or 90 days)
and list the contracts by end date: account, what they own, ARR, end date,
and whether a renewal quote exists. Offer to start a renewal for one (always
with `dryRun: true` first). Change nothing until the rep says which.
