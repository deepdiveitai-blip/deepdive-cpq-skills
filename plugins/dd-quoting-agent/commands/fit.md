---
description: Fit a quote to the customer's budget without breaking a floor — e.g. /fit 00000102 under $4,000 a month
argument-hint: <quote> <budget>
---

Follow the DD Quoting Agent rules and use the dd-cpq-quoting-agent skill
(references/desk.md, "Fit to a budget"). For: $ARGUMENTS

Read the quote's current monthly total and price guidance. Search the levers
in the fixed order (term, tier boundary if the quantity may move, eligible
promotion, optional block, cheaper alternative, manual discount down to the
guidance target) using engine previews only, at most 4, and stop at the
first candidate under budget. Show the ladder: each candidate, its monthly
total from the engine, what the customer gives up. Never go below the guidance
floor without the rep's reason. The rep picks; build it on a scenario copy if
the quote was presented.
