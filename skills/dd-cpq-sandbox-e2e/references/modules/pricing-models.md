# Rate plans and pricing models

> Installed build 0.1.0-9 · Verified 2026-10-07 in cpq-pkg: Tiered 1–10 @ $10,
> 11–20 @ $8, 21–100 @ $6, qty 25 → $8.40/unit ($210 total); Volume same
> brackets, qty 15 → $8/unit, qty 25 → $6/unit; Package 1–10 flat $50, 11–25
> flat $90, 26–50 flat $150, qty 25 → $3.60/unit ($90 total); FlatFee $500
> list, qty 25 → $20/unit ($500 total); PerUnit $25 list → $25/unit at any
> qty; PercentOfBasis rate 0.029 (2.9%), qty 100000 (basis dollars) → netPrice
> 0.029 (per-unit display), total $2,900.

## What it is for

Every product the engine prices needs a **Rate Plan** (`RatePlan__c`) — one
row that says, for one product in one currency: what kind of revenue this is,
how often it bills, and the math that turns a quantity into a price. Rate
plans replaced the older `ProductChargeProfile__c` object; a product with no
Active Rate Plan is still priced today (as a flat monthly per-unit
subscription — see the rate plan gate below), but every newer feature —
trials, payment-term discounts, prepaid credit, term curves, multi-currency —
only works through a Rate Plan.

This module covers: the six `PricingModel__c` values and their exact math,
where a product's starting price comes from, `RevenueNature__c` /
`BillingSchedule__c` / `BillingTiming__c`, the `PlanGroupKey__c` picker rule,
MRR/ARR math, the rate plan gate, and the Rate Plan Editor.

**Not this module:** free trials and payment-term discounts
([trials-and-payment-terms.md](trials-and-payment-terms.md)), prepaid credit
and minimum commitments ([prepaid-and-commitments.md](prepaid-and-commitments.md)),
usage/overage metering with its own tier matrix on the product
([usage-pricing.md](usage-pricing.md)), term discount curves and ramps
([ramps-and-term-curves.md](ramps-and-term-curves.md)), multi-currency
([multi-currency.md](multi-currency.md)).

## How it works

### Where the starting price comes from

A line's list price seed is **the Rate Plan's `UnitPrice__c` if the winning
plan has one, otherwise the product's Standard Price (`PricebookEntry.UnitPrice`)**.
The engine always needs an Active `PricebookEntry` on the product (so it can
be added to the quote at all), but once a Rate Plan is attached, that
Pricebook price is cosmetic — the Rate Plan's `UnitPrice__c` is what actually
feeds the waterfall. Verified: a product priced at $999 on the Standard
Pricebook, with a Rate Plan `UnitPrice__c` of $25, returned `derivedListPrice`
25, not 999.

