---
name: dd-cpq-rate-plan-authoring
description: >-
  Author, update, or clone a DeepDive CPQ Rate Plan (RatePlan__c) from a
  plain-English description, using ONLY grounded MCP tool calls — never
  fabricating Product2, RatePlan, or credit-pool Ids. Covers every pattern
  the DDCPQ-2027 rebuild ships: Slack per-seat, Zoom prepay (UpfrontForTerm),
  hybrid Datadog (Recurring + Overage), Snowflake prepaid credits +
  drawdown, Stripe percent-of-basis, trial modifiers, payment-term
  early-pay discounts, and multi-currency variants. Use when the user asks
  to "add a plan", "author pricing", "create a rate plan", "draft a
  pricing model", "add a prepay tier", "wire credits", "set up drawdown",
  or names any Zoom/Slack/Snowflake/Stripe-style pattern. Skip for
  quote-side operations (Add product to quote, Apply discount, Commit
  lines) — those live in dd_cpq.list_products / configure / commit tools,
  not here.
---

# DD CPQ · Rate Plan authoring — zero-hallucination operating guide

> The direct authoring path —
> "here's an English description, produce a RatePlan record with zero
> made-up Ids." Same schema as the compiler emits, no SourceLaw stamp
> (speed path, not governance path).

## Decision tree — should I use this skill?

```
User mentions:
  ├── "add a plan" / "author pricing" / "create rate plan" / "draft a plan"    → YES
  ├── "prepay" / "annual prepay" / "Zoom Business pattern"                     → YES
  ├── "trial" / "free trial" / "14-day trial"                                  → YES
  ├── "Snowflake credits" / "prepaid credits" / "drawdown from pool"           → YES
  ├── "Stripe 2.9%" / "percent of transaction" / "AUM fee"                     → YES
  ├── "Net-30 early-pay" / "2/10 Net 30" / "payment terms discount"            → YES
  ├── "hybrid product" / "Recurring plus Overage" / "Datadog pattern"          → YES
  ├── "clone plan to EUR" / "add a currency variant"                           → YES
  ├── Reading rules back / tracing a waterfall stage → dd_cpq_list_rules
  ├── Testing what you authored → dd-cpq-configurator-tester
  └── Anything unrelated to Rate Plan / pricing model authoring                → NO
```

## The 5 orthogonal axes (mental model — memorize this)

Every RatePlan__c row combines these 5 fields. Together they express every
industry pricing pattern without special-casing.

| Axis                 | Field             | Values                                                                             | What it controls                       |
| -------------------- | ----------------- | ---------------------------------------------------------------------------------- | -------------------------------------- |
| 1 · Revenue Nature   | `revenueNature`   | Recurring · OneTime · Usage · Overage                                              | Does this count toward MRR?            |
| 2 · Pricing Model    | `pricingModel`    | FlatFee · PerUnit · Tiered · Volume · Package · PercentOfBasis                     | Which math formula                     |
| 3 · Billing Schedule | `billingSchedule` | Monthly · Quarterly · SemiAnnual · Annual · **UpfrontForTerm** · OnEvent · OneTime | How often we invoice                   |
| 4 · Billing Timing   | `billingTiming`   | InAdvance · InArrears                                                              | Before or after the period             |
| 5 · Term / Commit    | `minCommitMonths` | integer (1 = M2M, 12 = annual)                                                     | Contract length; drives NMR for prepay |

**The Normalized Monthly Rate (NMR) formula — what powers MRR/ARR:**

```
periodMonths = switch (billingSchedule):
    Monthly           → 1
    Quarterly         → 3
    SemiAnnual        → 6
    Annual            → 12
    UpfrontForTerm    → minCommitMonths      ← the Zoom fix
    OnEvent / OneTime → 0 (not MRR)

if revenueNature != 'Recurring': NMR = 0
else:                            NMR = unitPrice ÷ periodMonths
```

## The 15-field schema at your disposal

The `dd_cpq_upsert_rate_plan` tool accepts these. Only 5 are strictly
required for a valid plan; the rest are modifiers.

**Required (always):**

- `productId` — the Product2 to attach to. Get from `dd_cpq_find_products`.
- `revenueNature` · `pricingModel` · `billingSchedule` · `billingTiming` — the 5 axes.

**Common:**

