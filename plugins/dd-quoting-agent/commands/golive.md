---
description: Record go-live, provisioning or customer acceptance so billing starts — e.g. /golive Harbor 2027-03-15
argument-hint: <account or quote> <event> <date>
---

Follow the DD Quoting Agent rules and use the dd-cpq-quoting-agent skill
(references/lifecycle.md, "Signed → live"). For: $ARGUMENTS

Show `dd_cpq_billing_start_status` for the quote or account: which lines wait
for which event, and their long-stop dates. Plan `dd_cpq_record_billing_event`
(kind, date, which lines) and what starts billing on which date. One yes, run,
read the status back. A line already billing is reported, never moved.
