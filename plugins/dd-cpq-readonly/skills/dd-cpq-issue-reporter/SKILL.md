---
name: dd-cpq-issue-reporter
description: >-
  File a DD CPQ problem as a Jira Bug with every technical detail attached.
  Takes the DD-XXXXXX reference a user was shown (or a quote), pulls the
  issue packet from the org with dd_cpq_issue_packet (error, stack, limits,
  the run's pipeline flow, rules fired, quote lines and waterfalls,
  settings, build), checks Jira for the same root cause first, and either
  adds a "seen again" comment to the existing issue or drafts a new Bug,
  shows it, and creates it only after the person says yes. Trigger on:
  "file this", "raise a Jira for DD-", "log this bug", "report this error",
  "create a defect from this reference", "I hit an error on a quote". Skip
  for working or fixing an existing Case or Jira (dd-cpq-case-resolver),
  exploratory testing (dd-cpq-pricing-tester, dd-cpq-configurator-tester)
  and story research (dd-cpq-story-research).
---

# Filing a DD CPQ issue from a reference

Every error DD CPQ shows ends with a reference like `DD-7F3K2Q`. That
reference is the key to everything the org recorded when it happened. Your
job is to turn it into a Jira Bug a developer can act on without asking a
single follow-up question, and to never file
the same root cause twice.

Nothing in this skill changes Salesforce. You read from the org through the
dd-cpq MCP tools and write only to Jira, and only after the person agrees.

## 1. Get the reference

Ask for the `DD-XXXXXX` reference if the person has not given one. It is
in the error message, in the cart's red banner, in the Run meter header, and
in Engine Health → Errors. If they have no reference, ask for the quote and
what they did (commit, reprice, add a product), and use the quote instead.

Check you are connected to the right org first: `dd_cpq_list_orgs`. The
packet is only as good as the org it came from.

## 2. Pull the packet

```
dd_cpq_issue_packet { reference: "DD-7F3K2Q" }
```

or `{ quoteId: "0Q0..." }` when there is no reference. The packet has:

| Section                                           | What it tells you                                                                                                                            |
| ------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------- |
| `error`                                           | code, message the user saw, detail, stack trace, stage, line (`localKey`), rule, limits at that moment, how many errors share the root cause |
| `run`                                             | the pipeline step by step: queries/CPU per step, rules applied/warned/blocked per step, the error pinned to its step                         |
| `quote`, `lines`                                  | header and every line with list, net, rate plan and price waterfall                                                                          |
| `rules`                                           | each rule involved, as the sentence Rules Studio shows                                                                                       |
| `settings`, `about`                               | every DD CPQ setting, package version, org, sandbox or not                                                                                   |
| `suggestedTitle`, `suggestedLabels`, `reproSteps` | your starting draft                                                                                                                          |

If it answers "No error is logged under …", the reference is older than the
retention period or the error log is off (Settings → Engine Health). Say so
and fall back to the quote.

## 3. Look for the same root cause in Jira first

`suggestedLabels` contains `ddfp-<12 hex>` — the root-cause fingerprint
(code + stage + message with ids and numbers taken out). Search for it:

```
searchJiraIssuesUsingJql:
  project = DDCPQ AND labels = "ddfp-abcdef012345" ORDER BY created DESC
```

- **Found, still open** → do not create a new issue. Add a comment:
  "Seen again — DD-XXXXXX on <quote name> by <user> at <time>. Same root
  cause, now N occurrences (packet: error.sameRootCause)." Tell the person
  the existing key and stop.
- **Found, Done** → it came back. Create a new Bug (step 4) and link it
  to the old one ("relates to"), saying "regression of DDCPQ-nn".
- **Not found** → step 4.

No `ddfp-` label (a packet by quote with no error) → search by summary text
instead and use judgement; say what you searched.

## 4. Draft the Bug and show it

Build the draft; do not post it yet.

- **Project** DDCPQ · **Type** Bug
- **Summary** `suggestedTitle`, edited to read naturally (keep the code)
- **Labels** every entry of `suggestedLabels`
- **Description** (Markdown):

````
## What happened
<one or two sentences in the person's words, plus the message they saw>

## Reference
DD-XXXXXX · trace <traceId> · <org> · package <about.packageVersion>

## Steps to reproduce
<reproSteps, then anything the person adds>

## Expected / actual
Expected: <what should have happened>
Actual: <error.code> — <error.message>

## Where it broke
Stage <error.stage>, line <error.localKey>, rule <rule sentence if any>.
<the run's steps that had rules or errors, one line each>

## Issue packet
<details><summary>Full packet (JSON)</summary>

```json
<the packet>
````

</details>
```

If the packet is too long for the description (Jira's limit is about
32,000 characters), put the error, run and rules sections in the
description and say "lines and settings trimmed; pull the full packet with
dd_cpq_issue_packet reference=DD-XXXXXX".

**Show the person the summary, labels and the first sections, and say the
packet includes product names and prices.** Create nothing until they say
yes. If they want changes, make them and show it again.

## 5. Create it

On yes: `createJiraIssue` with the draft, in status **To Do**. Never move it
to **DEV Ready**: that status means "Venkat has read this and it is ready
to build", and only he sets it. The `dd-cpq-dev-ready` skill builds what he
puts there, when he asks.

Reply with the new key and its link, and whether it was a duplicate.

## Things not to do

- Do not post before the person agrees, even if they said "just file it"
  earlier in the conversation — show the draft once.
- Do not invent a reference, an id or a stack trace. If the packet lacks
  something, the Jira says it is missing.
- Do not paste customer data beyond what the packet already carries.
- Do not create a Case in Salesforce for this; tester Cases are a separate
  process (dd-cpq-case-resolver).
