---
description: Build another version of a quote on a copy, then compare — e.g. /scenario 00000088 as a 36-month ramp
argument-hint: <quote> <what the version changes>
---

Follow the DD Quoting Agent rules and use the dd-cpq-quoting-agent skill
(references/structure.md, "Versions"). For: $ARGUMENTS

Resolve the quote. Copy it with `dd_cpq_create_quote` (`startFrom: {kind:
"copy", quoteId}`, name "<quote name> — scenario: <what> (Claude)") and plan
the change on the copy, never on the original. One yes, run, verify the copy
in the cart. Then compare original and scenario side by side: lines, ARR,
Exit ARR, contract value (one currency). Ask which one wins. Sync the winner
to the opportunity only after a yes. Offer to discard the losing scenario
(only one you created).