The waterfall's first stage is literally named **PricebookList** (the seed
before any override), then **ContractPrice** (an Account-specific override,
if any — see [price-rules-and-contract-prices.md](price-rules-and-contract-prices.md)),
then **DerivedListPrice** (after the Pricing Matrix's price rules). The
**PricingModel** stage runs after that — it is the stage this module is
about, and it can override `netPrice` to something that has nothing to do
with `derivedListPrice` once tiers are involved.

### The six pricing models — exact math

`PricingModel__c` on the Rate Plan picks the math. All six keep one
invariant: **`netPrice` (the Quote Line Item's `NetPrice__c`) is always the
effective PER-UNIT rate**, and the line total is `NetPrice__c × Quantity`.
Tiered, Volume and Package compute a line total first and then divide by
quantity to land back on a per-unit `NetPrice__c` — so on those three a
fractional-looking `NetPrice__c` (like `$8.40`) is normal and correct; the
total it multiplies back up to is the number that matches the tier math.

| `PricingModel__c` value | Label                           | Math                                                                                                                                                                                                                                                            | Needs tier rows? |
| ----------------------- | ------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ---------------- |
| `FlatFee`               | Flat Fee                        | Total = the plan's `UnitPrice__c`, **regardless of quantity**. `NetPrice__c` = total ÷ quantity.                                                                                                                                                                | No               |
| `PerUnit`               | Per Unit                        | `NetPrice__c` = the plan's `UnitPrice__c`, unchanged. Total = that × quantity.                                                                                                                                                                                  | No               |
| `Tiered`                | Tiered (Graduated)              | **Graduated bracket walk.** Each bracket's own rate applies only to the units that fall inside it; the brackets' sub-totals are summed, then divided by quantity.                                                                                               | Yes              |
| `Volume`                | Volume                          | **Whole-quantity, winning-bracket.** Find the one bracket the total quantity lands in; that bracket's rate applies to **every** unit, not just the ones past its floor.                                                                                         | Yes              |
| `Package`               | Package (Stairstep)             | **Flat bracket price.** Find the bracket the quantity lands in; charge its flat price for the **whole line**, however many units sit inside that bracket. `NetPrice__c` = that flat price ÷ quantity — it is not a per-unit rate at all, just displayed as one. | Yes              |
| `PercentOfBasis`        | Percent Of Basis (Stripe-style) | `UnitPrice__c` is a **decimal rate** (`0.029` = 2.9%), not a currency amount. `NetPrice__c` stays that same decimal; total = rate × Quantity.                                                                                                                   | No               |

Tier rows live on `UsageTier__c`, pointed at from the Rate Plan's
`UsageTierMatrix__c` lookup. Each row is `QtyMin__c` / `QtyMax__c` (blank max
= open-ended) / `RatePerUnit__c` / `TierIndex__c` (evaluation order). On a
`Package` plan, `RatePerUnit__c` is **reinterpreted as the flat price for
the whole bracket**, not a per-unit rate — the field name does not change,
only its meaning. If a Tiered/Volume/Package plan has **no tier rows at
all**, the engine does not refuse it — it silently falls back to `PerUnit`
math using the plan's own `UnitPrice__c`.

**`PercentOfBasis` reads Quantity as the basis amount, not a field.** The
plan carries a `BasisFieldApiName__c` (e.g. `QuoteLineItem.Amount`) that
documents which number the percentage should apply to, and the Rate Plan
Editor's Checks require it (`BASIS-1`) — but in this build the engine does
**not** read that field. It multiplies the rate by whatever Quantity the rep
typed. For a Stripe-style "2.9% of transaction volume" plan, the rep types
the estimated transaction volume (e.g. `100000` for $100K/period) into the
Quantity box, not a unit count. This is documented as the Phase E MVP
behavior, not a defect — see Limits and gotchas.

### Revenue nature, billing schedule, billing timing

Three independent picklists on the Rate Plan — do not confuse them:

- **`RevenueNature__c`** (Axis 1) — `Recurring` / `OneTime` / `Usage` /
  `Overage`. Only `Recurring` plans feed MRR/ARR. `Usage` by convention
  contributes no MRR unless the plan has a `MinimumCommitAmount__c` floor
  (see prepaid-and-commitments.md).
- **`BillingSchedule__c`** (Axis 3, invoice cadence) — `Monthly` /
  `Quarterly` / `SemiAnnual` / `Annual` / `UpfrontForTerm` (one invoice for
  the whole `MinCommitMonths__c` window — the Zoom prepay pattern) /
  `OnEvent` (per meter event) / `OneTime`.
- **`BillingTiming__c`** (advance vs. arrears) — `InAdvance` (default for
  Recurring) / `InArrears` (default for Usage/Overage).

### MRR, ARR — the formula

For a `Recurring` plan, the **Normalized Monthly Rate (NMR)** is the atomic
primitive, computed **per unit**:

```
periodMonths = Monthly:1, Quarterly:3, SemiAnnual:6, Annual:12,
               UpfrontForTerm: MinCommitMonths__c
NMR          = UnitPrice__c (post-PricingModel NetPrice__c) ÷ periodMonths
MRR (line)   = NMR × Quantity
ARR (line)   = MRR × 12
```

A non-`Recurring` plan (OneTime/Usage/Overage) reports `MRR__c` = `ARR__c` =
`0` on its line — a one-time fee or a metered line never inflates MRR. There
is no `TCV__c` field; a one-time line's total contract value is simply
`NetPrice__c × Quantity` (its `Amount`), and a recurring line's is
`ARR__c × (term in years)`.

A product with **no Rate Plan at all** falls back to the legacy formula:
`MRR = NetPrice__c × Quantity` with **no division by billing cadence** — a
quarterly list price is reported three times too high, an annual one twelve
times too high. This is the strongest reason to always author a Rate Plan.

### `PlanGroupKey__c` — must equal `RevenueNature__c`

**Rule (2026-09-10 design, current in this build): `PlanGroupKey__c` must
equal the plan's own `RevenueNature__c`** — one of exactly `Recurring` /
`OneTime` / `Usage` / `Overage`. There is no other valid value, and the Rate
Plan Editor's save button stamps it automatically from the charge type if
left blank. The Rate Plan Editor's Checks panel does **not** expose a
"Plan Group Key" input any more — it is derived, not typed.

What the key controls: **plans that share a key on the same product are
ALTERNATIVES** — the rep sees them as chips in the cart's plan picker and
picks one (a swap, not an addition). A product with 3 Recurring plans and 2
OneTime plans gets **two** chip groups — one per charge type — and the rep
picks one chip from each; picking a OneTime chip adds a second line, it does
not replace the Recurring one.

**Multiple charge-type "hybrids" on one product (e.g. Datadog: Recurring
base + Overage) are two separate `RatePlan__c` rows on the same
`TargetProduct__c`, each with its own `RevenueNature__c`/`PlanGroupKey__c`.**
They are NOT alternatives of each other (different keys), so both fan out
onto the quote as separate lines automatically — the rep does not pick
between a base fee and its overage.

Within one group, the winner is: **the rep's explicit pick (if any) >
`IsDefaultInGroup__c` (exactly one plan per group should carry this) >
`SortOrder__c` ascending.** Setting a new plan's `IsDefaultInGroup__c` does
not automatically clear the flag on its siblings — the Rate Plan Editor's
**Default** card action does that; hand-editing the field in Setup will not.

### The rate plan gate — Block / Warn / Off

Custom metadata type **DD CPQ Rate Plan Settings** (`DD_CPQ_Rate_Plan_Settings__mdt`),
one record, developer name `Default`, field **Enforcement**
(`Enforcement__c`). This build ships with it set to **`Block`**
(verified in cpq-pkg: `SELECT DeveloperName, Enforcement__c FROM
DD_CPQ_Rate_Plan_Settings__mdt` → `Default`, `Block`).

| Value   | Label                          | What happens                                                                                                                |
| ------- | ------------------------------ | --------------------------------------------------------------------------------------------------------------------------- |
| `Block` | Block (intended resting state) | A line with no Active plan cannot be committed. Refused with the exact message below.                                       |
| `Warn`  | Warn only (migration state)    | The line is flagged with a warning on every surface, but commit proceeds. For a catalogue mid-migration, not a destination. |
| `Off`   | Off (not recommended)          | No check. The silence this feature exists to end.                                                                           |

**Who is exempt, under every setting:** a **bundle parent priced by
roll-up** (its price is a sum of its children, so it is never itself
"rated"), and a bundle option whose `PricingRole__c` is **Included** or
**Informational** — the engine never prices those lines either, so asking
them for a plan would be asking a question nobody needs answered. A bundle
option with a **blank** pricing role is **not** exempt — it is treated as
charged, and needs a plan.

**Exact refusal message (Block, at commit):**

> _"One line cannot be priced: \<Product Name\>. Each needs an Active rate
> plan before it can be quoted. Bundle parents priced by roll-up and options
> marked Included or Informational are exempt and are not in this list."_

(For more than one offending line: `"N lines cannot be priced: <names,
comma-joined>. …"`.) This refusal fires only **at commit**, never at preview
— a priced preview shows the warning but still returns numbers (the flat
monthly fallback), so the rep can see what is wrong before they try to
commit.

**Where an admin changes it:** Setup → Quick Find **"Custom Metadata
Types"** → **DD CPQ Rate Plan Settings** → **Manage Records** → **Default**
→ **Edit** → Enforcement. Do **not** change it in cpq-pkg — this is shared
org-wide state other module writers depend on; leave it on `Block`.

### `ProductTier__c` is not a pricing-tier object — do not confuse it with `UsageTier__c`

The name is a trap. **`ProductTier__c` has nothing to do with Tiered/Volume/
Package pricing.** It assigns an ordinal **edition rank** to a product
(`Rank__c`, `TierLabel__c`, e.g. "Professional" = 20, "Enterprise" = 30) so a
compatibility rule can require "at least Professional" as arithmetic
(`30 >= 20`) instead of naming one exact edition. It is read by
`ProductTierService` for **compatibility/eligibility rules**, not by
`PricingModelService`. The bracket math in this module's "six pricing
models" table comes entirely from **`UsageTier__c`** rows pointed at by the
Rate Plan's `UsageTierMatrix__c` lookup. If a tester asks "where do I put my
tiered pricing bands," the answer is the Rate Plan Editor's **Tiers** tab —
never Product Tier.

## Objects and fields

| Object label (API name)                 | Field                                                     | Meaning / values                                                                                                                                                        |
| --------------------------------------- | --------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Rate Plan (`RatePlan__c`)               | `TargetProduct__c`                                        | The product this plan prices. Lookup to `Product2`.                                                                                                                     |
|                                         | `Status__c`                                               | `Draft` (default) / `Active` (only these are rated) / `Archived`.                                                                                                       |
|                                         | `RevenueNature__c`                                        | `Recurring` (default) / `OneTime` / `Usage` / `Overage`.                                                                                                                |
|                                         | `BillingSchedule__c`                                      | `Monthly` (default) / `Quarterly` / `SemiAnnual` / `Annual` / `UpfrontForTerm` / `OnEvent` / `OneTime`.                                                                 |
|                                         | `BillingTiming__c`                                        | `InAdvance` (default) / `InArrears`.                                                                                                                                    |
|                                         | `PricingModel__c`                                         | `FlatFee` / `PerUnit` (default) / `Tiered` / `Volume` / `Package` / `PercentOfBasis`.                                                                                   |
|                                         | `UnitPrice__c`                                            | Currency(18,4). The authored per-unit price (or flat total for FlatFee, or decimal rate for PercentOfBasis). Overrides the Pricebook list price when set.               |
|                                         | `UsageTierMatrix__c`                                      | Lookup to `UsageTierMatrix__c`. Required (by the Checks panel) for Tiered/Volume/Package.                                                                               |
|                                         | `BasisFieldApiName__c`                                    | Text(80). Dotted field name for PercentOfBasis's basis. Authored but not yet read by the engine (see Limits).                                                           |
|                                         | `PlanGroupKey__c`                                         | Text(40). Must equal `RevenueNature__c`. Blank is auto-filled with the charge type on save.                                                                             |
|                                         | `IsDefaultInGroup__c`                                     | Checkbox. One true flag per group picks the rep's first-shown chip.                                                                                                     |
|                                         | `SortOrder__c`                                            | Number. Tie-breaker when no default is flagged.                                                                                                                         |
|                                         | `MinCommitMonths__c` / `MaxCommitMonths__c`               | Number. Term window; also the divisor for `UpfrontForTerm` NMR.                                                                                                         |
|                                         | `MinimumCommitAmount__c`                                  | Currency. A dollar floor on usage MRR (see prepaid-and-commitments.md).                                                                                                 |
|                                         | `TrialDays__c` / `TrialUnitPrice__c`                      | See trials-and-payment-terms.md.                                                                                                                                        |
|                                         | `PaymentTerms__c` / `PaymentTermDiscountPct__c`           | See trials-and-payment-terms.md.                                                                                                                                        |
|                                         | `IsPrepaidCredit__c` / `DrawdownFromPlanId__c`            | See prepaid-and-commitments.md.                                                                                                                                         |
|                                         | `TermDiscountCurve__c`                                    | See ramps-and-term-curves.md.                                                                                                                                           |
|                                         | `RoundingMode__c` / `ProrationMode__c`                    | More settings — how NMR and mid-period charges round.                                                                                                                   |
| Usage Tier Table (`UsageTierMatrix__c`) | `Status__c`                                               | `Draft` (default) / `Active` / `Inactive`.                                                                                                                              |
|                                         | `PricingMode__c`                                          | `Volume` (default) / `Graduated` — label only; the Rate Plan's own `PricingModel__c` is what actually drives the math in `PricingModelService`.                         |
|                                         | `Currency__c`                                             | Optional scope.                                                                                                                                                         |
| Usage Tier (`UsageTier__c`)             | `Matrix__c`                                               | Master-detail to `UsageTierMatrix__c`.                                                                                                                                  |
|                                         | `TargetProduct__c`                                        | Lookup to `Product2`. `PricingModelService` queries tiers **by this field directly**, not by walking the Matrix — a tier row with no `TargetProduct__c` is unreachable. |
|                                         | `QtyMin__c` (required) / `QtyMax__c` (blank = open-ended) | The bracket bounds.                                                                                                                                                     |
|                                         | `RatePerUnit__c` (required)                               | Per-unit rate (Tiered/Volume) or flat bracket price (Package).                                                                                                          |
|                                         | `TierIndex__c` (required)                                 | Evaluation order.                                                                                                                                                       |
| Product Tier (`ProductTier__c`)         | `Product__c`, `Rank__c`, `TierLabel__c`                   | Edition ranking for compatibility rules — **not** a pricing table (see above).                                                                                          |
| Quote Line Item (standard, extended)    | `DerivedListPrice__c`                                     | List price after the Pricing Matrix's price rules, before PricingModel math.                                                                                            |
|                                         | `NetPrice__c`                                             | Final per-unit price after every stage, including PricingModel.                                                                                                         |
|                                         | `RatePlan__c`                                             | Lookup to the winning plan.                                                                                                                                             |
|                                         | `NormalizedMonthlyRate__c`, `MRR__c`, `ARR__c`            | See MRR/ARR formula above.                                                                                                                                              |

## Build it (tester)

Do this after the golden path (Stages 1–6), on your own products — do not
touch CRM Suite Pro, Sales Cloud or the other golden-quote products.

1. **Products tab → New.** Name it `PM Tiered Widget`. **Active** ticked.
   Save. Related tab → Price Books → **Add Standard Price** → any value
   (e.g. `999`) → Active → Save. (This Pricebook price is a placeholder —
   the Rate Plan below is what actually prices the line.)
2. **Rate Plan Editor tab** → open `PM Tiered Widget` → it shows
   **Start this product's pricing** → **Blank plan**.
3. In the inspector: **Shape** → Charge type **Recurring**, Pricing model
   **Tiered (Graduated)**, Billing schedule **Monthly**.
4. **Tiers** tab appears → **+ Add tier** three times:
   - From `1` To `10`, Rate `10`
   - From `11` To `20`, Rate `8`
   - From `21` To `100`, Rate `6`
5. **Preview** → type `25` in the quantities box. You should see per-unit
   **$8.40** (summed brackets: 10×$10 + 10×$8 + 5×$6 = $210 ÷ 25).
6. Set **Status** to **Active** → **Save**. The Checks panel should show no
   blocking errors (if `AXIS-4` "No price" complains, the plan still needs a
   `UnitPrice__c` even though tiers drive the math — set it to any small
   fallback value, e.g. `5`).
7. Repeat for `PM Volume Widget` (same three tiers, **Pricing model
   Volume**) and `PM Package Widget` (tiers `1–10` rate `50`, `11–25` rate
   `90`, `26–50` rate `150`, **Pricing model Package (Stairstep)** — note
   the Rate column here means the whole bracket's flat price, not a
   per-unit rate).
8. Build a quote on an Account/Opportunity of your own (e.g. `PM Test Co`)
   and **Add products** → each widget at quantity **25** (quantity **15**
   for a second Volume test). Read the net price shown per line against
   "Expected numbers" below.

## Check it (Claude)

```bash
sf data query -o cpq-pkg -q "SELECT DDCPQ__TargetProduct__r.Name, DDCPQ__Status__c, DDCPQ__PricingModel__c, DDCPQ__RevenueNature__c, DDCPQ__BillingSchedule__c, DDCPQ__UnitPrice__c, DDCPQ__PlanGroupKey__c FROM DDCPQ__RatePlan__c WHERE DDCPQ__TargetProduct__r.Name LIKE 'PM %'"
sf data query -o cpq-pkg -q "SELECT DDCPQ__TierIndex__c, DDCPQ__QtyMin__c, DDCPQ__QtyMax__c, DDCPQ__RatePerUnit__c FROM DDCPQ__UsageTier__c WHERE DDCPQ__TargetProduct__r.Name = 'PM Tiered Widget' ORDER BY DDCPQ__TierIndex__c"
sf data query -o cpq-pkg -q "SELECT Product2.Name, Quantity, DDCPQ__DerivedListPrice__c, DDCPQ__NetPrice__c, DDCPQ__MRR__c, DDCPQ__ARR__c FROM QuoteLineItem WHERE Quote.Name = 'PM Test Quote'"
```

Expect: every `PM …` Rate Plan `Active`, `PlanGroupKey__c` = `Recurring`
(equal to `RevenueNature__c`); 3 tier rows for the Tiered product in
ascending `TierIndex__c`; the QLI query's `NetPrice__c` matching the
"Expected numbers" table.

Read-only price preview (no commit) against a quote and product Ids you
query first:

```json
{
  "quoteId": "<0Q0…>",
  "selections": [
    {
      "localKey": "t1",
      "productId": "<PM Tiered Widget 01t…>",
      "quantity": 25,
      "sourceBundleId": "<PM Tiered Widget 01t…>"
    }
  ]
}
```

```bash
sf api request rest "services/apexrest/DDCPQ/dd/v1/price?includeWaterfall=true" -o cpq-pkg --method POST --body @body.json
```

Look at `lines[0].netPrice` and `lines[0].derivedListPrice` in the response —
`derivedListPrice` is the plan's raw `UnitPrice__c` (the seed), `netPrice` is
the PricingModel stage's output (the real per-unit charge).

## Expected numbers

All verified 2026-10-07 in cpq-pkg via the read-only price preview above
(tier/plan setup as in "Build it"; PM Percent Widget has no tiers, just a
PercentOfBasis plan at UnitPrice__c = 0.029).

| Product           | Pricing model  | Qty    | derivedListPrice (seed) | NetPrice__c | Line total |
| ----------------- | -------------- | ------ | ----------------------- | ----------- | ---------- |
| PM Tiered Widget  | Tiered         | 25     | 5                       | **8.40**    | 210.00     |
| PM Volume Widget  | Volume         | 15     | 5                       | **8.00**    | 120.00     |
| PM Volume Widget  | Volume         | 25     | 5                       | **6.00**    | 150.00     |
| PM Package Widget | Package        | 25     | 5                       | **3.60**    | 90.00      |
| PM Flat Widget    | FlatFee        | 25     | 500                     | **20.00**   | 500.00     |
| PM PerUnit Widget | PerUnit        | 25     | 25                      | **25.00**   | 625.00     |
| PM Percent Widget | PercentOfBasis | 100000 | 0.029                   | **0.029**   | 2,900.00   |

For the Tiered row: bracket walk is 10 units times $10, plus 10 units times
$8, plus 5 units times $6, equals $100 + $80 + $30 = $210, divided by 25 is
$8.40. For Package: quantity 25 falls in the 11-25 bracket, whose flat price
is $90 regardless of exactly how many of those 25 slots are used; $90
divided by 25 is $3.60.

## Troubleshooting

| Symptom                                                                                                            | Cause                                                                                                                                                                              | Fix                                                                                                                                                                                                          |
| ------------------------------------------------------------------------------------------------------------------ | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| Tiered/Volume/Package product prices at its plain UnitPrice__c (e.g. $5), not a tiered number                      | No tier rows reach PricingModelService, either none exist or they are not linked by TargetProduct__c                                                                               | Open the plan's Tiers tab and confirm rows exist; check that a query on DDCPQ__UsageTier__c filtered by DDCPQ__TargetProduct__r.Name returns rows with DDCPQ__TargetProduct__c populated, not just Matrix__c |
| Volume model gives a different number than expected                                                                | A Volume bracket's rate applies to the whole quantity, not just the units above its floor, the opposite of Tiered                                                                  | Re-check which bracket the quantity falls in; Volume has no partial bracket, unlike Tiered                                                                                                                   |
| Package model's number does not scale when more units are added within the same bracket                            | By design, Package charges one flat price per bracket, however many units sit inside it                                                                                            | Not a bug. For per-block-of-N scaling, author narrower brackets instead                                                                                                                                      |
| PercentOfBasis total looks way too big or too small                                                                | Quantity is being read as the basis dollar amount, not a unit count. A rep who types 5 seats instead of 100000 dollars in expected transactions gets 5 times 2.9%, which is $0.145 | Re-enter Quantity as the basis amount in dollars, not units                                                                                                                                                  |
| AXIS-4, "No price," blocks Activate on a Tiered/Volume/Package plan even though tiers are filled in                | UnitPrice__c is still required as the fallback seed, regardless of pricing model                                                                                                   | Set any UnitPrice__c; it is only used if tier rows are missing or emptied later                                                                                                                              |
| Commit refused: "...Each needs an Active rate plan..."                                                             | DD_CPQ_Rate_Plan_Settings__mdt Default.Enforcement__c is Block and a rated line has no Active plan                                                                                 | Author and Activate a Rate Plan for that product; confirm with the Rate Plan query in "Check it"                                                                                                             |
| A chip in the cart's plan picker swaps the wrong line, or adds a new line instead of swapping                      | PlanGroupKey__c on the two plans does not actually match, or does not equal their own RevenueNature__c                                                                             | Query both plans by TargetProduct__c; every row's RevenueNature__c and PlanGroupKey__c must be identical                                                                                                     |
| Rate Plan Editor's Needs attention lists a product under TIERS_MISSING even though tier rows are visible on screen | The tier rows' Matrix__c points at a different UsageTierMatrix__c than the plan's UsageTierMatrix__c lookup                                                                        | Both must reference the same Usage Tier Table record                                                                                                                                                         |
| MRR/ARR on a quarterly or annual line looks 3x or 12x too high                                                     | The line has no Rate Plan, so it fell back to the legacy NetPrice times Quantity formula with no cadence division                                                                  | Attach and Activate a Rate Plan with the correct BillingSchedule__c                                                                                                                                          |

## Limits and gotchas

- BasisFieldApiName__c is schema-only in this build. The engine never reads
  the field it names; PercentOfBasis always multiplies the rate by whatever
  Quantity was typed. This shipped deliberately as the Phase E MVP
  (2026-09-02); it is a stated scope limit, not a pending fix.
- No per-block-of-N "buy another block" Package math. Package is a flat
  price for whichever bracket the quantity lands in, not Zoom-style "every
  10 licenses is another $100 block" scaling. Model that shape with narrow,
  contiguous brackets instead.
- A Tiered/Volume/Package plan with zero tier rows does not error; it
  silently prices as PerUnit off the plan's own UnitPrice__c. This can hide
  a configuration mistake; always check the tier query in "Check it" when a
  number looks like a plain per-unit price and should not.
- DDCPQ-67's 2,000-line cart limit, DDCPQ-112's waterfall discount percent
  precision fix, and DDCPQ-69's rule-behind-each-step waterfall labels are
  not in this build; they land after commit 31444bb.
- Governor limits: UsageTierService and PricingModelService load tier rows
  in one bulk SOQL per preview or commit call, keyed by product Id. Fine for
  a normal cart, but a quote with hundreds of distinct tiered products in
  one request multiplies that one query's row count accordingly.
- Currency: a Rate Plan is scoped to one currency implicitly through the
  org's multi-currency setup; see multi-currency.md for the fallback-to-USD
  behavior when a product has no plan in the quote's currency.

## Questions testers ask

Q: Why did my Tiered product's net price come out as a decimal like $8.40,
not a round number?
A: NetPrice__c is always displayed as a per-unit rate. For Tiered/Volume/
Package it is a line total divided back down by quantity, so it rarely
lands on a round number. Check the line total (NetPrice__c times Quantity)
against the tier math instead.

Q: Tiers are set up, but the product still charges the plain UnitPrice__c.
What is missing?
A: Almost always the tier rows exist but their TargetProduct__c lookup is
blank, or the plan's PricingModel__c is still PerUnit. Both are required;
the Tiers tab being visually filled in is not enough on its own.

Q: Does PricingModel__c ever change which object stores the tier rows?
A: No. Tiered, Volume and Package always read UsageTier__c rows, by
TargetProduct__c. The UsageTierMatrix__c lookup on the plan exists for the
Rate Plan Editor's UI grouping and the "shared matrix, saving forks a copy"
behavior; the pricing engine itself queries by product, not by matrix.

Q: Can one product have both a Recurring plan and a Usage or Overage plan?
A: Yes, that is the Datadog-style hybrid pattern. They are two RatePlan__c
rows on the same product with different RevenueNature__c values (so
different PlanGroupKey__c values too), and both fan out onto the quote as
separate lines automatically; the rep does not choose between them.

Q: What is ProductTier__c for, then, if not pricing tiers?
A: Edition ranking for compatibility rules, such as "requires at least
Professional." See "Objects and fields" above; do not author pricing bands
there.

Q: The rate plan gate is set to Block. Why did adding the product to the
cart still show a price instead of refusing immediately?
A: The gate only refuses at commit. Preview, and the cart before committing,
always shows a number, the flat-monthly fallback, with a warning attached,
so the problem can be seen before trying to save.

Q: Why does the Rate Plan Editor no longer allow typing a Plan Group Key?
A: It used to accept any string; the 2026-09-10 design change made the key
always equal the charge type, auto-stamped on save, because free-typed keys
were the top source of misconfigured plan pickers.

Q: Is there a Total Contract Value field?
A: No TCV__c field exists. For a one-time line, its total is NetPrice__c
times Quantity. For a recurring line, multiply ARR__c by the contract
length in years; there is no stored field that already does this.
