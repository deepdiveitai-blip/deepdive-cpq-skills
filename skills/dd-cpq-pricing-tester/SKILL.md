---
name: dd-cpq-pricing-tester
description: >-
  Comprehensive DD CPQ pricing + cart + Invoice Timeline + OneTime
  Spread regression testing on Claude Cowork. Covers every real-world
  rate-plan combination (5-axis × trial/prepay/paymentTerm/ramp/
  prorate × multi-currency), verifies Timeline invariants + full
  back-end SOQL fields on QLI + Quote, and checks rate-plan seed
  compliance (PlanGroupKey NOT NULL). Current policy: REPORT-ONLY
  — no Salesforce Cases filed; every finding lands in a self-
  contained HTML report.

  Trigger when the user says: "run the pricing tests", "run
  comprehensive pricing tests", "test everything — waterfall +
  timeline + spread", "run PW scenarios", "verify pricing against
  Notion / Slack / OpenAI / etc.", "check for pricing regressions",
  "generate dynamic pricing tests", "test the Invoice Timeline",
  "test OneTime Spread / Amortization".

  Skip for authoring production rate plans, fixing bugs (that's
  Vijay's job — this skill NEVER modifies code), demoing DD CPQ
  (use dd-cpq-demo).
---

# DD CPQ Pricing Tester · operating guide

This skill turns Claude Cowork into an autonomous **read-and-report**
regression agent for DD CPQ. It runs against a pre-seeded scratch
org (currently `cpq-dev`), exercises every real-world pricing
combination the engine can express, verifies front-end (cart / Invoice
Timeline) against back-end (QuoteLineItem + Quote SOQL), and produces
a single self-contained HTML report per run.

**Current policy: report-only.** The skill does NOT call
`dd_cpq_file_defect` — every finding is captured inline in the HTML
report so the human triaging can decide which findings warrant Cases
and open them by hand. This lowers the false-positive risk while the
new Timeline + Spread coverage is being calibrated.

## Quickstart · execution recipe (read this first)

**If you can only remember one thing:** do these steps in order and
never open a desktop app. Everything you need is in this section — the
rest of the skill is reference. If mid-run you lose access to this
file, keep going from memory.

**Step-by-step for a full comprehensive run:**

1. Enumerate scenarios (Mode A static list + any Mode B / D additions
   the user asked for).
2. For each scenario, run the **3-pass verification protocol**:
   - **Pass 1 · Cart-side** — `dd_cpq_list_quote_lines({quoteId})` +
     `dd_cpq_explain_price({quoteId})`. Compare `ratePlanLabel`,
     `netPrice`, `NMR`, `MRR`, `ARR` against expected.
   - **Pass 2 · Back-end SOQL-equivalent** — `dd_cpq_list_quote_lines`
     (MCP 0.33+) projects the full DDCPQ-2027 QLI field set; walk the
     [SOQL back-end checklist](#soql-back-end-verification). Also
     call `dd_cpq_validate_rate_plan({quoteId})` for seed-rule
     compliance across every RatePlan bound to the quote.
   - **Pass 3 · Timeline + Spread** — call
     `dd_cpq_timeline_preview({quoteId})` (MCP 0.33+) once per
     Timeline-critical scenario. The tool returns pre-scored
     invariants (TL-01…TL-07 + SP-01…SP-05); feed them into the
     report directly. Fall back to manual invariant walking (see
     [Timeline checks](#invoice-timeline-checks) /
     [Spread checks](#onetime-spread-checks)) only if the tool is
     unavailable or a new invariant hasn't been baked in yet.
3. Aggregate findings — **do NOT file Cases**. Every finding goes in
   the HTML report body with numeric contrast + which pass failed +
   root-cause hint.
4. Write the HTML report **directly** via the file-write tool. Never
   open Notepad / VS Code / TextEdit. Target path:
   `~/Downloads/dd-cpq-comprehensive-YYYYMMDD-HHMM.html` (Windows =
   `C:\Users\<you>\Downloads\...`). Never overwrite — timestamp
   suffix makes it unique.
5. Print the absolute path to stdout so the user can open it.

**When things break — degraded-mode fallbacks:**

| If this fails                                                      | Do this instead                                                                                                                                                                                                                  |
| ------------------------------------------------------------------ | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Can't re-read this SKILL.md mid-run (401)                          | Continue from what you already loaded. Note "used cached instructions" in the report footer.                                                                                                                                     |
| HTML file write fails                                              | Retry ONCE. If it still fails, write as `.md` with same base name. If markdown also fails, print the full report body to chat and tell the user: `"File write is broken on your machine — copy this body and save it manually."` |
| Cowork's slash-command autocomplete misses `dd-cpq-pricing-tester` | Read `.claude/skills/dd-cpq-pricing-tester/SKILL.md` directly from the project folder — that's where it lives.                                                                                                                   |
| MCP OAuth expired mid-run                                          | STOP immediately. Ask the user to re-authenticate. Do not silent-retry.                                                                                                                                                          |
| Scenario Quote Name-prefix returns 0 rows                          | Skip the scenario, tally as SKIP (not FAIL). Note in the report footer: "PW-NN skipped — quote not found; may need re-seed."                                                                                                     |
| A back-end SOQL field isn't surfaced via MCP                       | Record as `field not surfaced via MCP` in the scenario section, don't invent a value.                                                                                                                                            |
| The whole run stalls for >2 min at one step                        | You've probably hit a Cowork planning loop. Emit a status update + try to continue. If literally stuck, print a partial report + stop.                                                                                           |

## Hard rules (read these first — non-negotiable)

1. **NEVER modify source code.** No file writes in `force-app/`,
   `mcp-server/`, `.claude/`, or the memory workspace. No git commits,
   no `sf project deploy`, no `npm run build`. If the user asks you
   to "fix" a bug you found, refuse politely — code fixes are Vijay's
   job.

2. **NEVER call config-mutation MCP tools in Mode A / C / D.** The
   forbidden list:
   - `dd_cpq_upsert_rate_plan` (except in Mode B — see below)
   - `dd_cpq_archive_rate_plan`
   - `dd_cpq_save_variable`
   - `dd_cpq_delete_variable`
   - `dd_cpq_apply_rate_plan_template` (except Mode B)
   - `dd_cpq_add_line_item` / `dd_cpq_create_quote` (except Mode B)

   Gone entirely, not merely forbidden: `dd_cpq_create_draft_law`,
   `dd_cpq_activate_constitution`, `dd_cpq_rollback_snapshot`,
   `dd_cpq_check_laws`, `dd_cpq_run_deal_score`, `dd_cpq_file_defect`.
   Constitutional CPQ, Deal Scoring and the defect tracker were cut
   from v1 on 2026-09-13 and their endpoints went with them. A call to
   any of these now fails.

   Rationale: an automated tester that mutates the surface it's
   testing produces false-positive regressions. Read + compare only.

3. **Mode B (dynamic) has one exception:** to seed a web-researched
   pricing pattern, you MAY call the mutation tools above. Every
   record you create MUST follow the [Rate plan seed rules](#rate-plan-seed-rules)
   below — the most important of which is **`PlanGroupKey__c` MUST
   NOT be null** on every seeded RatePlan__c.

4. **REPORT ONLY — do NOT call `dd_cpq_file_defect`.** This is the
   current-policy override. Every finding — whether a Cart-pass
   mismatch, a SOQL-pass field discrepancy, a Timeline-pass invariant
   violation, or a Spread residual bug — gets logged in the HTML
   report body with severity, numeric contrast, and root-cause hint.
   The human triaging the report decides which findings warrant a
   Case and opens them by hand. This intentionally lowers the tool
   surface exposed to Cowork while the new coverage is calibrated.

5. **Every seeded `RatePlan__c` MUST have `PlanGroupKey__c` set to a
   non-null value.** This is the discovery key the Plan Picker and
   the alternatives-list use — a null key hides the plan from every
   UI surface downstream. See [Rate plan seed rules](#rate-plan-seed-rules)
   for the naming convention. Applies to Mode B (dynamic) seeds and
   any composition-mode seeds you author. If you find a pre-seeded
   plan with null `PlanGroupKey__c`, that's a finding — log it and
   move on.

6. **Never launch desktop apps.** No Notepad, no VS Code, no TextEdit,
   no browser tabs (WebSearch is a tool, not a launched app), no
   `powershell.exe -c notepad …`. Everything happens via MCP tools +
   direct file writes. If the runtime prompts "Claude wants to use
   Notepad" during report generation, that's a wrong turn — DENY and
   use the direct file-write path instead.

## The 4 testing modes

### Mode A · Static regression (baseline scenarios)

Fastest mode, and the default when the user says "run the pricing
tests" without specifying.

**Read this before you enumerate anything.** There are two static
baselines, and they live in different orgs:

| Baseline                      | Org              | Status                 |
| ----------------------------- | ---------------- | ---------------------- |
| Rate Plan Coverage (16 plans) | `cpq-dev`        | **current — use this** |
| The 21 PW scenarios           | `cpq-scratch-v3` | retired, not re-seeded |

The [PW scenario table](#pw-scenario-table) further down is kept for
history and for anyone still on the old org. On `cpq-dev` those
quotes do not exist, so enumerating them yields 21 SKIPs and a report
that says nothing. **Do not run the PW list against `cpq-dev`** — use
the coverage matrix below.

#### The `cpq-dev` coverage baseline

One quote, **`Rate Plan Coverage`** on the `Rate Plan Testbed`
Account, backed by 16 Active plans chosen to cover every axis at
least once. All 16 satisfy `PlanGroupKey__c = RevenueNature__c`, so a
seed-compliance pass should come back clean — if it doesn't, that's
the finding.

| Plan     | Product             | Nature    | Model          | Schedule       | Unit    |
| -------- | ------------------- | --------- | -------------- | -------------- | ------- |
| RP-00000 | Collab Seats        | Recurring | PerUnit        | Monthly        | 12      |
| RP-00001 | Collab Seats        | Recurring | PerUnit        | Annual         | 120     |
| RP-00002 | Workspace Pro       | Recurring | PerUnit        | Monthly        | 15      |
| RP-00003 | Support Retainer    | Recurring | FlatFee        | Quarterly      | 2000    |
| RP-00004 | Enterprise Bundle   | Recurring | Package        | SemiAnnual     | 50000 † |
| RP-00005 | Video Conferencing  | Recurring | PerUnit        | UpfrontForTerm | 180     |
| RP-00006 | LLM Tokens          | Usage     | PerUnit        | Monthly        | 0.002   |
| RP-00007 | Payments Processing | Usage     | PercentOfBasis | Monthly        | 0.029   |
| RP-00008 | Object Storage      | Usage     | Tiered         | Monthly        | 0.023   |
| RP-00009 | Log Ingestion       | Usage     | Volume         | Monthly        | 0.1     |
| RP-00010 | API Overage         | Overage   | PerUnit        | Monthly        | 0.05    |
| RP-00011 | Onboarding Package  | OneTime   | FlatFee        | OneTime        | 7500    |
| RP-00012 | Implementation Fee  | OneTime   | FlatFee        | OnEvent        | 2500    |
| RP-00013 | Compute Credits     | Recurring | FlatFee        | UpfrontForTerm | 100000  |
| RP-00014 | Compute Usage       | Usage     | PerUnit        | Monthly        | 0.0025  |
| RP-00015 | Security Add-on     | Recurring | FlatFee        | Annual         | 900     |

RP-00000 and RP-00001 share `PlanGroupKey = Recurring` on the same
product, so they are **alternatives**: exactly one may be on the
quote at a time, and picking the other must swap rather than stack.
That pair is the cheapest regression test in the whole matrix — run
it first.

**† RP-00004 is bound to the `Enterprise Package Brackets` matrix**
(2026-09-14) — 1-50 $50k, 51-200 $180k, 201+ $400k — so the Package
axis is genuinely exercised for the first time. Before this it had no
tier rows at all and every price took the engine's
`'Package fallback (no tier rows): treated as PerUnit'` branch, which
is correct at qty 1 and therefore read green while testing nothing.

**The quote now carries qty 100** (changed 2026-09-14 for N-10). That
matters: qty 1 sits in the first bracket, where the bracket price and
the per-unit price are the same number, so the line reads identically
whether Package is running or has silently fallen back to PerUnit. At
qty 100 the two cannot produce the same figure. Verified values, with
the plan's 2% Net-30 applied:

| qty | bracket  | netPrice | line total |
| --- | -------- | -------- | ---------- |
| 10  | $50,000  | $4,900   | $49,000    |
| 100 | $180,000 | $1,764   | $176,400   |
| 500 | $400,000 | $784     | $392,000   |

The total steps at each boundary and the per-unit price falls — that
stairstep IS the Package model. A line total that rises smoothly with
quantity means Package is not running.

Seeding this immediately surfaced a real defect: `UsageTierService`
had no Package mode and fell back to Volume, handing the flat bracket
price to `PricingModelService` labelled as a per-unit rate, so qty 100
billed $17.64M. Fixed 2026-09-14 (DDCPQ-QA-PKG). If those totals come
back ~100× high, that fix has regressed.

Package lines deliberately carry no ramp modulation, no first-period
proration, no cliff warning and no per-bracket SubRows. Do not report
their absence as a finding.

#### Tier bounds are INCLUSIVE and must not share a boundary

`UsageTier__c.QtyMin__c` / `QtyMax__c` are both inclusive. The engine
sizes a bracket as `QtyMax - QtyMin + 1`, and the admin UI defaults
the first tier's minimum to **1**, not 0. Adjacent tiers therefore
touch without overlapping:

```
correct    1-50000,  50001-500000,  500001-null
wrong      0-50000,  50000-500000,  500000-null
```

The wrong form puts 50,000 in both tiers, so one unit is double-counted
at every boundary — 100,000 units split 50,001 / 49,999. Every seeded
matrix was authored that way until 2026-09-14 (`normalize-usage-tier-
bounds.apex` fixed them). The Object Storage rates differ by only
$0.001, so the blend still read $0.0225 and it went unnoticed through
two runs; a wider rate spread makes it real money.

If a graduated line's SubRows show an odd split at a boundary, check
the bounds before reporting an engine defect.

**Verified on this seed (2026-09-14), so these are real expected
values, not estimates:**

- Collab Seats × 50 on RP-00000 → **$12.00/seat, $7,200 ARR, 14
  waterfall stages**
- Object Storage at 100,000 units on RP-00008 → blended
  **$0.0225**, proving tier blending rather than top-tier-only
- Prepaid credit pools are discoverable via
  `dd_cpq_list_prepaid_credit_pools`

Beyond the matrix, `cpq-dev` also carries two non-rate-plan
baselines worth a pass:

- **`Q-0042 Acme (golden $101.83)`** on `Acme Corp` — the CLAUDE.md
  golden canary. Seeded but **empty**: the four pricing pieces
  (Healthcare −5%, Renewal −3%, $130 absolute PricingRule, 100+
  −15% VolumeDiscountTier) are all in place, and the quote has no
  lines yet. Add 100 of `CRM Suite Pro` and the waterfall must land
  on $101.83 / $122,196 ARR.
- **`Acme Cloud Platform Quote`** on `Acme Corporation` — four
  features, nine options, four OptionConstraints (3 Block + 1
  Warning). Constraint enforcement, not pricing.

### Mode B · Dynamic (web-researched patterns)

When user says "test against real-world pricing" or names a specific
vendor. Cowork uses `WebSearch` to research current pricing, maps to
DD CPQ's 5-axis model, seeds a `DYN-` prefixed plan (with mandatory
non-null `PlanGroupKey__c`), tests it, records the finding in the
report.

**Loop, for each pricing target:**

1. `WebSearch` for the vendor's current pricing (e.g. "Notion Business
   plan pricing 2026").
2. Parse the pattern into DD CPQ's 5-axis + modifier terms — see
   [Rate plan combination matrix](#rate-plan-combination-matrix) for
   every axis and every modifier.
3. Seed per [Rate plan seed rules](#rate-plan-seed-rules).
4. Create test quote and add the line item.
5. Run the 3-pass verification protocol against the new quote.

**Starter WebSearch URL list** (extend as needed):

- https://slack.com/pricing (Recurring PerUnit Monthly/Annual)
- https://zoom.us/pricing (Recurring PerUnit + UpfrontForTerm prepay)
- https://notion.so/pricing (Recurring PerUnit Monthly/Annual)
- https://openai.com/api/pricing (Usage PerUnit per-token)
- https://anthropic.com/pricing (Usage PerUnit per-token + prompt caching)
- https://aws.amazon.com/s3/pricing (Usage Tiered/Graduated per GB)
- https://www.twilio.com/en-us/sms/pricing (Usage Volume per message)
- https://stripe.com/pricing (Usage PercentOfBasis + FlatFee)
- https://www.datadoghq.com/pricing (Recurring + Overage hybrid)
- https://www.snowflake.com/pricing (Prepaid credits + Usage drawdown)
- https://hubspot.com/pricing (Recurring tiered + OneTime onboarding)
- https://zendesk.com/pricing (Package brackets)
- https://linear.app/pricing (Recurring PerUnit + Trial)
- https://vercel.com/pricing (Usage + Recurring hybrid)
- https://cloudflare.com/plans (Freemium + Recurring)

### Mode C · Composition (multi-line stress test)

Optional if Mode A + B leave time. Cowork picks 2-3 PW scenarios,
adds them to ONE Quote via `dd_cpq_add_line_item` (Mode B rules
apply — the composition quote itself must be `DYN-` prefixed), runs
the 3-pass verification protocol on the composed quote, verifies:

- Line totals sum correctly
- Quote-level MRR = Σ (per-line MRR where chargeType='Recurring' and
  isPercentOfBasis=false)
- Quote-level ARR = MRR × 12
- OneTime lines do NOT contribute to MRR (Bessemer clean)
- Amortized OneTime lines contribute to a SEPARATE "Amortized MRR"
  metric (never fold into MRR__c)

**Suggested compositions:**

- PW-01 + PW-03 → Slack monthly + Zoom prepay (2 Recurring naturers)
- PW-05 + PW-15 → SaaS+onboarding + Snowflake pool (Recurring +
  OneTime + Usage mix)
- PW-06 + PW-18 → S3 tiered + Datadog hybrid (Usage + Recurring +
  Overage mix)
- PW-11 + PW-13 + PW-05 → trial + payment terms + amortized onboarding
  (verifies trial savings + payment discount + spread all compose
  cleanly on the Invoice Timeline)

### Mode D · OneTime Spread + Invoice Timeline focused

Dedicated mode when the user says "test OneTime Spread", "test
Invoice Timeline", or "test amortization". Also runs automatically at
the end of Mode A when trial/prepay/ramp/multi-currency scenarios
fire — those are the four Timeline-critical modifiers.

**Loop, for each scenario:**

1. Same 3-pass protocol, but Pass 3 (Timeline + Spread) is deeper:
   - Verify **every Timeline invariant** in [Invoice Timeline checks](#invoice-timeline-checks)
     — sum of Recurring cells, ramp modulator mean-1 property, trial
     coverage math, PercentOfBasis skip.
   - For OneTime lines with `AmortizationScheduleJson__c` set, verify
     **every Spread invariant** in [OneTime Spread checks](#onetime-spread-checks) —
     residual absorbed in last month, `AmortizedMRRContribution__c`
     matches per-month slice, `MRR__c` untouched.
2. Any invariant violation is a finding — log to the report.

## Rate plan combination matrix

Every real-world CPQ pattern the engine should express, with the
5-axis mapping. Mode B seeds should draw from this list; Mode A
scenarios cover these combinations via the PW- table. If a real-world
vendor's pricing doesn't fit any row here, that's a schema gap —
record it in the report footer.

**5 orthogonal axes on `RatePlan__c`**:

| Axis                        | Values                                                                                       | Field                                 |
| --------------------------- | -------------------------------------------------------------------------------------------- | ------------------------------------- |
| RevenueNature               | `Recurring` / `OneTime` / `Usage` / `Overage`                                                | `RevenueNature__c`                    |
| BillingSchedule             | `Monthly` / `Quarterly` / `SemiAnnual` / `Annual` / `UpfrontForTerm` / `OnEvent` / `OneTime` | `BillingSchedule__c`                  |
| PricingModel                | `FlatFee` / `PerUnit` / `Tiered` / `Volume` / `Package` / `Graduated` / `PercentOfBasis`     | `PricingModel__c`                     |
| UnitPrice + MinCommitMonths | positive decimal + integer months                                                            | `UnitPrice__c` + `MinCommitMonths__c` |
| RoundingMode                | `HalfUp` / `HalfEven` / `Ceiling` / `Floor`                                                  | `RoundingMode__c`                     |

**Modifiers on `RatePlan__c`** (compose freely):

| Modifier             | Fields                                                                                             | Test scenario keyword      |
| -------------------- | -------------------------------------------------------------------------------------------------- | -------------------------- |
| Trial                | `TrialDays__c` (int) + `TrialUnitPrice__c` (decimal)                                               | Trial-Free, Trial-Reduced  |
| Payment Terms        | `PaymentTermDiscountPct__c` + `PaymentTerms__c`                                                    | Net30-Terms, Prepay-Terms  |
| Prepaid Credit       | `IsPrepaidCredit__c` (bool)                                                                        | Snowflake-Pool credit line |
| Drawdown             | `DrawdownFromPlanId__c` (lookup → RatePlan)                                                        | Snowflake-Pool draw line   |
| Amortization default | `DefaultAmortizationMonths__c` (int)                                                               | Spread-Onboarding          |
| Segments / Ramp      | `RampSchedule__c` rows w/ `PeriodIndex__c` + `AdjustmentType__c` + `AdjustmentValue__c`            | Ramp-Y1Y2Y3                |
| Prorate              | `EffectiveDate__c` mid-period on Selection                                                         | Proration-Bounds           |
| Multi-currency       | `CurrencyIsoCode` on RatePlan + Quote                                                              | MC-EUR, MC-GBP             |
| Trial + Prepay stack | Trial fields + `BillingSchedule=UpfrontForTerm`                                                    | Trial-then-Prepay          |
| Fan-out (hybrid)     | Multiple RatePlan__c rows sharing Product2 with distinct `PlanGroupKey__c` prefix but same product | Datadog-Hybrid             |

**Real-world combinations Cowork should be able to seed and test**
(one row per DD CPQ-shape that should exist in a comprehensive run —
these are the Mode-B targets):

| #    | Pattern                               | RevenueNature                                         | BillingSchedule | PricingModel   | Modifier                            | Vendor exemplar               |
| ---- | ------------------------------------- | ----------------------------------------------------- | --------------- | -------------- | ----------------------------------- | ----------------------------- |
| C-01 | Per-seat SaaS monthly                 | Recurring                                             | Monthly         | PerUnit        | —                                   | Slack Business+               |
| C-02 | Per-seat SaaS annual (billed once/yr) | Recurring                                             | Annual          | PerUnit        | —                                   | Slack Business+ Annual        |
| C-03 | Prepay 12-mo                          | Recurring                                             | UpfrontForTerm  | PerUnit        | MinCommit=12                        | Zoom Business Prepay          |
| C-04 | Prepay 24-mo w/ discount              | Recurring                                             | UpfrontForTerm  | PerUnit        | MinCommit=24 + payment-term prepay% | Zoom Enterprise 2yr           |
| C-05 | Setup fee                             | OneTime                                               | OneTime         | FlatFee        | —                                   | Onboarding fee                |
| C-06 | Amortized setup fee                   | OneTime                                               | OneTime         | FlatFee        | DefaultAmortizationMonths=12        | Spread onboarding             |
| C-07 | Graduated tier storage                | Usage                                                 | OnEvent         | Graduated      | UsageTierMatrix 3-bracket           | AWS S3 Standard               |
| C-08 | Volume tier msgs                      | Usage                                                 | OnEvent         | Volume         | UsageTierMatrix 3-bracket           | Twilio SMS                    |
| C-09 | Package brackets                      | Recurring                                             | Monthly         | Package        | UsageTierMatrix package             | Zendesk Support               |
| C-10 | Percent-of-basis usage                | Usage                                                 | OnEvent         | PercentOfBasis | UnitPrice=0.029                     | Stripe                        |
| C-11 | Percent-of-basis recurring (AUM)      | Recurring                                             | Monthly         | PercentOfBasis | UnitPrice=0.01 basis-monthly        | Wealth advisor 1% AUM         |
| C-12 | Per-token usage                       | Usage                                                 | OnEvent         | PerUnit        | UnitPrice=0.000015                  | Anthropic Opus input          |
| C-13 | Free trial + Monthly                  | Recurring                                             | Monthly         | PerUnit        | TrialDays=14, TrialUnitPrice=0      | Linear                        |
| C-14 | Reduced trial + Monthly               | Recurring                                             | Monthly         | PerUnit        | TrialDays=30, TrialUnitPrice=1      | Reduced first-month           |
| C-15 | Net-30 early-pay                      | Recurring                                             | Monthly         | PerUnit        | PaymentTermDiscountPct=2            | Net-30 2% off                 |
| C-16 | Prepaid credit pool                   | Recurring                                             | UpfrontForTerm  | FlatFee        | IsPrepaidCredit=true                | Snowflake $50K pool           |
| C-17 | Drawdown-from-credit                  | Usage                                                 | OnEvent         | PerUnit        | DrawdownFromPlanId → C-16           | Snowflake Compute             |
| C-18 | Ramp Y1/Y2/Y3                         | Recurring                                             | Monthly         | PerUnit        | RampSchedule 3 periods +5%/+5%      | Enterprise ramp               |
| C-19 | Prorate mid-period start              | Recurring                                             | Monthly         | PerUnit        | EffectiveDate mid-month             | Mid-month add                 |
| C-20 | Multi-currency EUR                    | Recurring                                             | Monthly         | PerUnit        | CurrencyIsoCode='EUR'               | EU seat                       |
| C-21 | Multi-currency GBP                    | Recurring                                             | Monthly         | PerUnit        | CurrencyIsoCode='GBP'               | UK seat                       |
| C-22 | Hybrid: Recurring + Usage + Overage   | fan-out (3 plans same product, distinct PlanGroupKey) | mix             | mix            | siblings                            | AWS S3 / Datadog Pro          |
| C-23 | Bundle parent + children              | Recurring                                             | Monthly         | PerUnit        | ParentLine linkage via bundle       | Suite + 3 features            |
| C-24 | System + Volume discount stack        | Recurring                                             | Monthly         | PerUnit        | System 5%+3% × Volume 15%           | Healthcare/Renewal enterprise |
| C-25 | Golden $101.83 canary                 | Recurring                                             | Monthly         | PerUnit        | full waterfall                      | CRM Suite Pro Premium         |

## Per-scenario verification protocol

Every scenario MUST run these three passes in order. A "PASS" verdict
requires **all three passes green**. Any failed pass → the scenario
is red; log each pass's finding independently in the report.

### Pass 1 · Cart-side (MCP + engine)

**Tools**: `dd_cpq_list_quote_lines({quoteId})`, `dd_cpq_explain_price({quoteId})`,
`dd_cpq_find_quote({name: 'PW-01...'})` to resolve Ids from Name.

**Checks** (record every field, expected + actual):

- `ratePlanId` populated (non-null)
- `ratePlanLabel` matches scenario's Expected Plan (skip check if
  Expected Plan is `(from <seed>)`)
- `netPrice` per QLI matches expected within ±$0.01
- `NMR` per QLI matches expected within ±$0.01
- `MRR` per QLI = NMR × qty (Bessemer / DDCPQ-2027-M-7 rule)
- `ARR` per QLI = MRR × 12
- `chargeType` and `pricingModel` match the expected axis
- Waterfall stages returned by `dd_cpq_explain_price` — verify the
  EXPECTED stages fired (per scenario) and no unexpected stage fired

#### `applicability` labels are by design — do NOT report them

A stage carries one of three labels, and two of them mean different
things that look alike:

| label            | meaning                                        |
| ---------------- | ---------------------------------------------- |
| `applied`        | the stage changed the price, or cited a rule   |
| `passthrough`    | the stage COULD fire for this line, and didn't |
| `not-applicable` | the stage can NEVER fire for this line         |

Which one a stage gets is decided by the (revenue nature, pricing
model) cell it sits in — `WaterfallApplicability.APPLICABLE_BY_CELL`
lists the applicable stage set per cell. So the SAME stage carrying
the SAME note legitimately gets different labels on different lines:

| line              | cell                | ContractPrice                     |
| ----------------- | ------------------- | --------------------------------- |
| Collab Seats      | Recurring · PerUnit | in the set → `passthrough`        |
| Support Retainer  | Recurring · FlatFee | in the set → `passthrough`        |
| Enterprise Bundle | Recurring · Package | not in the set → `not-applicable` |
| Object Storage    | Usage · Tiered      | not in the set → `not-applicable` |

This was verified line-by-line against the table on 2026-09-14 and
reported twice as an inconsistency before it was. **It is not a
finding.** A not-applicable stage now appends its reason to the note
("… — never fires for Recurring lines priced as Package"), so the
distinction is visible in the payload.

Report a real problem here only if a label contradicts the table —
for example a stage that demonstrably changed the price but is
labelled `passthrough`.

### Pass 2 · Back-end SOQL-equivalent (via MCP field projection)

Use `dd_cpq_list_quote_lines` (**MCP 0.33+** projects the full
DDCPQ-2027 QLI field set: ramp, amortization, trial, prepay, prorate,
sibling, waterfall JSON, all RatePlan modifier metadata) and
`dd_cpq_find_quote` for Quote-level totals. See [SOQL back-end
verification](#soql-back-end-verification) for the full field
checklist. Any field surfaced by MCP that is null-but-should-be-
populated OR populated-but-wrong is a finding.

Also use **`dd_cpq_validate_rate_plan`** (new in MCP 0.33) to walk
seed-rule compliance against every RatePlan bound to a QLI on the
quote:

```
dd_cpq_validate_rate_plan({ quoteId })
```

Returns per-plan `isValid` + severity-tagged findings for SEED-1
(PlanGroupKey non-null), SEED-3 (PricingEngineVersion=v2), SEED-4
(Status=Active), AXIS-1..5 (5-axis completeness), SCOPE-1
(TargetProduct non-null), NAMING-1 (DYN- prefix convention). Feed
every finding into the report's [Rate-plan seed compliance section].

### Pass 3 · Timeline + Spread

**Only runs on scenarios that touch Timeline-critical modifiers**
(trial, ramp, prepay, multi-currency) OR OneTime lines with an
active amortization schedule.

Use **`dd_cpq_timeline_preview`** (new in MCP 0.33) to compute the
Timeline aggregation server-side + get a pre-scored invariant list:

```
dd_cpq_timeline_preview({ quoteId })
```

Returns:

- `columns[]` — the adaptive month/quarter/year axis (matches LWC).
- `rows[].cells[]` — per-column cell values, computed with NMR-based
  math (matches DDCPQ-2027-M-timeline commit `2155246`).
- `sidebar` — TCV, ARR, First-month, Steady-state, Trial savings,
  Amortized MRR.
- `invariants[]` — pre-scored TL-01…TL-07 (Timeline) and
  SP-01…SP-05 (Spread) checks with `ok`/`expected`/`actual`/`delta`.

Feed each invariant into Pass 3's row of the report. Since DDCPQ-59
a ramped new-sale line is one QLI per contract year (see TL-02), so
each year's cells are checked against that year's own price. All
other invariants (Trial, UpfrontForTerm, Amortization,
PercentOfBasis skip) are checked to full precision.

If a scenario needs manual invariant walking (older MCP, or a
new invariant not yet baked into the tool), consult [Invoice
Timeline checks](#invoice-timeline-checks) and [OneTime Spread
checks](#onetime-spread-checks) for the math.

## Invoice Timeline checks

Timeline (`cpqInvoiceTimeline` LWC — see the 2026-09-09 refresh in
MILESTONES.md) is a client-side derivation from `CartLine[]`. To
verify without JS execution, walk these invariants using the raw QLI

- RatePlan + RampSchedule data:

### Invariant TL-01 · Sum of Recurring cells = MRR × termMonths (no ramp)

For a Recurring line without ramp:

- Timeline cell = `NormalizedMonthlyRate__c × Quantity` per month
- Sum across term = `NMR × Qty × TermMonths__c`
- Should equal `MRR__c × TermMonths__c` (since MRR = NMR × Qty)

**Test**: use `MRR__c`, `TermMonths__c` from QLI, compute expected
sum, compare against per-scenario expected TCV. Delta > $0.01 → finding.

**2026-09-10 fix (report finding F-01/F-02)**: sidebar TCV now equals
`Σ row.cells.amount` by construction — the per-branch accumulator
that hit an off-by-term bug on M2M trial plans (`TermMonths__c=1`)
was replaced with a direct cell sum. On those plans the row paints
12 cells at the plan's monthly rate, and TCV now matches. TL-01
should be tautological on any fresh reprice; any drift means a cell
went wrong, not the accumulator.

### Invariant TL-02 · Ramp years (DDCPQ-59)

A recurring, rateable new-sale line over 12 months whose product has
`RampSchedule__c` rows is **one QLI per contract year**, sharing
`SegmentGroupKey__c`; `SegmentIndex__c` 0 is Year 1.

- Each year: 12 months (the last takes the rest), `EffectiveDate__c`
  = the previous year's start + 12 months.
- `SegmentSource__c = Rule`: Year n's `RampAdjustedPrice__c` = Year n-1's
  rate adjusted by the rule for period n-1; Year 1 is list.
  `Manual`: a year with `SegmentBasePrice__c` uses it, any other year keeps
  the rule's price.
- Discounts (term, system, volume, manual) run on each year separately.
- Quote ARR = Year 1 ARR; Exit ARR = the last year's ARR.

**Test**: per year, rule-compounded rate → `RampAdjustedPrice__c`,
then the waterfall to `NetPrice__c`. Any drift > $0.02 → finding.
Three lines where you expected one is correct; one averaged line on a
new sale over 12 months with a ramp rule is a finding.

The term-averaged check below still applies to lines that are NOT split
(usage charges, amendment/renewal lines, hybrid siblings):

- Ramp rates per period: `[r1, r2, ..., rn]`
- Months per period: `[m1, m2, ..., mn]` (12 each; last absorbs
  leftover)
- Weighted average = `Σ(ri × mi) / Σ mi`
- Modulator[p] = `rp / weightedAvg`
- Σ (modulator[p] × months[p]) should equal `Σ months = TermMonths`
  (that's the mean-1 property).

**Test**: read RampSchedule rows for the product, compute the weighted
avg, verify it matches `RampAdjustedPrice__c` on the QLI. Compute
per-period cell modulation, verify sum equals flat MRR × TermMonths.
Any drift > $0.02 → finding.

### Invariant TL-03 · UpfrontForTerm cells use NMR (not netPrice)

For a Recurring line with `BillingSchedule__c='UpfrontForTerm'`:

- `NetPrice__c` = term total (e.g. Zoom $180 for 12mo)
- `NormalizedMonthlyRate__c` = term total / MinCommitMonths ($15/mo)
- Timeline cell should show NMR × qty per month, NOT netPrice × qty
- Row total = NMR × qty × termMonths = netPrice × qty (they match by
  construction for pure UpfrontForTerm)

**Test**: verify `NMR × TermMonths = NetPrice` (per unit) for the
QLI. If they don't match → engine mis-stamped NMR, log finding.

### Invariant TL-04 · Trial reduces TCV but NOT ARR

For a Recurring line with `TrialDays__c > 0`:

- Trial coverage in months = `TrialDays / 30`
- Trial savings = `(NMR × Qty − TrialUnitPrice × Qty) × trialCoverage`
- Expected TCV = `(MRR × TermMonths) − trialSavings`
- Expected ARR = `MRR × 12` (Bessemer — ARR reflects steady state,
  UNCHANGED by trial)

**Test**: verify `TrialEndDate__c` is stamped and equals
`EffectiveDate__c + TrialDays`. Verify the reduced TCV matches sum
of Timeline cells within ±$0.02.

### Invariant TL-05 · PercentOfBasis skipped from cell math

For a Recurring line with `PricingModel__c='PercentOfBasis'` (AUM
case): Timeline cells should be `—` (empty), not `netPrice × qty × months`
which would produce cents-scale garbage.

**Test**: verify the LWC would skip these cells — no direct SOQL
check possible, but confirm `NetPrice__c` is a decimal like 0.029 (not
a dollar value) so any wrong renderer would produce visibly-tiny
numbers.

### Invariant TL-06 · Multi-currency plumbing

For a quote where `Quote.CurrencyIsoCode ≠ 'USD'`:

- Every QLI on the quote inherits the quote's currency
- `RatePlan__c.CurrencyIsoCode` matches (else engine falls back to
  USD and stamps `currencyFallbackApplied` — DDCPQ-2027-C behavior).

**Test**: verify `Quote.CurrencyIsoCode`, verify the picked RatePlan
matches (query RatePlan.CurrencyIsoCode = Quote.CurrencyIsoCode).
Mismatch → USD fallback fired (log as informational, not a bug per
DDCPQ-2027-C's intentional fallback).

**2026-09-10 (F-15 fix)**: `dd_cpq_list_quote_lines` (MCP 0.34+) now
surfaces `quoteCurrencyIsoCode` at the result root and, per line,
`ratePlanCurrencyIsoCode` + `currencyFallbackApplied`. On the cart,
a matching line renders an amber "USD fallback" badge. When you see
`currencyFallbackApplied === true`, log it as **Info** (documented
DDCPQ-2027-C behavior), never Red. Also assert the cart badge is
visible if you verify via LWC.

### Invariant TL-07 · Prepaid + drawdown badges

For a Recurring line with `IsPrepaidCredit__c=true`: expect a
"Prepaid credit $X" chip. For a Usage line with
`DrawdownFromPlan__c` non-null: expect a "Draws from credit" chip.
Cell math unchanged (drawdown is Usage, still Phase 2 — no
projection yet).

**Test**: verify `IsPrepaidCredit__c` on RatePlan and
`DrawdownFromPlanId__c` are stamped as expected.

## OneTime Spread checks

For OneTime lines with `AmortizationScheduleJson__c` populated
(DDCPQ-058 Spread feature):

### Invariant SP-01 · Amortization fields all stamped

- `AmortizationScheduleJson__c` non-null (the client-authored schedule)
- `AmortizedMRRContribution__c` non-null = per-month contribution
- `AmortizationStartMonth__c` = 1-based start month (typically 1)
- `AmortizationEndMonth__c` = 1-based end month (typically N)
- `AmortizationEndMonth - AmortizationStartMonth + 1 = duration in
months`

**Test**: read all 4 fields, verify none are null and the arithmetic
holds.

### Invariant SP-02 · Sum of per-month contributions = line total

- `AmortizedMRRContribution__c × durationMonths` should equal
  `NetPrice__c × Quantity` within ±$0.01
- **BUT**: the last active month absorbs the rounding residual (per
  Zuora/SBQQ/NetSuite standard). So if `NetPrice × Qty = 10,000` and
  duration = 12: `AmortizedMRR = 833.33`, but the M12 cell shows
  `833.37` to close the $10,000 exactly.
- Timeline must sum to exactly `NetPrice × Qty` (not `833.33 × 12 =
9,999.96`).

**Test**: compute `AmortizedMRRContribution × (duration - 1) + <last
month absorbing residual>` should equal `NetPrice × Qty`. If the QLI
doesn't have `LastMonthAdjustment__c` (or equivalent), rely on the
Timeline math confirming the sum matches.

### Invariant SP-03 · MRR__c UNTOUCHED by amortization

- Bessemer / Meritech / SaaSMetrics rule: MRR is recurring-only.
- OneTime amortized fees are `AmortizedMRRContribution__c`, kept
  SEPARATE from `MRR__c`.
- **Test**: verify `MRR__c` on the OneTime QLI is either null or 0
  (never non-zero — a non-zero MRR on a OneTime QLI is a finding).

### Invariant SP-04 · Timeline distribution correctness

Amortized OneTime spreads across cells starting at
`AmortizationStartMonth` and ending at `AmortizationEndMonth`, with
the final active cell absorbing residual. Cells before start and
after end contribute nothing from this line. Non-amortized OneTime
folds into M1 (first-period column).

**Test**: given the schedule bounds and per-month contribution,
compute expected Timeline cell values for M1..M<term>. Compare
against expected TCV = sum of cells = `NetPrice × Qty`.

### Invariant SP-05 · Amortized MRR sidebar row

Timeline sidebar shows a distinct "Amortized MRR" row when
`AmortizedMRRContribution > 0.005` (SUM across all amortized
OneTime lines). This is separate from the "Steady-state / month"
row (recurring-only MRR).

**Test**: sum `AmortizedMRRContribution__c` across all OneTime QLIs
on the quote. If sum > 0.005, expect the sidebar to render this
row. Report as informational.

## SOQL back-end verification

Every field the DD CPQ engine writes back to `QuoteLineItem` should
be surfaced through `dd_cpq_list_quote_lines` (MCP 0.29+). The
tester walks this checklist for every scenario. Any field surfaced
but null-when-should-be-populated, or populated-but-wrong, is a
finding. Any field NOT surfaced is a coverage gap (record separately).

### QLI fields to verify per scenario

**Identity + pricing seed:**

- `Product2Id`, `Product2.Name`
- `Quantity`
- `UnitPrice` (should equal engine's NetPrice per DDCPQ-2027-M-7
  post-fix commit `fb94704`)
- `ListPrice`

**Engine waterfall stamps** (v2 pipeline output):

- `DerivedListPrice__c`
- `RampAdjustedPrice__c` (null if no ramp)
- `TermAdjustedPrice__c` (null if no term curve)
- `UsageTierPrice__c` (null if no usage tier)
- `SystemDiscount__c` (percent, cumulative)
- `VolumeDiscount__c` (percent)
- `Discount` (manual override %)
- `NetPrice__c` (final per-unit or term-total for UpfrontForTerm)

**Normalized economics** (Bessemer / DDCPQ-2027-M-7):

- `NormalizedMonthlyRate__c` (per-unit, not line-total!)
- `MRR__c` (= NMR × Qty for Recurring; 0 or null for OneTime/Usage)
- `ARR__c` (= MRR × 12)

**Contract shape:**

- `TermMonths__c`
- `BillingFrequency__c` — **F-14 (2026-09-10) convention**: on
  Recurring lines, mirrors the plan's BillingSchedule when picklist-
  legal (`Monthly` / `Quarterly` / `Annual`), else null. On **Usage /
  Overage** lines whose plan's schedule is `OnEvent` /
  `UpfrontForTerm` / `OneTime`, defaults to `Monthly` (Bessemer/Zuora
  invoicing cadence). Null on OneTime lines (picklist has no value
  for it). If you see a Usage/Overage line with `null`, log finding.
- `EffectiveDate__c` — **F-12 (2026-09-10)**: ProrationService now
  stamps this on every Recurring draft (was only stamping when
  proration math actually fired). On a fresh reprice, every Recurring
  QLI should carry a date. Null → finding (unless line was written
  pre-fix and never reconciled).
- `ChargeType__c` (Recurring / OneTime / Usage / Overage)
- `PricingModel__c`
- `RatePlan__c` (lookup — should be non-null in DDCPQ-2027 era)
- `ChargeProfile__c` (legacy PCP; deprecated post J1)

**Modifier stamps:**

- `TrialDays__c`, `TrialUnitPrice__c`, `TrialEndDate__c` (trial)
- `PaymentTerms__c`, `PaymentTermDiscountPct__c` (payment terms)
- `IsProrated__c`, `ProratedDays__c`, `ProratedFirstPeriodAmount__c`
  (prorate). **F-10 (2026-09-10)**: BoundaryScale (Usage/Overage
  tier proration) now stamps all three fields — previously only
  ProrationService (Recurring path) did. On PW-17 with
  `EffectiveDate__c = 2026-09-16` expect `true / 15 / $480.00`.
  Missing = finding.

**Spread / amortization** (DDCPQ-058):

- `AmortizationScheduleJson__c`
- `AmortizedMRRContribution__c`
- `AmortizationStartMonth__c`, `AmortizationEndMonth__c`

**Sibling linkage** (DDCPQ-059):

- `SiblingGroupKey__c` (UUID External Id)
- `SiblingRoot__c` (self-lookup to primary)

**Bundle + hierarchy:**

- `ParentLine__c` (null for standalone / bundle parent; set on children)
- `IsBundleParent` (server-computed)
- `SourceBundle__c` (Product2 lookup to the bundle SKU)

**Commit + audit:**

- `WaterfallJson__c` (DDCPQ-052 serialized WaterfallStage[]). **F-11
  (2026-09-10)**: reconcile UPDATE now stamps this field — before,
  only the INSERT path did, so QLIs seeded pre-DDCPQ-052 or
  reconciled since had null. On a fresh reprice every line should
  carry non-null waterfall. Legacy nulls heal on next reprice.
- `BlockedByFloor__c` (margin floor)
- `IsFloorAdjustment__c` (DDCPQ-046)

**Rate plan invariants (SEED-\*):**

- **SEED-1 (Rule 1)**: `RatePlan__c.PlanGroupKey__c = RevenueNature__c`
  case-sensitive. `dd_cpq_validate_rate_plan` enforces this
  per-quote. Org-wide sweep still worth a spot check — 6 pre-existing
  lowercase `recurring` values were normalised 2026-09-10 (F-03).
- **SEED-2 (F-06 · 2026-09-10)**: every Active `(TargetProduct,
PlanGroupKey)` group must have **exactly one** plan with
  `IsDefaultInGroup__c = true`. Enforced automatically at write time
  by `CpqRatePlanEditorController.ensureGroupHasDefault` and
  backfilled once org-wide. Zero defaults → Plan Picker has no
  authored initial chip; multiple defaults → ambiguous. Both
  findings.

### Quote-level fields to verify per scenario

- `Quote.TotalPrice`, `Quote.GrandTotal`, `Quote.Subtotal`
- `Quote.CurrencyIsoCode` (surfaces to Timeline via
  `BootstrapResult.quoteCurrencyIsoCode`)
- Any DD CPQ custom Quote fields (Total MRR, Total ARR) — surface
  via `dd_cpq_find_quote` if available

### Rate-plan-level fields (for DYN- seeds Cowork creates)

Verify on every RatePlan__c Cowork creates in Mode B:

- **`PlanGroupKey__c` MUST NOT be null** — see [Rate plan seed rules](#rate-plan-seed-rules)
- `Product__c` non-null (or applicability via criteria)
- `Status__c` = 'Active' (else invisible to picker)
- `PricingEngineVersion__c` = 'v2' (else quotes route through v1
  which ignores RatePlan.UnitPrice — see memory
  `feedback_seed_active_constitution`)
- `RevenueNature__c`, `BillingSchedule__c`, `PricingModel__c` all
  populated per axis rules

## Rate plan seed rules

**Every RatePlan__c Cowork creates in Mode B must obey these rules.**
Break any of them and the plan will silently misbehave downstream —
usually invisibly to the picker.

### Rule 1 · `PlanGroupKey__c` = `RevenueNature__c`

**⚠ Rewritten 2026-09-10 (Venkat design call).** The old rule was
"non-null with your-choice-of-value; distinct = siblings, same =
alternatives." That was confusing and inconsistent. The new rule is
mechanical:

```
PlanGroupKey__c = RevenueNature__c
```

Every seed sets `PlanGroupKey__c` to one of the 4 RevenueNature
values: `Recurring` / `OneTime` / `Usage` / `Overage`. That's it.

Consequences the tester should verify:

- **Alternatives** = multiple plans on the SAME product with the SAME
  RevenueNature. E.g. E2E Product Test - Balaji has 3 Recurring plans
  (M2M / SemiAnnual / Annual) → all share `PlanGroupKey='Recurring'`
  → picker shows 3 chips, rep picks ONE.
- **Siblings that all fire** = plans on DIFFERENT products. Old
  "same product with distinct keys" fan-out is retired. If you need
  Stripe's "2.9% + $0.30 per transaction", model as TWO products
  (Stripe Payment Processing @ 2.9% + Stripe Per-Transaction Fee @
  $0.30). Two products in the cart → two lines.
- **Two RevenueNatures on the same product** (E2E with Recurring +
  OneTime) → two groups in the picker; the cart's handlePlanPick
  correctly adds a second line when the rep clicks from a new group.

**Field is hidden from the Rate Plan Editor**. Admins can no longer
type a key — the editor's save handler stamps `payload.planGroupKey
= m.revenueNature`. Any seed authored via the editor honors the rule
automatically. Mode B seeds should populate it explicitly using the
`= RevenueNature` rule.

**Tester's SEED-1 check** now verifies:

- Not just "non-null" — also `PlanGroupKey__c === RevenueNature__c`.
- Any mismatch is a finding: the plan will still rate but the
  picker's grouping is wrong, and the LWC handlePlanPick's SWAP-
  vs-ADD logic depends on group inference.

**Old naming convention retired.** DYN- prefix still applies to Name
(so admins can clean up), but PlanGroupKey should just be the plain
RevenueNature value.

### Rule 2 · `DYN-` prefix on Name and PlanGroupKey

Every Mode B-created record MUST start with `DYN-` so admins can
clean up dynamic seeds without touching pre-seeded PW- data. Applies
to:

- `RatePlan__c.Name`
- `RatePlan__c.PlanGroupKey__c`
- Any `UsageTierMatrix__c` / `UsageTier__c` rows Cowork creates
- Any `Quote.Name` Cowork creates (`DYN-<Vendor>-Q<n>`)
- Any `Product2.Name` (only if Cowork is creating a new product;
  prefer reusing existing PW products where possible)

### Rule 3 · (removed) Constitution engine version

This rule required every seeded run to stamp
`PricingEngineVersion__c = 'v2'` on an active `PolicyConstitution__c`,
because the legacy v1 pipeline ignored `RatePlan.UnitPrice__c` and
routed through the old ProductChargeProfile__c instead.

Both are gone. The v1 pipeline was removed during the AppExchange
extraction, and Constitutional CPQ was cut from v1 on 2026-09-13, so
`PolicyConstitution__c` no longer ships. There is one pricing engine
now and nothing to select between.

### Rule 4 · Set `Status__c = 'Active'`

For a RatePlan__c to be discoverable, `Status__c = 'Active'`. The
former carve-out for Constitutions no longer applies — see Rule 3.

### Rule 5 · Currency-scope the plan when quote is multi-currency

If Mode B seeds a plan against a quote whose `CurrencyIsoCode ≠ 'USD'`,
the RatePlan must have matching `CurrencyIsoCode` set. Otherwise the
engine falls back to USD and stamps `currencyFallbackApplied` on the
LineDraft (DDCPQ-2027-C intentional behavior). If Cowork wants to
test EUR / GBP scenarios end-to-end, seed the plan in that currency.

### Rule 6 · UsageTier structure needs a matrix header

For any `PricingModel__c ∈ {Tiered, Volume, Graduated, Package}`:

- Create `UsageTierMatrix__c` header (name `DYN-...`)
- Create N `UsageTier__c` bracket rows referencing the matrix
- Reference the matrix from the RatePlan (or ProductChargeProfile if
  legacy path) — check current MCP tool contract for
  `dd_cpq_upsert_rate_plan` matrix-linking support.

### Rule 7 · Ramp needs both header + schedule rows

For per-period modulation (DDCPQ-2027-M-timeline consumption):

- Create `RampMatrix__c` header (Status='Active')
- Create N `RampSchedule__c` rows w/ `PeriodIndex__c` = 0, 1, 2, ...
  and `AdjustmentType__c` + `AdjustmentValue__c`
- Scope via `TargetProduct__c` (one product), `Target__c` (many: a list,
  product-field conditions such as Family, or a product group — preview
  with `dd_cpq_rule_target_preview`), `TargetQuote__c`, or shared 5-field
  criteria.

### Rule 8 · Any rule can cover many products

Every rule type except Commitment Discounts takes `Target__c` JSON:
`{"mode":"All"}`, `{"mode":"Products","products":[ids]}`,
`{"mode":"Criteria","criteria":{"conditions":[{"n":1,"field":"Product.Family","op":"equals","value":"Hardware"}]}}`
or `{"mode":"Group","groupId":id}`, each with optional `"exclude":[ids]`.
When two rules cover a line, the one that names the product wins, then
conditions or a group, then every product. Promotions, channel discounts
and contract prices now take WHEN conditions too. Test both a named-product
rule and a family rule on the same product to check the order.

## PW scenario table

> **Historical — `cpq-scratch-v3` only.** None of these 21 quotes
> exist in `cpq-dev`, the org this skill now targets. Running the
> list there produces 21 SKIPs. The current static baseline is the
> [`cpq-dev` coverage baseline](#the-cpq-dev-coverage-baseline) under
> Mode A. Kept here because the expected values are still correct for
> anyone testing on the old org, and because several document real
> engine behaviour worth preserving.

Query the quote by Name (e.g. `Quote.Name LIKE 'PW-01%'`)
rather than by hardcoded Id — Ids rotate on scratch-org rebuilds.

| #     | Keyword          | Quote name prefix                                 | Product                              | Expected Plan (ratePlanLabel)                | Expected pricing                                                                                                                                                  | Functionality     |
| ----- | ---------------- | ------------------------------------------------- | ------------------------------------ | -------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------- | ----------------- |
| PW-01 | Slack-Monthly    | `PW-01 Slack per-seat monthly`                    | PW Slack Business Plus               | `Monthly · $12/user`                         | NMR=$12.00, MRR=$1,200, ARR=$14,400                                                                                                                               | Rate Plan         |
| PW-02 | Slack-Annual     | `PW-02 Slack per-seat annual`                     | PW Slack Business Plus Annual        | `Annual · $120/user/yr`                      | NMR=$10.00, MRR=$500, ARR=$6,000                                                                                                                                  | Rate Plan         |
| PW-03 | Zoom-Prepay      | `TRAIL2027 Quote A — Zoom Prepay`                 | TRAIL2027 Zoom Business Prepay       | `Prepay 12mo`                                | Line total $180 (12-mo upfront); NMR=$15/user/mo; ARR=$180                                                                                                        | Rate Plan         |
| PW-04 | Zoom-PlanPicker  | `TRAIL2027 Quote B — Zoom Plan Picker`            | TRAIL2027 Zoom Business Multi-Plan   | `Month to Month`                             | Initial default = M2M ($20 NMR). Rep can swap via Plan Picker chips to 12M Commit ($150/yr → NMR $12.50) or 12 Months Min Commit Prepay ($180 upfront → NMR $15)  | Rate Plan         |
| PW-05 | SaaS-OneTime     | `PW-05 SaaS + onboarding`                         | PW SaaS Suite + PW Onboarding        | (2 lines, plan-per-line)                     | MRR=$200 (SaaS only); onboarding is OneTime, MRR=$0                                                                                                               | MRR / ARR         |
| PW-06 | S3-Graduated     | `PW-06 AWS S3 tiered`                             | PW AWS S3 Standard                   | `Tiered (Graduated) · 3-bracket`             | UsageTierPrice=$0.0225 (50×$0.023 + 50×$0.022)                                                                                                                    | Usage Tier        |
| PW-07 | Twilio-Volume    | `PW-07 Twilio SMS volume`                         | PW Twilio SMS                        | `Volume · whole qty at winner`               | UsageTierPrice=$0.005/msg (winning bracket 10001-100K)                                                                                                            | Usage Tier        |
| PW-08 | Stripe-Hybrid    | `TRAIL2027E Quote A`                              | (Stripe hybrid — 2 lines)            | (Percent + FlatFee, per-line)                | 2 sibling lines: PercentOfBasis + FlatFee                                                                                                                         | Pricing Model     |
| PW-09 | Anthropic-Tokens | `PW-09 Anthropic Opus`                            | PW Anthropic Opus Input Tokens       | `Per-token · $0.000015/tok`                  | 1M tokens × $0.000015 = $15 line total                                                                                                                            | Pricing Model     |
| PW-10 | Category-Rating  | `TRAIL2027M M-3`                                  | (M-3 seed)                           | (from M-3 seed)                              | UsageTierPrice=$12 (EU wins under Highest)                                                                                                                        | Usage Tier        |
| PW-11 | Trial-Free       | `TRAIL2027D Quote A — Free Trial Active`          | (TRAIL2027D seed)                    | `Monthly · 14-day Free Trial`                | TrialEndDate stamped, MRR at FULL post-trial rate (Bessemer)                                                                                                      | Rate Plan         |
| PW-12 | Trial-Reduced    | `TRAIL2027D Quote B — Reduced Trial Active`       | (TRAIL2027D seed)                    | `Monthly · $1 first month`                   | TrialEndDate + $1 trial rate; MRR at post-trial rate                                                                                                              | Rate Plan         |
| PW-13 | Net30-Terms      | `TRAIL2027F Quote A — Net-30 · 2% off`            | (TRAIL2027F seed)                    | `Monthly · Net-30 · 2% early-pay`            | PaymentTermDiscount stage fires -2%                                                                                                                               | Discounting       |
| PW-14 | Prepay-Terms     | `TRAIL2027F Quote B — Prepaid · 5% off`           | (TRAIL2027F seed)                    | `Annual · Prepaid · 5% off`                  | Prepay + PaymentTermDiscount stack                                                                                                                                | Discounting       |
| PW-15 | Snowflake-Pool   | `TRAIL2027G Quote — $50K credits + $2/hr compute` | (TRAIL2027G seed)                    | `Prepaid · $50K credit pool`                 | 2 lines with Prepaid Credit + Draws-from-credit badges                                                                                                            | Pricing Model     |
| PW-16 | Segments-Ramp    | `TRAIL2027M M-2`                                  | (M-2 seed)                           | (from M-2 seed)                              | UsageTierPrice=$8.00 (weighted-avg Y1/Y2/Y3)                                                                                                                      | Usage Tier        |
| PW-17 | Proration-Bounds | `TRAIL2027M M-4`                                  | (M-4 seed)                           | (from M-4 seed)                              | UsageTierPrice=$8 (tier bounds ×0.5 mid-month)                                                                                                                    | Usage Tier        |
| PW-18 | Datadog-Hybrid   | `PW-18 Datadog hybrid`                            | PW Datadog Pro Host + Overage        | (2 lines, plan-per-line)                     | Recurring $30/host + Overage $0.05 (Bessemer split)                                                                                                               | MRR / ARR         |
| PW-19 | SysDisc-Stack    | `PW-19 SystemDiscount`                            | PW Discount Test Product             | `Monthly · $100/user (watch discounts fire)` | UnitPrice=$92.15 (Healthcare -5% × Renewal -3% multiplicative)                                                                                                    | Discounting       |
| PW-20 | VolDisc-Stack    | `PW-20 VolumeDiscount`                            | PW Discount Test Product             | `Monthly · $100/user (watch discounts fire)` | UnitPrice=$78.33 (Sys stack + Volume -15% at qty=150)                                                                                                             | Discounting       |
| PW-21 | Golden-101.83    | `PW-21 Golden $101.83`                            | PW Golden CRM Suite Pro              | `Monthly · $150 list · Premium tier`         | **Golden canary**: UnitPrice=$101.83, MRR=$10,183, ARR=$122,196                                                                                                   | Pricing Waterfall |
| PW-22 | Spread-Onboard   | `PW-22 Onboarding Amortized`                      | PW Enterprise Onboarding (Amortized) | `One-time · $12K (spread 12 months)`         | OneTime NetPrice=$12,000 · **MRR/ARR = $0** (Bessemer clean) · AmortizedMRR=$1,000/mo · Sched=`{"months":12,"startPeriod":1,"total":12000}` · lights SP-01..SP-05 | Spread (Amort.)   |

**Rate-plan verification note**: `Expected Plan` values come from the
seed scripts + trail doc. When `Expected Plan` says `(from <seed>)`,
the plan-binding pre-check is skipped — the seed is the source of
truth and any actual bound plan is treated as intentional. Only log
"wrong plan" findings when the expected label is an exact string.

## Report format — HTML, self-contained, written every run

Write to `~/Downloads/dd-cpq-comprehensive-YYYYMMDD-HHMM.html` on
Windows (`C:\Users\<you>\Downloads\...`) or `~/Downloads/...` on
macOS/Linux. Inline CSS, no external dependencies, no images — must
open standalone in any browser and be email-shareable.

**File-name convention** (do not deviate):
`dd-cpq-<modeLabel>-<yyyymmdd>-<hhmm>.html` where `<modeLabel>` is
`mode-a` / `mode-b` / `mode-c` / `mode-d` / `comprehensive` (when 2+
modes ran together). Never overwrite — timestamp suffix guarantees
uniqueness so multiple runs on the same day stack up in Downloads.

**Required structure** (14 sections, in order):

1. `<title>` — `DD CPQ <modeLabel> · <timestamp>`.

2. **Sticky header bar**: run timestamp (ISO), org alias, org
   instance URL, MCP server version, active Constitution name (if
   known), mode(s) run, total wall-clock duration. Include a bold
   "REPORT-ONLY MODE — no Cases filed" note in the header so the
   reader knows to triage findings by hand.

3. **Summary tiles**:
   - Total scenarios · Green · Red · Skipped · Findings-count (across
     all 3 passes)
   - Style green count bold-green (`#16a34a`) and red bold-red
     (`#dc2626`).

4. **Findings triage table** (replaces the old "Cases filed" section):
   - Columns: `Scenario · Pass (Cart/SOQL/Timeline/Spread) · Severity
· Field / Metric · Expected · Actual · Root-cause hint`
   - Sort by Severity DESC (High first)
   - This is the "handoff to the human" — the reader uses this to
     decide which findings deserve Cases.

5. **Scenario index** — jump table with one row per scenario:
   `Keyword · Quote name · Cart pass · SOQL pass · Timeline pass ·
Spread pass · Overall`. Each row anchor-links to the scenario
   section below.

6. **Per-scenario section** — one `<section>` per scenario with an
   anchor `id="pw-01"`, containing:
   - **Use case** — one-sentence plain-English of the real-world
     pricing pattern.
   - **Tested record** — Quote Name + Number, Id as clickable link
     (target=_blank). URL: `<orgInstanceUrl>/lightning/r/Quote/<quoteId>/view`.
   - **QuoteLineItems** — one bullet per QLI with Product, Qty,
     NetPrice, MRR, ARR, ratePlanLabel, trialEndDate (if set),
     billingFrequency, termMonths, QLI Id (monospace).
   - **Waterfall** — compact table (Stage / Value / Rule cited / Note).
   - **Pass 1 · Cart-side** table (Metric / Expected / Actual / Delta /
     PASS-FAIL badge)
   - **Pass 2 · Back-end SOQL** table — every field in the [SOQL back-end
     checklist](#soql-back-end-verification) surfaced by MCP, in a
     three-column layout: `Field · Value · Check`. Green `✓` for
     "matches expected", red `✗` for "mismatch", grey `—` for "field
     not surfaced by MCP" (coverage gap).
   - **Pass 3 · Timeline + Spread** table — one row per invariant
     (TL-01 … TL-07, SP-01 … SP-05, only those applicable to the
     scenario), with PASS/FAIL badge and numeric contrast.
   - **Overall verdict** badge (PASS / FAIL / SKIP) — PASS requires
     all applicable passes green.
   - **Root-cause hint** — one sentence if inferrable
     ("seed missed PlanGroupKey", "engine stamps line-total NMR
     — DDCPQ-2027-M-7 regression?", etc).

7. **Rate-plan seed compliance section** — dedicated section listing
   every RatePlan__c touched during the run with its `PlanGroupKey__c`
   value. Flag any null values in red. Motivation: the #1 seed bug
   this skill catches.

8. **Rate-plan verification section** — table: `Scenario · Expected
Plan · Actual Plan · Match?`. Makes wrong-plan-bound scenarios
   visually obvious.

9. **Waterfall stage matrix** — matrix `Scenarios (rows) × Stages
(columns)`, cells `✓` (fired), `·` (passthrough), `—` (not
   applicable). Spot patterns like "PaymentTermDiscount never fires
   anywhere".

10. **Timeline invariant matrix** — matrix `Scenarios (rows) ×
Invariants TL-01…TL-07 (columns)`, cells `✓` / `✗` / `—`.

11. **Spread invariant matrix** — matrix `OneTime scenarios (rows) ×
Invariants SP-01…SP-05 (columns)`, cells `✓` / `✗` / `—`.

12. **Delta vs previous run** — if a prior report exists in
    `~/Downloads/dd-cpq-*-*.html`, parse its Overall column and
    diff. Which scenarios flipped green? Which are still red? Which
    are newly red? If no prior report, note "First run — no baseline".

13. **Green scenarios** — one line, comma-separated Keyword list,
    small grey font. For completeness.

14. **Run environment** — collapsible `<details>` block with full
    JSON of tool responses, so reruns are debuggable without re-hitting
    the org. Also include the raw `dd_cpq_list_quote_lines` response
    per scenario so the human can spot fields the report didn't
    explicitly surface. Footer — `Generated by Claude Cowork ·
dd-cpq-pricing-tester · MCP <version> · REPORT-ONLY MODE`.

**Style requirements** (baked, don't deviate):

- System font stack: `-apple-system, BlinkMacSystemFont, "Segoe UI", sans-serif`
- Monospace for Ids, currency, waterfall rows: `ui-monospace, "SF Mono", Consolas, monospace`
- Max page width `1200px` with `margin: 0 auto`
- Sticky table headers on the per-scenario tables so long scrolls
  stay readable
- Subtle borders (`#e5e7eb`), muted body text (`#374151`), bright
  accents only for badges + deltas
- **Print-friendly**: badges must retain color under
  `-webkit-print-color-adjust: exact`

**After writing the file**, print the absolute path to stdout so the
user can open it. Do not paste the HTML into the chat.

**File-write mechanism** — write the HTML **directly** using your
file system tool. **NEVER open Notepad, VS Code, TextEdit, or any
desktop text editor** — 100× slower + blocks the user's other windows.
If the runtime asks "do you want to use Notepad?", DENY and use the
direct write.

**Backwards compatible**: if the org can't provide any enriched
fields (older MCP version), render the cell as `—` and note in the
footer "some fields unavailable on this MCP version".

## Known engine bugs (surfaced 2026-09-09)

Two Apex-side replay bugs the comprehensive report found. They are
NOT fixed by this skill; document them in the report body and flag
them for Vijay:

- **Replay ignores QLI.RatePlan__c on sibling lines (PW-08 pattern)**
  — When two QLIs share a Product2 but were committed with different
  `RatePlan__c` values (Stripe percent + Stripe setup, Datadog
  Recurring + Overage, etc.), `dd_cpq_explain_price` reprices line 2
  from line 1's plan. The initial commit is correct — this is an
  explain-only bug in `CpqEngine.previewDraftsV2` / replay path.
  Symptom: TotalNet ratio mismatch between `list_quote_lines` and
  `explain_price` on siblings sharing a Product.

- **Replay ignores `RatePlan.ProrationMode__c = 'BoundaryScale'` (PW-17
  pattern)** — When a plan sets `ProrationMode__c='BoundaryScale'`,
  the initial commit correctly scales tier bounds by the proration
  factor (a 15-day partial period halves the bounds). Replay does not
  apply the scaling — it reads full-period bounds so a scenario that
  should hit `$8/unit` at the scaled bound reads `$10/unit` at the
  unscaled bound. Committed line is correct; replay disagrees.

When either of these symptoms appears, flag as HIGH in the findings
table with the pattern name and let the human decide whether to file
a Case for Vijay's fix queue.

## Common issues + how to respond

| Symptom                                            | Response                                                                             |
| -------------------------------------------------- | ------------------------------------------------------------------------------------ |
| MCP call returns 401 / OAuth expired               | Ask user to re-authenticate in Cowork; don't retry silently                          |
| Scenario Quote Id 404 (deleted)                    | Skip + note in report — don't try to re-seed                                         |
| Ambiguous expected value (trail doesn't specify)   | Skip the scenario + note "no baseline" in report                                     |
| A back-end field is not surfaced by MCP            | Log as coverage gap in scenario's Pass 2 table (grey `—`), don't invent              |
| User asks you to fix a bug                         | Refuse — this skill is read-only; the report is the deliverable                      |
| User asks you to file a Case                       | Refuse — current policy is report-only; explain the finding is already in the report |
| WebSearch returns paywalled / login-gated page     | Skip that vendor + try the next on the list                                          |
| Cowork tries to launch Notepad to write the report | Deny + use the direct file-write tool                                                |

## When to escalate to a human

- Any MCP tool returns a novel error you can't decode
- WebSearch returns pricing that fundamentally can't be modeled by
  DD CPQ (auction, dynamic bid-per-hour) — record as schema gap
- More than 5 scenarios in one run all fail on the same waterfall
  stage or the same Timeline invariant → probably a systemic bug.
  Flag prominently at the top of the report.
- A seeded RatePlan with `PlanGroupKey__c = null` appears in the org
  — call it out in the seed-compliance section AND at the top of the
  report; that's a silent-picker-hide bug that other tools may miss.
- Multi-currency scenario shows `currencyFallbackApplied = true` when
  a matching-currency plan was expected — call out in report; may be
  a seeding gap (RatePlan not in target currency) rather than an
  engine bug.
