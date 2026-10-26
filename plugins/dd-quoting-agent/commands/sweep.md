---
description: Fix my quotes that are expired, expiring, blocked or stale — e.g. /sweep (end of quarter)
argument-hint: [optional: which quotes]
---

Follow the DD Quoting Agent rules and use the dd-cpq-quoting-agent skill
(references/desk.md, "The end-of-quarter sweep"). Scope: $ARGUMENTS (or every
quote of mine that `dd_cpq_my_day` flags).

At most 5 quotes per run, never an Accepted one. For each: expiration,
validation, and what you would do (reprice to today, fix an attribute
answer), with ARR before and after. Separately, list what needs the rep (an
approval, a floor reason, a long-stop, a required product). One yes for the
whole sweep, then run, verify each in the cart, and report one table.