- `unitPrice` — number. For PercentOfBasis: decimal (0.029 = 2.9%).
- `minCommitMonths` — integer. 1 for M2M, 12 for annual, or the prepay term.
- `displayLabel` — rep-facing pill (e.g. "Prepay 12mo · $180 · NMR $15").
- `status` — Draft / Active / Archived. **Default to Draft** — admin activates manually.
- `sortOrder` — rendering order in the picker.

**Alternatives vs additions (Phase B):**

- `planGroupKey` — same key across plans on same product = **alternatives** (rep picks ONE). Null / distinct = **additions** (all fan out as sibling QLIs).
- `isDefaultInGroup` — exactly ONE plan per group flagged true; that plan pre-selects.

**Phase-specific modifiers:**

- `trialDays` + `trialUnitPrice` (Phase D) — free/reduced trial. 0 = free.
- `basisFieldApiName` (Phase E) — REQUIRED when `pricingModel = PercentOfBasis`. Common: `QuoteLineItem.Amount`.
- `paymentTerms` (Phase F) — Net15 / Net30 / Net45 / Net60 / Net90 / Prepaid / OnReceipt.
- `paymentTermDiscountPct` (Phase F) — 2 = 2% off. Requires `paymentTerms` set.
- `isPrepaidCredit` (Phase G) — TRUE for the deposit plan (typically OneTime FlatFee $50K).
- `drawdownFromPlanId` (Phase G) — usage plan → pool plan Id. **Get from `dd_cpq_list_prepaid_credit_pools`, NEVER invent.**

## The zero-hallucination workflow (memorize this)

Follow this recipe **every time** you author a plan. Never skip a step.

```
Step 1 · Resolve the product Id
  → call dd_cpq_find_products({ q: "<name from user>" })
  → 0 matches?      → REFUSE. Ask user for a different name / SKU.
  → 1 match?        → use it.
  → 2+ matches?     → ask user which one (show name + productCode).

Step 2 · Check what's already on the product
  → call dd_cpq_rate_plan_alternatives({ productIds: [<id>] })
  → Existing plan with same schedule + price? → ask user if they want to
    UPDATE the existing planId or ADD a new alternative (different
    planGroupKey or same with different label).

Step 3 · Author the plan
  → For Phase G drawdown plans ONLY:
    → call dd_cpq_list_prepaid_credit_pools({ targetProductId: <credit product Id> })
    → 0 pools? → REFUSE. Ask user to create the pool first (IsPrepaidCredit=true plan).
    → Use the returned ratePlanId as drawdownFromPlanId.
  → Build the upsert payload matching the 5-axis + modifier schema above.
  → Set status='Draft'.
  → call dd_cpq_upsert_rate_plan({ ...payload })

Step 4 · Report to the user
  → Confirm what was created + link back to the Rate Plan Editor tab so
    the admin can review + activate.
```

## Common patterns cookbook (with exact tool payloads)

### Slack per-seat monthly

```json
{
  "action": "upsert",
  "plan": {
    "productId": "<from find_products>",
    "displayLabel": "Monthly · $12.50/user",
    "revenueNature": "Recurring",
    "pricingModel": "PerUnit",
    "billingSchedule": "Monthly",
    "billingTiming": "InAdvance",
    "unitPrice": 12.5,
    "minCommitMonths": 12,
    "status": "Draft"
  }
}
```

NMR = 12.50/1 = **$12.50/mo**.

### Zoom prepay 12mo $180 (the pattern that motivated the whole rebuild)

```json
{
  "action": "upsert",
  "plan": {
    "productId": "<from find_products>",
    "displayLabel": "Prepay 12mo · $180 → NMR $15/mo",
    "revenueNature": "Recurring",
    "pricingModel": "PerUnit",
    "billingSchedule": "UpfrontForTerm",
    "billingTiming": "InAdvance",
    "unitPrice": 180,
    "minCommitMonths": 12,
    "status": "Draft"
  }
}
```

NMR = 180/12 = **$15/mo**. The `UpfrontForTerm + minCommitMonths=12` combo is the Zoom fix.

### Zoom alternatives — rep picks one of three

Emit 3 upserts, all with the same `planGroupKey`. Mark ONE `isDefaultInGroup: true`.

```json
// Plan 1 (default)
{ "action": "upsert", "plan": { ..., "billingSchedule": "Monthly", "unitPrice": 20, "planGroupKey": "ZOOM-BASE", "isDefaultInGroup": true } }
// Plan 2
{ "action": "upsert", "plan": { ..., "billingSchedule": "Annual", "unitPrice": 150, "minCommitMonths": 12, "planGroupKey": "ZOOM-BASE" } }
// Plan 3
{ "action": "upsert", "plan": { ..., "billingSchedule": "UpfrontForTerm", "unitPrice": 180, "minCommitMonths": 12, "planGroupKey": "ZOOM-BASE" } }
```

