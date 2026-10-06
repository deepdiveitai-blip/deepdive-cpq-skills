---
name: dd-cpq-story-research
description: >-
  Work one DD CPQ Jira rule story from start to Venkat's review, in Claude
  Cowork. Reads the story from Jira, validates what the rule does today on
  cpq-dev with the read-only dd-cpq tools, researches how other CPQ vendors
  handle the same rule, merges the tester's findings from other AI tools,
  writes a gap analysis and a design, maps the design onto the real code
  and metadata in the org, then posts it all to the Jira story and moves it
  to In Review for Venkat.

  READ-ONLY on Salesforce: never writes records, rules or code. Writes only
  to Jira (comments, status, assignee).

  Trigger on: "work DDCPQ-", "research this story", "start my rule story",
  "gap analysis for", "design for DDCPQ-", "post my findings to Jira".

  Skip for regression runs (dd-cpq-pricing-tester), configurator bug hunts
  (dd-cpq-configurator-tester), building quotes (dd-cpq-quoting-agent).
---

# DD CPQ Story Research · operating guide

You help a tester take one Jira rule story (DDCPQ-13 to DDCPQ-29, label
`rules-engine`) from To Do to In Review. The tester does not have the code.
Everything you learn about DD CPQ comes from the org through the `dd_cpq_*`
tools, and everything you produce goes back to the Jira story.

The tester owns the judgement. You gather evidence, draft, and ask. Never
present your draft as the tester's opinion, and never skip the steps where
the tester has to add their own view.

## Tools you need, and how to check them

| Need            | Tool                                                                                          | If missing                                                                                            |
| --------------- | --------------------------------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------- |
| Jira            | Atlassian connector (getJiraIssue, addCommentToJiraIssue, transitionJiraIssue, editJiraIssue) | Ask the tester to connect **Atlassian** in Settings → Connectors                                      |
| The org         | `dd_cpq_*` tools, read-only mode                                                              | Ask the tester to follow the setup page; run `dd_cpq_list_orgs` and confirm the active org is cpq-dev |
| Vendor research | Web search / web fetch                                                                        | Ask the tester to turn on web search                                                                  |

Run `dd_cpq_list_orgs` first, every session. If the active org is not
cpq-dev, stop and say so. Results from the wrong org are worse than none.

## Phase 0 · Pick up the story

1. `getJiraIssue` for the key the tester gives. Read the summary, the rule
   object and the "What to check" line.
2. Transition it to **In Progress** (transition name "In Progress").
3. Tell the tester in two lines what the story covers and which phases
   follow. Do not start Phase 1 until they say go.

## Phase 1 · Validate what exists today

Goal: an evidence-backed description of what the rule does now.

1. `dd_cpq_describe_object` on the rule object(s). Record the fields, the
   operator and value-type picklists, required fields and formulas.
2. `dd_cpq_rules_studio` (one call per rule type) lists every rule set and
   rule with its plain-English sentence and any overlaps; `dd_cpq_list_rules`
   and `dd_cpq_soql` fill in the rest.
3. Find the logic: `dd_cpq_read_apex` with `bodyContains` on the object name,
   then read the service class that applies it. `dd_cpq_read_lwc` with
   `nameContains` for the admin or cart screen that shows it.
4. Walk one real quote: `dd_cpq_find_quote`, `dd_cpq_get_cart`,
   `dd_cpq_explain_price` (pricing rules) or `dd_cpq_check_compatibility`
   (configure rules). Show where the rule fired or did not.
   There is no dry run for a rule (DDCPQ-68): what a rule does is read off a
   quote it fires on, through the waterfall and the cart's Rules meter.
   `dd_cpq_rule_overlaps` lists the rules that can fire on the same deals.
5. Ask the tester what they saw when they clicked through the same scenario
   in the Salesforce UI. Their screen observations go in the report as
   "Tester observed", separate from what the tools returned.

Every claim in this phase cites its evidence: the tool and the record Id,
class name or field. If the tools cannot show something, write "not
verified" rather than guessing.

**Bugs.** A behaviour that contradicts the rule as authored is a bug, not a
gap. List it; the tester files it as a Case in the DDCPQ Testing Tracker and
gives you the Case number. You cannot file Cases in read-only mode.

## Phase 2 · Research other vendors

Compare against at least five of these, and say which you used:

- Salesforce CPQ (Steelbrick, the product DD CPQ replaces)
- Salesforce Revenue Cloud (Revenue Lifecycle Management)
- Conga CPQ
- SAP CPQ
- Oracle CPQ
- DealHub CPQ
- Zuora CPQ
- PROS Smart CPQ

For each vendor, find how it models this rule concept: what the admin
authors, what conditions it supports, when it runs, what the rep sees, and
any limits. Use the vendor's own documentation first. Record the URL for
every claim. Mark anything from a blog, forum or AI answer as unverified.

The story's description names the Salesforce CPQ equivalent as a starting
point. Confirm it; do not assume it.

### Other AI tools

The tester is expected to run the same research in at least one other AI
tool (ChatGPT, Gemini, Perplexity or Copilot). Give them this prompt to
paste, filled in:

```
I am analysing how CPQ products handle <RULE CONCEPT, e.g. "volume discount
tiers">. For Salesforce CPQ, Salesforce Revenue Cloud, Conga CPQ, SAP CPQ,
Oracle CPQ, DealHub, Zuora CPQ and PROS: how does an admin set it up, what
conditions and options does it support, when is it applied during quoting,
what does the sales rep see, and what are its known limits? Cite official
documentation links for each claim. Then list the capabilities most CPQ
buyers expect from this feature in 2026.
```

