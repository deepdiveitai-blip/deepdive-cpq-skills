# Usage and consumption pricing

> Installed build 0.1.0-9 · Verified 2026-10-07 in org **cpq-pkg**: a Graduated
> tiered Usage plan (1–10 @ $5, 11–50 @ $4, 51+ @ $3) at qty 30 → net
> **$4.333333/unit** ($130 total, MRR/ARR **$0**); a standalone Overage plan
> (PerUnit $0.05) at qty 20 → net **$0.05/unit** ($1 total, MRR/ARR **$0**);
> the same tiered product as a bundle option with Minimum Commit 10 /
> Overage Rate $1 — qty 30 (over the allowance) → net **$5.00/unit** ($150 =
> $130 tier + $20 overage); qty 5 (under the allowance) → net **$10.00/unit**
> (billed as if 10 were bought).

## What it is for

Usage pricing is how DD CPQ charges for consumption that is measured, not
just bought — API calls, GB-months, metered minutes, AI tokens. It covers
three distinct things that often get asked about together:

1. **`RevenueNature__c = Usage` or `Overage` on a Rate Plan** — the two
   picklist values that mark a line as consumption-based rather than a flat
   subscription. They behave almost identically; `Overage` exists so a
   second Rate Plan on the same or a different product can say "this is the
   charge for going past an included allowance," distinct from the base
   metered charge.
2. **`UsageTierMatrix__c` / `UsageTier__c`** — the bracket table a Tiered,
   Volume or Package Rate Plan reads to turn a quantity into a rate. This is
   the same tier engine `pricing-models.md` documents for Tiered/Volume/
   Package math in general — this module covers it specifically in its
   Usage/Overage home, plus the two features that only matter for metered
   lines: **Minimum Commit** (an included allowance) and **Overage Rate**
   (what you pay past it).
3. **Consumption Schedule parity (DDCPQ-2027-M)** — four small, real
   features that copy Salesforce CPQ's "Consumption Schedule" behavior
   piece by piece: Category-based rating (M-3), multi-period Segments for
   ramps (M-2), first-period proration of tier boundaries (M-4), and the
   `OrderItemRatePlan__c` junction that survives Order sync (M-1).

**Not this module:** the six `PricingModel__c` values and their general
math, MRR/ARR, the rate plan gate
([pricing-models.md](pricing-models.md)); prepaid credit pools and
drawdown, minimum-commitment floors, and commitment-discount rules
([prepaid-and-commitments.md](prepaid-and-commitments.md)) — read that
module for anything about a dollar pool or an account-level spend floor,
even though both sit near usage conceptually.

## How it works

### Usage and Overage never touch MRR/ARR — by design

`NormalizedRateService.compute` returns `normalizedMonthlyRate = 0` whenever
`RevenueNature__c != 'Recurring'`. Usage and Overage lines get `MRR__c = 0`
and `ARR__c = 0` on every committed QuoteLineItem, no matter the quantity or
rate. This is the "Bessemer-clean" convention: MRR means _recurring_
revenue, and metered consumption is variable by nature. The one exception —
`CommittedUsageMRR__c` — is a **separate** field (see below), never folded
into `MRR__c`.

Verified: a Usage Tiered line billing $130/mo (qty 30 × effective $4.333/unit)
committed with `MRR__c = 0`, `ARR__c = 0`. An Overage line billing $1
committed the same way.

### The tier engine: Tiered, Volume, Package — shared with pricing-models.md

`PricingModel__c` on the winning Rate Plan picks the math; the bracket rows
live on `UsageTier__c`, looked up by `TargetProduct__c` directly (not by
walking the parent `UsageTierMatrix__c` first), filtered to `Status__c =
'Active'` matrices and the quote's currency (blank `Currency__c` = catch-all).
`pricing-models.md` has the full Tiered/Volume/Package table; the two things
specific to a Usage/Overage line are Minimum Commit and Overage Rate, next.

**Mode precedence** (`UsageTierService.resolveMatrixPricingMode`): the
matrix header's `PricingMode__c` wins when set; otherwise the winning tier's
own `PricingMode__c`; otherwise `Volume`. Author the mode on the matrix
header, not per-tier — the per-tier field is a legacy fallback.

