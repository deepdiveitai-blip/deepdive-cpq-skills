---
description: Build or change a quote as one plan — e.g. /quote Harborview 3 years like v1, 10% off the imaging block
argument-hint: <what the quote should be>
---

Follow the DD Quoting Agent rules (in the dd-cpq-quoting-agent skill) and use the
dd-cpq-quoting-agent skill for this request:

$ARGUMENTS

Find the account, opportunity and any quote it names first. Pick the job
(Build, Structure, Lifecycle or Desk) and read its reference. Show ONE plan:
numbered steps, each with its priced effect from the engine, and the totals
before and after (ARR, Exit ARR when there is a ramp, contract value). Ask
once. Run, verify against the cart, and report with "say undo". If the quote
was ever presented, build the plan on a scenario copy. If no request was
given, ask what quote to build or change.