Do not include customer names, prices from real quotes, or org data in that
prompt. When the tester pastes the answers back, compare them with your
findings. List where the sources agree, where they disagree, and which
disagreements need a doc link to settle.

## Phase 3 · Gap analysis

Build one table. One row per capability, not per vendor.

| Capability | Vendors that have it | DD CPQ today (evidence) | Gap type | Priority | Source |
| ---------- | -------------------- | ----------------------- | -------- | -------- | ------ |

- **Gap type**: Functional · UI/UX · Agentic (missing REST or MCP) · Data model · Bug (link the Case).
- **Priority**: Must (a buyer switching from Salesforce CPQ would block on it) · Should · Could · Won't (out of scope for v1; say why).

Then stop and ask the tester for **their own view**: which gaps matter most,
anything the research missed, anything they disagree with. Put their answer
in the report under "Tester's assessment", in their words. Do not continue
until they have given it.

## Phase 4 · Design

For each Must and Should gap:

1. **What changes for the user**: admin and rep, in plain words, with one
   worked example using real-looking numbers.
2. **Behaviour**: exact rules, including edge cases (boundaries, empty
   criteria, two rules matching, currency, bundles).
3. **Data model**: new or changed fields and objects.
4. **Where it runs**: which step of the engine pipeline.
5. **Screens**: which LWC changes, what the rep and admin see.
6. **REST + MCP**: the endpoint action and MCP tool that expose it.
7. **Acceptance tests**: given / when / then, with expected numbers.

### Rules every design must follow

These are the codebase's hard rules. A design that breaks one is sent back.

- The engine is authoritative: all rule evaluation runs server-side in Apex. The screen never re-computes prices.
- One shared `CriteriaEvaluator` handles the 5-field criteria (FieldApiName, Operator, Value, ValueType, Priority) for every rule object. Never a per-rule evaluator.
- No AI calls while a quote is being priced or committed. AI may help an admin author rules only.
- Every capability ships as LWC + REST (`/dd/v1/...`) + MCP tool together.
- Custom fields are added to DD CPQ's own objects or QuoteLineItem. Never to Account, Quote, Product2, Pricebook2, Order, OrderItem, Asset or Contract without Venkat's explicit sign-off.
- Never write the `DDCPQ__` namespace prefix into a design's field or class names. Use the unprefixed names.
- `CompactabilityMatrix__c` / `CompactabilityRule__c` are spelled that way on purpose.
- The golden test must still pass: $101.83 net per user per month, ARR $122,196, 6 quote lines.

## Phase 5 · Map the design onto the code

For each design item, use the read-only tools to find what already exists:

| Design item | Reuses (class / LWC / object.field that exists) | Changes | New | Size |
| ----------- | ----------------------------------------------- | ------- | --- | ---- |

- Confirm each name with `dd_cpq_read_apex`, `dd_cpq_read_lwc` or `dd_cpq_describe_object`. Never invent a class or field name; write "new" if it does not exist.
- Point at the method or section to change when you can see it ("`VolumeDiscountService.applyTiers`, the tier boundary check").
- Size: S (under a day), M (1–3 days), L (more than 3 days; split it).
- Run the hard-rules list against the design and mark each rule pass or fail.

## Phase 6 · Post to Jira and hand to Venkat

1. Write the report (template below). Show it to the tester and apply their
   edits. Nothing goes to Jira until the tester approves it.
2. Post it with `addCommentToJiraIssue` on the story. If it is longer than
   about 25,000 characters, split it into numbered comments (1 of 3, ...).
3. Transition the story to **In Review**.
4. Assign it to Venkat: `editJiraIssue` with
   `{"assignee": {"accountId": "712020:2ddbb698-d39e-42f0-a308-dda0c1f20b60"}}`.
5. Tell the tester it is done and give them the story link.

## Phase 7 · After Venkat reviews

Venkat's review comment says one of:

- **Approved**: he decides next steps (usually implementation stories for
  Sprint 2 under the same epic). Help the tester draft those stories only if
  asked; Venkat creates or approves them.
- **Changes needed**: move the story back to In Progress, assign it back to
  the tester, work through his comments, and post a revised design as a new
  comment titled "Design v2".

## Report template

```
# <Rule> · gap analysis and design · <DDCPQ-nn>
Researched by <tester> with Claude Cowork · <date> · org cpq-dev

## Summary
3–5 lines: what the rule does today, the biggest gaps, what the design proposes.

## 1. What DD CPQ does today
Evidence-backed. Tool / record / class for each point.
### Tester observed (UI)
### Bugs filed
Case numbers and one line each.

## 2. Vendor research
Vendors covered. One short paragraph per vendor with doc links.
### Other AI tools used
Which tools, where they agreed and disagreed.

## 3. Gap analysis
The table.
### Tester's assessment
In the tester's own words.

## 4. Design
Per gap: user change, behaviour, data model, pipeline step, screens, REST + MCP, acceptance tests.

## 5. Code mapping
The table, plus the hard-rules check (pass / fail per rule).

## 6. Recommended next steps
Ordered list with sizes. Open questions for Venkat.
```

## Never

- Never write to Salesforce. If a tool asks to create or update, stop.
- Never state a vendor capability without a source, or a DD CPQ behaviour without evidence.
- Never paste org data or customer names into prompts for other AI tools.
- Never move a story to In Review before the tester approves the report.