**No tiers on a Tiered/Volume/Package plan silently falls back to PerUnit**
math using the plan's own `UnitPrice__c` — the engine never refuses to price
the line. The Rate Plan Editor's Checks panel is stricter: finding `TIER-1`
blocks **saving** such a plan as Active with zero tier rows ("A Tiered plan
prices from its tiers, and it has none. Add at least one tier.").

### Minimum Commit + Overage Rate — the included-allowance pattern

`ProductOption__c.MinimumCommit__c` and `ProductOption__c.OverageRate__c`
are the fields that model "100 included, then $X per unit over" — **but
they only exist on a bundle option**, not on a standalone product's Rate
Plan. To use them, the metered product has to be a priced option inside a
bundle, with active `UsageTier__c` rows for that same product (if there are
no tiers at all, `UsageTierService` skips the line entirely and these two
fields are never read).

The math, verified at qty 30 against a 10-unit Minimum Commit and a $1
Overage Rate on the 3-bracket matrix above:

```
billableQty   = max(actualQty, minimumCommit)        = max(30, 10) = 30
tierBillable  = graduated bracket walk on billableQty = 10×$5 + 20×$4 = $130
overageQty    = max(0, actualQty − minimumCommit)     = max(0, 20) = 20
overageBill   = overageQty × overageRate              = 20 × $1 = $20
totalBillable = tierBillable + overageBill            = $150
perUnitRate   = totalBillable / actualQty             = $150 / 30 = $5.00
```

Below the commit, the customer still pays for the **committed** volume, not
the actual one — verified at qty 5 against the same 10-unit commit:
`billableQty = max(5, 10) = 10`, `tierBillable = 10 × $5 = $50` (all of it
in bracket 1), `perUnitRate = $50 / 5 = $10.00`. The rep typed 5; the bill
reflects 10.

Downstream discount stages (System, Volume, Manual, Commitment) run on this
blended `perUnitRate`, not on the raw tier rate — a volume discount composes
correctly whether the line's price came from a plain tier walk or from a
commit+overage blend.

### Consumption Schedule parity (DDCPQ-2027-M) — four small features

- **M-1, Order integration.** `OrderItemRatePlanLinkTrigger` (after-insert
  on `OrderItem`) calls `OrderItemRatePlanLinkService.link`, which inserts
  one `OrderItemRatePlan__c` junction row per OrderItem whose source QLI has
  a `RatePlan__c` set. This is purely an **observer** — rule 8 compliance —
  native Sync-to-Order still creates the Order/OrderItem; the trigger only
  copies the `RatePlan__c` reference sideways so a downstream billing system
  can read the plan off the Order side without hopping through
  `OrderItem.QuoteLineItemId`. Legacy lines still on
  `ProductChargeProfile__c` (no `RatePlan__c`) are silently skipped — this
  only covers the DDCPQ-2027-A+ Rate Plan pathway.
- **M-2, Segments.** If a `RampSchedule__c` matching the line carries a
  `UsageTierMatrixOverride__c` and the line's `termMonths > 12`, tier math
  runs per ramp-year against whichever matrix each year resolves to (the
  override matrix, or the base plan's matrix if that year has none), then
  weighted-averages the per-unit rate by months-per-period. A term of 12
  months or less, or no reachable override, falls back to the single-matrix
  path untouched.
- **M-3, Category-based rating.** If any `UsageTier__c` row on a matrix sets
  `Category__c`, tiers are grouped by category and each group is priced
  independently; the matrix's `CategoryRatingMethod__c` (`Highest`, the
  SBQQ default, or `Lowest`) picks which group's `totalBillable` wins. Tiers
  with a blank `Category__c` run the plain ungrouped path — no regression.
- **M-4, Proration.** A Usage/Overage line on a Rate Plan with
  `ProrationMode__c = 'BoundaryScale'` and a partial first period scales the
  tier _boundaries_ (`QtyMin__c`/`QtyMax__c`) by the prorated fraction —
  **not** the per-unit rates — so a customer who only consumes for half a
  month crosses tier thresholds proportionally sooner. `IsProrated__c` /
  `ProratedDays__c` get stamped on the QLI.

### How estimated usage is entered on a quote

There is no separate "estimated usage" field — the rep types the forecast
straight into the cart line's **Quantity** box, same box as any other
product. For `PercentOfBasis` Stripe-style plans the quantity box holds a
basis _dollar amount_, not a unit count (see pricing-models.md); for a plain
Usage/Overage plan it is literally the forecast unit count for the period
(e.g. "30" GB, "5000000" tokens). There is no forecast slider, no
non-destructive what-if — changing the quantity, repricing, and changing it
back is the only way to try a second scenario in this build.

### The cart line's Usage/Overage display — more than the design docs claim

`cpqCartLine.js`'s `productTypeLabel` badges a line **Usage**, **Overage**,
**Recurring**, or **One-time** straight from `chargeType` (exact strings).
Contrary to a line in `DD_CPQ_2027_USAGE_QUOTING_UX_ANALYSIS.html` ("no
MRR/ARR row for Usage lines — nowhere"), the committed build **does** render
an ARR line for Usage/Overage via `arrFormatted`: with no `CommittedUsageMRR__c`
floor and a priced quantity it reads **"variable · usage"**; with a floor
set, it reads **"$X ARR · projected"** when the projection exceeds the
floor, or **"$X ARR · committed"** when the floor is the larger number
(`MAX(committedFloor×12, netPrice×qty×12)` — the same MAX semantics as the
minimum-commitment floor in prepaid-and-commitments.md, applied here to a
single plan's own floor). The metric _row_ with NMR/MRR/ARR/Committed badges
(`showMetricRow`) still only renders for Recurring lines, or for Usage/
Overage lines that carry a `CommittedUsageMRR__c` floor — a pure metered
line with no floor shows neither the metric row nor a dollar figure, just
"variable · usage". Read the doc's other findings (tier brackets three
clicks deep in the waterfall, no prepaid-burndown visualizer in the cart, no
forecast slider) as still accurate — code confirms those gaps too.

## Objects and fields

| Object label (API name)                                 | Field                                                                           | Meaning / values                                                                                                                                                                                  |
| ------------------------------------------------------- | ------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Rate Plan (`RatePlan__c`)                               | `RevenueNature__c`                                                              | `Usage` — metered per event, no MRR unless `MinimumCommitAmount__c` is set. `Overage` — pay beyond an included allowance, Usage-like, same MRR=0 rule.                                            |
|                                                         | `PricingModel__c`                                                               | `Tiered` / `Volume` / `Package` need a `UsageTierMatrix__c`; `PerUnit` / `FlatFee` don't.                                                                                                         |
|                                                         | `UsageTierMatrix__c`                                                            | Lookup to the tier table. Required by the Checks panel (`TIER-1`) for Tiered/Volume/Package before the plan can go Active.                                                                        |
|                                                         | `MinimumCommitAmount__c` / `MinCommitMonths__c`                                 | A **dollar** floor on this one plan's MRR (`CommittedUsageMRR__c`) — see prepaid-and-commitments.md. Not the same thing as `ProductOption.MinimumCommit__c` below, which is a **quantity** floor. |
|                                                         | `ProrationMode__c`                                                              | `BoundaryScale` prorates tier boundaries for a partial first period (M-4); other values leave tiers unprorated.                                                                                   |
| Usage Tier Table (`UsageTierMatrix__c`)                 | `Status__c`                                                                     | `Draft` (default) / `Active` (only these are read by `UsageTierService`) / `Inactive`.                                                                                                            |
|                                                         | `PricingMode__c`                                                                | `Volume` (default) / `Graduated` — the header value wins over any per-tier value.                                                                                                                 |
|                                                         | `Currency__c`                                                                   | ISO code; blank = catch-all across currencies.                                                                                                                                                    |
|                                                         | `CategoryRatingMethod__c`                                                       | `Highest` (default, SBQQ convention) / `Lowest` — which `Category__c` group wins when tiers are categorized (M-3).                                                                                |
| Usage Tier (`UsageTier__c`)                             | `Matrix__c`                                                                     | Master-detail to `UsageTierMatrix__c`.                                                                                                                                                            |
|                                                         | `TargetProduct__c`                                                              | Lookup to `Product2` — the field `UsageTierService` actually filters on; a tier with this blank is unreachable.                                                                                   |
|                                                         | `QtyMin__c` (required) / `QtyMax__c` (blank = open-ended)                       | Bracket bounds.                                                                                                                                                                                   |
|                                                         | `RatePerUnit__c` (required)                                                     | Per-unit rate (Tiered/Volume) or flat bracket price (Package — see pricing-models.md).                                                                                                            |
|                                                         | `TierIndex__c` (required)                                                       | Evaluation order.                                                                                                                                                                                 |
|                                                         | `Category__c`                                                                   | Groups tiers for Category-based rating (M-3); blank = ungrouped (today's default path).                                                                                                           |
|                                                         | `FieldApiName__c` / `Operator__c` / `Value__c` / `ValueType__c` / `Priority__c` | Shared 5-field criteria (CLAUDE.md rule 5) — empty = catch-all.                                                                                                                                   |
|                                                         | `Conditions__c`                                                                 | Rules-Studio-written advanced JSON (`{"logic":"1 AND (2 OR 3)", "conditions":[...]}`); when filled, overrides the single Field/Operator/Value triple.                                             |
| Product Option (`ProductOption__c`)                     | `MinimumCommit__c`                                                              | Guaranteed minimum **quantity** per period for this bundle option — `billableQty = MAX(actualQty, this)`. Null = no commit.                                                                       |
|                                                         | `OverageRate__c`                                                                | Per-unit rate charged on quantity past `MinimumCommit__c`. Null = no overage. Only read when the option's product also has active `UsageTier__c` rows.                                            |
| Quote Line Item (standard, extended)                    | `UsageTierPrice__c`                                                             | The blended per-unit rate `UsageTierService` computed (tier math ± commit/overage) — distinct from `NetPrice__c`, which is this value after downstream discounts.                                 |
|                                                         | `CommittedUsageMRR__c`                                                          | `RatePlan.MinimumCommitAmount__c / MinCommitMonths__c` — a dollar floor, zero unless that plan sets one.                                                                                          |
|                                                         | `MRR__c` / `ARR__c`                                                             | Always `0` for `Usage` / `Overage` lines.                                                                                                                                                         |
|                                                         | `ChargeType__c`                                                                 | Denormalized copy of `RevenueNature__c` the cart LWC reads for badges (`Usage` / `Overage` / `Recurring` / `OneTime`).                                                                            |
|                                                         | `IsProrated__c` / `ProratedDays__c`                                             | Stamped when `BoundaryScale` proration fires on this line (M-4).                                                                                                                                  |
| Order Item Rate Plan (`OrderItemRatePlan__c`, junction) | `OrderItem__c` / `RatePlan__c` / `QuoteLineItem__c`                             | Written once per OrderItem by `OrderItemRatePlanLinkTrigger` after native sync creates the Order — see "Consumption Schedule parity" above.                                                       |

## Build it (tester)

Do this on your own products — do not touch CRM Suite Pro, Sales Cloud or
the golden-quote products. Two scenarios: a standalone tiered Usage plan,
and the same product wired into a bundle option with a Minimum Commit +
Overage Rate.

### Scenario 1 — tiered Usage plan, no commit

1. **Products tab → New.** Name it `UP Tiered Widget`. **Active** ticked.
   Save. Related tab → Price Books → **Add Standard Price** → any
   placeholder (e.g. `999`) → Active → Save.
2. **Rate Plan Editor tab** → open `UP Tiered Widget` → **Blank plan**.
3. **Shape**: Charge type **Usage**, Pricing model **Tiered (Graduated)**,
   Billing schedule **Monthly**.
4. **Tiers** tab → **+ Add tier** three times:
   - From `1` To `10`, Rate `5`
   - From `11` To `50`, Rate `4`
   - From `51` To (blank — open-ended), Rate `3`
5. **Preview** → quantity `30` → per-unit **$4.333333** (10×$5 + 20×$4 = $130
   ÷ 30). Set **Status** to **Active** → **Save**.
6. Build a quote on an Account/Opportunity of your own (e.g. `UP Test Co`)
   and **Add products** → `UP Tiered Widget`, quantity **30**. Read the net
   price against "Expected numbers" below. Note the line shows **no**
   NMR/MRR/ARR metric row — only the "variable · usage" ARR caption,
   because there is no `CommittedUsageMRR__c` floor on this plan.

### Scenario 2 — a standalone Overage plan

1. **Products tab → New** → `UP Overage Widget`, Active, placeholder price.
2. **Rate Plan Editor** → **Shape**: Charge type **Overage**, Pricing model
   **Per Unit**, Billing schedule **Monthly**, price **0.05** → **Active** →
   **Save**.
3. **Add products** on the same test quote → `UP Overage Widget`, quantity
   **20**. Net price **$0.05**, line total **$1.00**, MRR/ARR **$0**.

### Scenario 3 — Minimum Commit + Overage Rate inside a bundle

This needs `UP Tiered Widget` (Scenario 1) as a **bundle option**, because
`MinimumCommit__c` / `OverageRate__c` only exist on `ProductOption__c`.

1. **Products tab → New** → `UP Commit Bundle`, Active, Standard Price `0`.
2. **Bundle Builder tab** → use the existing `UP Commit Bundle` product.
   Pricing: **Options carry the price**. **+ Add feature** → `Core`.
   **+ Add option** → `UP Tiered Widget`, **Charged separately**, on by
   default, required.
3. Open the option's detail and set **Minimum Commit** `10` and
   **Overage Rate** `1` (these two fields sit on the Product Option, not
   the Rate Plan — if your Bundle Builder build doesn't surface them yet,
   ask Claude to check with SOQL rather than hunting for a UI field that
   may not be placed on this layout).
4. **Create bundle** → add `UP Commit Bundle` to the test quote at quantity
   **1**; its option `UP Tiered Widget` comes with it.
5. Set the option's quantity to **30** → net price **$5.00** ($150 = $130
   tier math + $20 overage on the 20 units past the 10-unit commit).
6. Change the option's quantity to **5** (below the commit) → net price
   jumps to **$10.00** — billed as if 10 were bought, not 5.

## Check it (Claude)

```bash
sf data query -o cpq-pkg -q "SELECT DDCPQ__TargetProduct__r.Name, DDCPQ__Status__c, DDCPQ__RevenueNature__c, DDCPQ__PricingModel__c, DDCPQ__UnitPrice__c, DDCPQ__UsageTierMatrix__c FROM DDCPQ__RatePlan__c WHERE DDCPQ__TargetProduct__r.Name LIKE 'UP %'"

sf data query -o cpq-pkg -q "SELECT DDCPQ__TierIndex__c, DDCPQ__QtyMin__c, DDCPQ__QtyMax__c, DDCPQ__RatePerUnit__c FROM DDCPQ__UsageTier__c WHERE DDCPQ__TargetProduct__r.Name = 'UP Tiered Widget' ORDER BY DDCPQ__TierIndex__c"

sf data query -o cpq-pkg -q "SELECT DDCPQ__ChildProduct__r.Name, DDCPQ__MinimumCommit__c, DDCPQ__OverageRate__c FROM DDCPQ__ProductOption__c WHERE DDCPQ__ParentProduct__r.Name = 'UP Commit Bundle'"

sf data query -o cpq-pkg -q "SELECT Product2.Name, Quantity, DDCPQ__NetPrice__c, DDCPQ__UsageTierPrice__c, DDCPQ__MRR__c, DDCPQ__ARR__c, DDCPQ__ChargeType__c FROM QuoteLineItem WHERE Quote.Name = 'UP Test Quote'"
```

Expect: both Rate Plans `Active`; 3 tier rows on `UP Tiered Widget` in
ascending `TierIndex__c`; the Product Option's commit/overage fields `10`
and `1`; every `UP %` QLI's `MRR__c` and `ARR__c` reading `0`.

**Read-only price preview** (no commit) — the fastest way to see the tier
breakdown without opening the waterfall drawer three clicks deep. Query the
quote and product Ids first, write the body to a file:

```json
{
  "quoteId": "<0Q0…>",
  "selections": [
    {
      "localKey": "t1",
      "productId": "<UP Tiered Widget 01t…>",
      "quantity": 30,
      "sourceBundleId": "<UP Tiered Widget 01t…>"
    }
  ]
}
```

```bash
sf api request rest "services/apexrest/DDCPQ/dd/v1/price?includeWaterfall=true" -o cpq-pkg --method POST --body @body.json
```

Look at the `UsageTierPrice` stage in `lines[0].waterfall` — its `subRows`
array names each bracket consumed (`"Tier 1 (1–10): 10 × $5.0000"`, etc.)
and its `note` reads `"Graduated: billableQty=<N>"`, or
`"...+overage(<qty>×<rate>=$<amount>)"` when a Minimum Commit/Overage Rate
pair is in play. For the bundle scenario, add a second selection with
`parentLocalKey` pointing at the bundle parent's `localKey` and
`productOptionId` set to the option's Id — the exact `Selection` fields
`CpqEngine.Selection` accepts are documented in golden-path.md.

## Expected numbers

All rows below are from real `sf api request rest` runs against cpq-pkg,
cross-checked against the committed QuoteLineItems.

| Scenario                                                                           | Input                 | Result                                                                                                                                           |
| ---------------------------------------------------------------------------------- | --------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------ |
| Tiered Usage, no commit, qty 5 (bracket 1 only)                                    | 1–10 @ $5             | netPrice **$5.00**, totalNet **$25**                                                                                                             |
| Tiered Usage, no commit, qty 30 (crosses 2 brackets)                               | 1–10 @ $5, 11–50 @ $4 | netPrice **$4.333333**, totalNet **$130**, `MRR__c`/`ARR__c` **0**                                                                               |
| Overage plan, PerUnit $0.05, qty 20                                                | standalone, no tiers  | netPrice **$0.05**, totalNet **$1.00**, `MRR__c`/`ARR__c` **0**                                                                                  |
| Same tiered product as a bundle option, Minimum Commit 10 / Overage Rate 1, qty 30 | over the allowance    | `UsageTierPrice` note "Graduated: billableQty=30, +overage(20×1.0000=$20.0000)"; netPrice **$5.00**; totalNet **$150** ($130 tier + $20 overage) |
| Same option, qty 5                                                                 | under the allowance   | `UsageTierPrice` note "Graduated: billableQty=10"; netPrice **$10.00** (billed as 10, not 5)                                                     |
| Bundle parent line itself                                                          | roll-up header        | `UsageTierPrice` stage applicability not-applicable, note "Bundle parent skips usage tiers by design"; netPrice **$0**                           |

## Troubleshooting

| Symptom                                                                             | Cause                                                                                                                                                                 | Fix                                                                                                                                   |
| ----------------------------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------- |
| Usage/Overage line shows no MRR, no ARR, nothing in the metric row                  | By design — `RevenueNature__c != 'Recurring'` always yields `MRR__c = ARR__c = 0`                                                                                     | Not a bug. Look for the "variable · usage" or "$X ARR · projected/committed" caption under the line instead of a metric row           |
| `UsageTierPrice` stage reads not-applicable and the note says no usage tiers active | No Active `UsageTierMatrix__c` has `UsageTier__c` rows pointed at this `TargetProduct__c`                                                                             | Add at least one tier row targeting the exact product, on a matrix with `Status__c = Active`                                          |
| Net price ignores the tiers and just mirrors the plan's flat `UnitPrice__c`         | Tiered/Volume/Package plan has zero matching tier rows — the engine silently falls back to PerUnit math rather than refusing                                          | Check the tier rows target this product (`TargetProduct__c`), not a different one, and that their matrix is Active                    |
| Rate Plan Editor refuses to save a Tiered plan as Active                            | Checks panel finding `TIER-1`: "A Tiered plan prices from its tiers, and it has none."                                                                                | Add at least one `UsageTier__c` row before setting Status to Active                                                                   |
| Minimum Commit / Overage Rate on the Product Option have no effect on the price     | The product's own `UsageTier__c` rows don't exist or aren't Active — `UsageTierService` skips the line (and both fields) before it ever reads the commit/overage pair | Confirm the SAME product has Active tiers; Minimum Commit/Overage Rate are modifiers on top of tier math, not a standalone mechanism  |
| Below-commit quantity bills for more than was typed                                 | Correct — `billableQty = MAX(actualQty, MinimumCommit__c)`. A quantity of 5 against a 10-unit commit bills as 10                                                      | Not a bug; this is the included-allowance contract. Lower the Minimum Commit if that's not the intent                                 |
| Tier brackets are nowhere to be found on the cart line                              | By design in this build — click the line's strategy icon to open the waterfall drawer, then expand the `UsageTierPrice` stage's `subRows`                             | There is no cart-level bracket viewer; use the REST price-preview's `subRows` array instead of hunting through three clicks in the UI |
| `AXIS-5` error on a plan with `BillingSchedule = UpfrontForTerm`                    | That picklist value means a prepay-of-Recurring pattern, not Usage/Overage — `MinCommitMonths__c` is required and missing                                             | See trials-and-payment-terms.md / pricing-models.md; `UpfrontForTerm` is not the pairing used for metered billing                     |
| `FLOOR_BLOCKED` on commit for a Usage line                                          | Unrelated feature — Margin Floor, a different guardrail on discounts                                                                                                  | See margin-and-manual-discounts.md; nothing to do with tiers or commits                                                               |
| A Usage line's `CommittedUsageMRR__c` stays 0 even though the plan looks floored    | Confusing `RatePlan.MinimumCommitAmount__c` (a dollar floor) with `ProductOption.MinimumCommit__c` (a quantity floor) — only the former feeds `CommittedUsageMRR__c`  | Set `MinimumCommitAmount__c` + `MinCommitMonths__c` on the Rate Plan itself; see prepaid-and-commitments.md                           |

## Limits and gotchas

- **MRR/ARR are always zero for Usage and Overage** — this is intentional
  across the whole engine, not a per-feature setting. The only exception is
  the separate `CommittedUsageMRR__c` field, which is a dollar floor, not a
  recurring-revenue number.
- **`ProductOption.MinimumCommit__c` / `OverageRate__c` require a bundle.**
  There is no way to apply an included-allowance-plus-overage shape to a
  standalone (non-bundle) product in this build — those two fields simply
  don't exist outside `ProductOption__c`. A standalone Overage plan on its
  own product (Scenario 2) is the pattern for that instead, with no
  automatic linkage to a base allowance — the rep (or a naming convention)
  has to connect the two lines mentally.
- **Package-model plans never run through the commit/overage path.**
  `UsageTierService` explicitly skips any draft whose pricingModel is
  Package — running Package tiers through this service's Volume/Graduated
  math double-counts (multiplies then divides by quantity against the
  wrong number). Package lines fall back to `PricingModelService`'s own
  branch, get no Minimum Commit/Overage Rate, no ramp modulation, no
  first-period proration, and no subRows in the waterfall.
- **No forecast tool, no bracket chart, no burndown viewer in the cart.**
  Everything a rep needs to reason about "what does this cost at 2x usage"
  is a mental exercise or a change-reprice-change-back cycle. This matches
  `DD_CPQ_2027_USAGE_QUOTING_UX_ANALYSIS.html`'s "Usage Quoting Workbench"
  proposal, which is analysis only — none of its 10 named features (a
  forecast slider, a bracket visualizer, an inline drawdown simulator, a
  base+overage grouped line, a 12-month usage timeline, etc.) exist as code
  in this build. Don't describe any of them as available.
- **The AI token pricing trail is also analysis, with one real seed script.**
  `DD_CPQ_2027_AI_TOKEN_PRICING_ANALYSIS.html` claims 11 of 14 modern AI
  pricing patterns (input/output tokens per 1M, cache write/read, batch
  discount, volume tiers, reseller markup, committed provisioned throughput,
  fine-tuned surcharge) map cleanly onto the 5-axis Rate Plan model used in
  this module and in pricing-models.md — all of those are ordinary Usage
  PerUnit or Tiered plans, nothing new. Two patterns are called Partial
  (free promo credits that flip to metered; multi-region rate variants) and
  one is a real gap (attribute-based long-context surcharge, e.g. Gemini's
  2x rate past 128K tokens — there is no field that conditions a rate on a
  request attribute like context length). The script
  `seed-anthropic-cpq-catalog.apex` the trail cites is a real, idempotent
  seed of 15 Claude API products exercising all five pricing models — it is
  not installed-package behavior, just a demo data script in the repo.
  Treat every number and gap claim in that doc as design analysis, not a
  tested feature, until it's been verified the way this module's numbers
  were verified.
- **Consumption-schedule Segments (M-2) only engage above 12 months.** A
  `RampSchedule__c` with a `UsageTierMatrixOverride__c` is silently ignored
  for any line whose `termMonths <= 12` — the single-matrix path always
  wins for a one-year term, by design.
- **Category-based rating (M-3) requires every relevant tier to share a
  `Matrix__c`.** Groups are evaluated as if they were one Consumption
  Schedule; mixing categories across two different matrices on the same
  product isn't a supported shape.
- **Governor limits.** `UsageTierService.applyAll` is a single bulk pass:
  one SOQL for all matching tiers (filtered by product), one for any
  ProductOption commit/overage rows, one for ramp-override probing per
  product, one for proration-mode lookups per referenced Rate Plan.
  DDCPQ-39's signature cache means two lines with identical tiers, quantity,
  commit and overage reuse one computed result instead of recomputing —
  safe at any cart size seen so far.

## Questions testers ask

**"Why does my Usage line show $0 ARR on the Quote total?"** — Not a bug.
Usage and Overage never contribute to MRR/ARR; only Recurring plans do. Look
for the line's own "variable · usage" caption for a sense of its size.

**"Where do I see the tier brackets without clicking three times?"** — There
isn't a cart-side bracket view in this build. Ask Claude to run the
read-only price preview and read the `UsageTierPrice` stage's `subRows` —
that's the fastest route to the same numbers the waterfall drawer shows.

**"I set Minimum Commit on my product but nothing changed."** — Check two
things: the field only exists on a bundle's Product Option (not on a
standalone product or its Rate Plan), and the product needs Active
`UsageTier__c` rows for the commit/overage math to run at all.

**"Why did a quantity of 5 bill as if I bought 10?"** — That's the Minimum
Commit floor working as designed: `billableQty = MAX(actualQty,
MinimumCommit__c)`. The customer is paying for the allowance they committed
to, not the smaller amount they actually used this period.

**"Is Overage the same thing as the margin floor / commitment floor?"** —
No. `RevenueNature__c = 'Overage'` is a charge-type axis value on a Rate
Plan (this module). The Margin Floor and the account-level commitment floor
are unrelated features — see margin-and-manual-discounts.md and
prepaid-and-commitments.md.

**"Can I model a prepaid pool that drains as usage comes in?"** — That's
prepaid credit, not this module. See prepaid-and-commitments.md — it's a
what-if simulator over REST/MCP, not a live cart balance either.

**"Does the Usage Quoting Workbench from the design doc exist yet?"** — No.
That document is a design proposal; none of its 10 named rep-facing features
(forecast slider, bracket visualizer, drawdown projector, base+overage
grouping, usage timeline, etc.) are built in 0.1.0-9.

**"What's the difference between RatePlan.MinimumCommitAmount__c and
ProductOption.MinimumCommit__c?"** — The first is a dollar floor on one
plan's MRR (`CommittedUsageMRR__c`), documented in
prepaid-and-commitments.md. The second is a quantity floor consumed right
here by `UsageTierService` to compute `billableQty`. They never interact
with each other.