### Trial (Phase D)

Add `trialDays` + `trialUnitPrice`. MRR calc uses the LIST unit price (Bessemer-clean); trial is transient.

```json
{ "plan": { ..., "unitPrice": 12.50, "trialDays": 14, "trialUnitPrice": 0 } }
```

### Payment terms with early-pay discount (Phase F)

```json
{ "plan": { ..., "unitPrice": 100, "paymentTerms": "Net30", "paymentTermDiscountPct": 2 } }
```

Runtime applies 2% off netPrice.

### Stripe 2.9% of txn (Phase E · PercentOfBasis)

```json
{
  "plan": {
    ...,
    "revenueNature": "Usage",
    "pricingModel": "PercentOfBasis",
    "billingSchedule": "OnEvent",
    "billingTiming": "InArrears",
    "unitPrice": 0.029,
    "basisFieldApiName": "QuoteLineItem.Amount"
  }
}
```

`basisFieldApiName` is REQUIRED with PercentOfBasis. Skip it and the validator rejects.

### Snowflake credit pool + drawdown (Phase G — the 2-plan pattern)

**Plan A · the deposit pool** (do this first):

```json
{
  "plan": {
    "productId": "<Cloud Credits product Id>",
    "displayLabel": "Prepaid · $50K credit pool",
    "revenueNature": "OneTime",
    "pricingModel": "FlatFee",
    "billingSchedule": "OneTime",
    "billingTiming": "InAdvance",
    "unitPrice": 50000,
    "isPrepaidCredit": true,
    "status": "Draft"
  }
}
```

**Plan B · the usage plan** (do this AFTER pool exists — must ground the pool Id):

```json
// 1. Call dd_cpq_list_prepaid_credit_pools to get the real pool Id.
// 2. Then:
{
  "plan": {
    "productId": "<Compute Hours product Id>",
    "displayLabel": "Compute · $2/hr (draws from credits)",
    "revenueNature": "Usage",
    "pricingModel": "PerUnit",
    "billingSchedule": "OnEvent",
    "billingTiming": "InArrears",
    "unitPrice": 2,
    "drawdownFromPlanId": "<pool ratePlanId from list_prepaid_credit_pools>",
    "status": "Draft"
  }
}
```

Then optionally call `dd_cpq_prepaid_credit_simulate` to show the admin
a what-if — "if they burn $30K in Q1 and $25K in Q2 you overrun by $5K."

## Reading a product's pricing, and bundle allocation (Case 1050)

`dd_cpq_pricing_view(productId, currencyCode?)` is the Product 360 Pricing
tab as data. Call it **before** authoring, to see what the product already has:
plans grouped by PlanGroupKey (a blank key falls back to revenue nature; a
group of 2+ is alternatives the rep picks from, with the default marked), the
bundle's one pricing mode, what a RollUp/Mixed bundle costs (baseline =
defaults, low–high across valid configurations), and `findings` such as
`DRAFT_PLAN_ONLY`, `PLAN_GROUP_KEY_BLANK` or `CHILD_NO_ACTIVE_PLAN`. Treat each
finding as a to-do to mention, not something to fix silently.

**Allocation** applies to HeaderPriced and Mixed bundles only. The header price
is split across the **Included** options by relative standalone price (their
own Active plan price, ASC 606). Priced options charge on their own and
Informational options are not performance obligations, so neither is
allocated.

- `dd_cpq_allocation_read` shows each row as `derived`, `override` or
  `missing`. A `missing` row has no Active plan, so it has no standalone price
  and needs a typed weight. It is never counted as 0.
- `dd_cpq_save_allocation(productId, weights)` writes only
  `ProductOption__c.AllocationWeight__c`. `weights` maps option id → percent,
  or → `null` to go back to derived. The result must total 100% ± 0.05, or
  nothing is saved and the reason comes back. Read it to the user.
- After a save every Included option carries a stored weight. Resetting one
  row then gives it only what the others leave; resetting **every** row
  restores the standalone-price split.
- If you show amounts, always add: these are for revenue recognition only; the
  customer is charged the header price, not their sum.

## MCP tool reference (the full authoring toolkit)

| Tool                               | Purpose                                                    | Read/Write |
| ---------------------------------- | ---------------------------------------------------------- | ---------- |
| `dd_cpq_find_products`             | Resolve product name → Product2 Id                         | Read       |
| `dd_cpq_rate_plan_alternatives`    | List existing plans on a product (shows Phase G flags too) | Read       |
| `dd_cpq_list_prepaid_credit_pools` | Discover IsPrepaidCredit=true pool Ids                     | Read       |
| `dd_cpq_upsert_rate_plan`          | Insert / update a RatePlan (this is the writer)            | **Write**  |
| `dd_cpq_archive_rate_plan`         | Soft-delete (Status='Archived'); preserves committed QLIs  | **Write**  |
| `dd_cpq_apply_rate_plan_template`  | Stamp a template pattern onto N products                   | **Write**  |
| `dd_cpq_clone_rate_plan_currency`  | Copy a plan to a new currency (MC orgs)                    | **Write**  |
| `dd_cpq_prepaid_credit_simulate`   | What-if drawdown against a quote + hypothetical usage      | Read       |
| `dd_cpq_pricing_view`              | Plans grouped, bundle composition, pricing findings        | Read       |
| `dd_cpq_allocation_read`           | A HeaderPriced/Mixed bundle's allocation rows              | Read       |
| `dd_cpq_save_allocation`           | Set allocation weights (Included only, total 100%)         | **Write**  |

## Anti-hallucination rules — non-negotiable

1. **Never invent a Product2 Id.** Always resolve via `dd_cpq_find_products`. If 0 matches, refuse and ask.
2. **Never invent a RatePlan Id.** For `drawdownFromPlanId`, always resolve via `dd_cpq_list_prepaid_credit_pools`.
3. **Never invent an Account/Quote/User Id.** These aren't needed for RatePlan authoring — if the user's request implies one, ask for it explicitly.
4. **Default to `status='Draft'`.** Never write `status='Active'` unless the user explicitly asks. Admin activates via the Editor tab after review.
5. **Refuse ambiguous requests.** "Set up a Snowflake plan" without pool amount → ask "how many credits, deposited when?" instead of guessing $50K.
6. **Never write a PolicyConstitution__c.** Constitutional CPQ was cut from v1 on 2026-09-13 and its endpoints went with it — the object no longer ships, so there is nothing to write to.
7. **Respect the 5-axis enum lists.** No invented values like `"TriMonthly"` or `"Freemium"`. The upsert tool's server-side validator will reject them anyway.
8. **PercentOfBasis needs `basisFieldApiName`.** If you emit one without the other, the validator rejects. Common values: `QuoteLineItem.Amount`, `Quote.Grand_Total__c`.
9. **`paymentTermDiscountPct > 0` needs `paymentTerms`.** Never emit a discount pct without naming the term it applies to.
10. **Report Ids you used, back to the user.** After writing, echo the resolved Product2 Id + created RatePlan Id so the user can spot-check.

## Gotchas — the sharp edges we've hit

**A plan that is not Active prices nothing.** The engine rates only Active
plans. If the cart shows no price after you author, the plan is probably still
Draft, which is this skill's default. `dd_cpq_pricing_view` reports it as
`DRAFT_PLAN_ONLY`.

**DrawdownFromPlanId points at the POOL, not the reverse.** Common mental slip: "the usage plan is the drawdown, so the pool references it." Wrong. The **usage** plan's `drawdownFromPlanId` field points at the **pool** plan's Id. If you get this backward, the simulator returns 0 drawdown + everything unmatched.

**MRR is not affected by trials.** `trialDays > 0` is transient — the Bessemer/Meritech convention says MRR is steady-state. NMR reads the LIST unit price, not the trial rate. Don't be tempted to "average in" the trial.

**Currency variants are separate rows.** In MultiCurrency orgs, each (Product × Currency) needs its own RatePlan row. Use `dd_cpq_clone_rate_plan_currency` to duplicate an existing plan; don't try to stuff multi-currency into one row.

## When in doubt, ask the user

- Ambiguous term? Ask ("do you want annual $150 or prepay 12mo at $180?").
- Multiple products match? List them and ask which.
- No pool exists but user wants drawdown? Refuse + propose creating the pool first.
- User names a pattern you don't recognize? Ask them to describe it in terms of the 5 axes.

Progressive disclosure works both ways — the tool is safe to use for
Slack-simple plans and safe to refuse on Snowflake-with-drawdown when
the user's ask is under-specified.
