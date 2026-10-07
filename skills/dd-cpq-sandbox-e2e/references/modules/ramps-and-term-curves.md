# Ramps and term discount curves

> Installed build 0.1.0-9 · Verified 2026-10-07 in org **cpq-pkg** (0.1.0-9
> installed): a 36-month ramp of +10%/+10% on a $100 product gave three
> QuoteLineItems — Year 1 **$100**, Year 2 **$110**, Year 3 **$121** — and a
> step Term Discount Curve (12mo=0%, 24mo=10%, 36mo=15%) priced the same
> $100 product at **$90** for a 24-month term and **$85** for a 36-month
> term, both via REST `price`/`commit`.

## What it is for

Two different ways to price a multi-year deal, and they can both apply to
the same line:

1. **Ramps** (`RampMatrix__c` / `RampSchedule__c`, DDCPQ-59) — the price
   itself steps up (or down, or resets) year over year, e.g. "$100/seat in
   Year 1, +10% in Year 2, +10% in Year 3." Since DDCPQ-59, a ramped
   Recurring line over 12 months becomes **one QuoteLineItem per year**,
   not one line with an averaged rate — each year has its own start date,
   term, quantity, discounts, volume tier and margin floor.
2. **Term discount curves** (`TermDiscountCurve__c` /
   `TermDiscountPoint__c`, DDCPQ-51..58) — a single discount percentage
   determined by how long the whole commitment is, e.g. "0% at 12 months,
   10% at 24, 15% at 36." One line, one price, no splitting. **This object
   is deprecated** (2026-09-11 waterfall gap review) — the engine still
   reads it and Rules Studio still edits it, but the recommended way to
   price by commitment length going forward is a separate Rate Plan per
   term (Monthly / Annual / 3-Year, each with its own `UnitPrice__c`), not
   a curve. Tell testers this plainly if they ask whether to build new
   pricing on curves.

Both run in the same waterfall, back to back: Ramp, then Term Curve, then
Usage Tier. A line can have a ramp AND a term curve — the curve's discount
applies to whatever price the ramp produced for that year (see "How it
works").

## How it works

### Ramps — the year-by-year split (DDCPQ-59)

- A **Ramp Rule Set** (`RampMatrix__c`) is a named header you switch
  `Active`/`Draft`/`Inactive`. Only Active sets are read.
- A **Ramp Schedule** (`RampSchedule__c`) is one year's step, under a set:
  - `PeriodIndex__c` — 0-indexed. **0 = Year 1** (never has a schedule row;
    Year 1 is always the plan's own list price). **1 = Year 2, 2 = Year 3**,
    and so on.
  - `AdjustmentType__c` — `PctIncrease` (multiply the prior year's rate by
    `1 + value/100`), `PctDecrease` (`1 - value/100`), or `AbsoluteReset`
    (replace the prior rate with `value` outright).
  - `AdjustmentValue__c` — the percent or the absolute price, depending on
    type.
  - `TargetProduct__c` — blank matches every product; a product narrows it.
  - `TargetQuote__c` — optional, scopes the whole schedule to one quote.
  - The shared 5-field criteria (`FieldApiName__c` / `Operator__c` /
    `Value__c` / `ValueType__c` + `Priority__c`) can further gate a step,
    e.g. "only in Year 2 if Account.Industry = Healthcare." Empty = always.
  - `Currency__c` — blank is a catch-all; set it to scope one step to one
    currency on a multi-currency org.
  - A missing period (no schedule row at that index) **carries the prior
    year's rate forward flat** — it is not a no-op for the whole ramp.
- **Who gets segmented.** A Recurring line qualifies for the one-QLI-per-
  year split only if: its committed term is **more than 12 months**, at
  least one Active schedule at `PeriodIndex ≥ 1` matches it, product +
  criteria + currency all match, and the line is a **new-sale** line — not
  a bundle parent, not a OneTime or Usage charge, not a hybrid charge-
  profile sibling, and not a line that changes an existing Asset
  (amendment/renewal). Those excluded cases keep the older, single-line,
  **term-weighted-average** ramp instead (see "The two ramp math paths"
  below) — or no ramp at all if they also skip that path.
- **The split itself.** Year 1 keeps the line's own QuoteLineItem Id. Year
  2 onward are brand-new lines, `12` months each by default (the last year
  absorbs whatever the term doesn't divide evenly, e.g. a 30-month term
  gives Years of 12/12/6). Every year after Year 1 carries
  `SegmentRoot__c` = Year 1's QLI Id, and all years share one
  `SegmentGroupKey__c` (a GUID) so billing/reporting can group them.
  `SegmentIndex__c` is the year number, 0-based.
- **Rates.** Year 1 is the plan's `DerivedListPrice__c`, untouched. Year 2's
  rate = Year 1's rate adjusted by Year 2's schedule (or carried flat if
  none matches); Year 3 compounds off Year 2; and so on — exactly the
  product's own waterfall shown per year, not a blended average.
  `RampAdjustedPrice__c` is set on Year 2+ (the adjusted rate) but stays
  **null on Year 1**, because Year 1 is never itself "adjusted."
- **Manual edits stick.** A rep can open a year in the ramp editor and
  type their own quantity, price and/or discount for just that year. That
  year's `SegmentSource__c` flips to `Manual`; the rule stops touching its
  price (but keeps laying out the _shape_ — months, start date — unless
  the rep also edits those). A later reprice keeps the manual year's price
  and only recomputes the years still on `Rule`. "Reset to rule" in the
  editor puts a year back under rule control.
- **Contract length changes survive.** If a rep's manually-typed years
  don't add up to the commitment term any more (e.g. the term grew to 42
  months), DD CPQ lengthens the last year to absorb the gap, or trims/drops
  years past the new end, and says so as a warning
  (`RAMP_YEARS_ADJUSTED` / `RAMP_YEAR_INVALID`) rather than failing commit.

### The two ramp math paths — don't confuse them

|                                                     | One-QLI-per-year (DDCPQ-59, the normal case)                     | Term-weighted average (older, still live)                                       |
| --------------------------------------------------- | ---------------------------------------------------------------- | ------------------------------------------------------------------------------- |
| Applies to                                          | New-sale Recurring lines, term > 12mo                            | Amendment/renewal lines, hybrid charge siblings, OneTime/Usage-ineligible lines |
| Lines produced                                      | One QLI per year                                                 | One QLI, one price                                                              |
| `RampAdjustedPrice__c`                              | Set on Year 2+ only; null on Year 1                              | The single blended rate, or null if no schedule matched                         |
| `RampSchedule__c` (the QLI text field, see Gotchas) | **Stays blank** — nothing writes it on this path                 | `{"rates":[...],"months":[...]}` — the per-period shape behind the average      |
| Waterfall stage shown                               | `RampAdjustedPrice` per year, e.g. "Year 3 of 3: +10% on Year 2" | `RampAdjustedPrice`, one weighted-average note                                  |

### Term discount curves (TermDiscountService)

- A **Term Discount Curve** (`TermDiscountCurve__c`) is a named, versioned
  header: `Status__c` (Draft/Active/Inactive — only Active is read),
  `InterpolationMode__c` (`step` or `linear`), `Priority__c` (tie-break
  between curves that cover the same product equally — lower wins).
- A **Term Discount Point** (`TermDiscountPoint__c`) is one breakpoint:
  `TermMonths__c`, `DiscountPct__c`, optional `TargetProduct__c` (blank =
  every product), `BillingFrequency__c` (blank = any billing frequency;
  set it to scope a point to e.g. Annual-only pricing), and the shared
  5-field criteria.
- **Step mode**: the engine finds the largest point whose `TermMonths__c`
  is **at or below** the line's term and uses its `DiscountPct__c` flat —
  a staircase. A 30-month deal against points at 24mo/10% and 36mo/15%
  gets 10%, not something in between.
- **Linear mode**: interpolates a weighted value between the two
  surrounding points. A 30-month deal between 24mo/10% and 36mo/15% gets
  `10 + (15-10) × (30-24)/(36-24) = 12.5%`.
- **Below the smallest point, the curve is a no-op** — a 6-month term
  against a curve starting at 12mo gets no discount at all (not 0%
  explicitly — the stage reads "passthrough").
- **Base price**: the curve discounts `rampAdjustedPrice` if the line has
  one, else `derivedListPrice` — so a ramp and a curve compose: the curve
  takes a cut of whatever the ramp already produced for that line/year.
- **Choosing between curves**: when more than one Active curve covers the
  same product, the curve whose point **names the product** beats one that
  only covers it by criteria or a group; then the curve's own `Priority__c`
  (lower wins), then the point's `Priority__c`, then the curve name.
- **Skipped points are reported, not silent.** A point whose `TermMonths__c`
  the line already reached, but whose criteria failed, is listed by name in
  the waterfall note instead of the curve quietly falling back to a lower
  tier — e.g. "The 24-month point (10% off) ... was skipped: Account.Industry
  equals Healthcare, but this quote has Technology."
- **Term discount curves do not split lines.** Unlike a ramp, a curve
  always produces one line with one `TermAdjustedPrice__c` — there is no
  per-year behaviour here even on a 5-year deal.

### Term months and dates driving both

- `QuoteLineItem.TermMonths__c` is the line's own commitment length; blank
  means "follow the deal's term." `RatePlan__c.MinCommitMonths__c` can
  raise the effective commitment above what was typed (see
  `CommitService.commitmentTerm`).
- Ramp years are laid out from `EffectiveDate__c` forward in 12-month
  blocks, each year's `EndDate__c` one day before the next year's start —
  this is the same co-term machinery the rest of the quote uses (see
  `contracts-and-acceptance.md`), just invoked once per ramp year instead
  of once per line.
- **Reprice to Today reads the _stored_ waterfall, not a live one.** What a
  rep sees on a quote is each line's `WaterfallJson__c` as of its last
  commit. If a ramp rule or a term curve point changes after that, the
  quote keeps showing the old numbers until someone runs **Reprice to
  Today** — which recommits the quote through the normal cart path and
  diffs the old stored waterfall against the new one, stage by stage,
  naming which rule moved it. There is no background job that silently
  reprices a ramp year as its start date arrives; a stale Year 2 only
  updates when the quote is reopened/recommitted or Reprice to Today runs.

## Objects and fields

**Ramp Rule Set** (`RampMatrix__c`)

| Field       | Meaning / values                                                             |
| ----------- | ---------------------------------------------------------------------------- |
| `Status__c` | `Draft` / `Active` / `Inactive` — only Active is used when quotes are priced |

**Ramp Schedule** (`RampSchedule__c`, autonumber `RS-{00000}`, child of a Ramp Rule Set)

| Field                                                                           | Meaning / values                                                                                                                       |
| ------------------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------- |
| `Matrix__c`                                                                     | The Ramp Rule Set this step belongs to                                                                                                 |
| `PeriodIndex__c`                                                                | 0 = Year 1 (never has a row), 1 = Year 2, 2 = Year 3, …                                                                                |
| `AdjustmentType__c`                                                             | `PctIncrease` / `PctDecrease` / `AbsoluteReset`                                                                                        |
| `AdjustmentValue__c`                                                            | The percent (Pct*) or the new flat price (AbsoluteReset)                                                                               |
| `TargetProduct__c`                                                              | Blank = every product                                                                                                                  |
| `TargetQuote__c`                                                                | Blank = every quote; set to scope one schedule to one quote                                                                            |
| `Currency__c`                                                                   | Blank = catch-all; set to scope a step to one currency                                                                                 |
| `UsageTierMatrixOverride__c`                                                    | Optional — swaps in a different Usage Tier matrix for just this period (consumption/Segments parity); unrelated to the rate math above |
| `FieldApiName__c` / `Operator__c` / `Value__c` / `ValueType__c` / `Priority__c` | Shared 5-field criteria; empty = catch-all                                                                                             |

**Term Discount Curve** (`TermDiscountCurve__c`) — deprecated, still live

| Field                  | Meaning / values                                                                                        |
| ---------------------- | ------------------------------------------------------------------------------------------------------- |
| `Status__c`            | `Draft` / `Active` / `Inactive`                                                                         |
| `InterpolationMode__c` | `step` (staircase to the last point reached) / `linear` (interpolate between the two bracketing points) |
| `Priority__c`          | Tie-break vs. another curve covering the same product equally; lower wins; blank loses to any number    |

**Term Discount Point** (`TermDiscountPoint__c`, autonumber `TDP-{00000}`, child of a curve)

| Field                                                                           | Meaning / values                                                |
| ------------------------------------------------------------------------------- | --------------------------------------------------------------- |
| `Curve__c`                                                                      | The curve this breakpoint belongs to                            |
| `TermMonths__c`                                                                 | The contract length this point applies to, e.g. 12/24/36/48/60  |
| `DiscountPct__c`                                                                | % off at (step) or at exactly (linear anchor) this term         |
| `TargetProduct__c`                                                              | Blank = every product                                           |
| `BillingFrequency__c`                                                           | Blank = any billing frequency; set to scope to e.g. Annual-only |
| `FieldApiName__c` / `Operator__c` / `Value__c` / `ValueType__c` / `Priority__c` | Shared 5-field criteria                                         |

**Quote Line Item** — ramp and term fields

| Field                  | Meaning / values                                                                                                                                                                                                                                                      |
| ---------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `TermMonths__c`        | This line's own commitment length; blank follows the deal's term                                                                                                                                                                                                      |
| `RampAdjustedPrice__c` | The adjusted per-unit rate for Year 2+ of a segmented ramp, or the blended average on the older single-line path; **null on Year 1** and on any unramped line                                                                                                         |
| `RampSchedule__c`      | **Confusingly named** — this is a _field_, not the `RampSchedule__c` _object_. Long text JSON `{"rates":[...],"months":[...]}`. Only written on the older term-weighted-average path; **stays blank on every line produced by the one-QLI-per-year split** (DDCPQ-59) |
| `TermAdjustedPrice__c` | Per-unit price after the Term Discount Curve stage; null if no curve/point matched                                                                                                                                                                                    |
| `TermDiscountPct__c`   | The % that `TermAdjustedPrice__c` applied; null if nothing matched                                                                                                                                                                                                    |
| `SegmentGroupKey__c`   | GUID shared by every year of one ramped line; blank if not ramped. External Id, so billing adapters can upsert by it                                                                                                                                                  |
| `SegmentIndex__c`      | 0 = Year 1 (the root), 1 = Year 2, … ; blank if not ramped. Filter `SegmentIndex__c = 0 OR SegmentIndex__c = null` to count a ramped line's ARR once                                                                                                                  |
| `SegmentRoot__c`       | On Year 2+, a lookup back to the Year 1 QuoteLineItem. Blank on Year 1                                                                                                                                                                                                |
| `SegmentSource__c`     | `Rule` (the ramp rule still sets this year's price) or `Manual` (a rep edited it; the rule now only shapes other years)                                                                                                                                               |
| `SegmentBasePrice__c`  | On a `Manual` year, the rep's own per-unit price before discounts; blank on a `Rule` year                                                                                                                                                                             |

**Rate Plan** (`RatePlan__c`) — the dead link

| Field                  | Meaning / values                                                                                                                                                                       |
| ---------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `TermDiscountCurve__c` | **Never read by the engine.** `TermDiscountService` walks every Active curve by criteria + precedence, not this lookup. Setting it does nothing; kept for backwards compatibility only |

## Build it (tester)

Two scenarios, matching the verified numbers above.

### Scenario A — a 36-month ramp

1. **Product** `RT Ramp Product`, Active, Standard Price **$100**. Give it a
   Rate Plan: Recurring / Per unit / Monthly / **$100**, Active.
2. **Ramp Rule Sets tab → New**: Name `RT Ramp Set`, Status **Active** →
   Save.
3. On that record, related list **Ramp Schedules → New** (twice):
   - Period Index **1**, Adjustment Type **Percent Increase**, Adjustment
     Value **10**, Applies To Product `RT Ramp Product` → Save.
   - Period Index **2**, Adjustment Type **Percent Increase**, Adjustment
     Value **10**, Applies To Product `RT Ramp Product` → Save.
4. **Account** `RT Test Co` → **Opportunity** `RT Ramp Opp` → **New Quote**
   `RT Ramp Quote` → **Configure Products**.
5. **Add products** → `RT Ramp Product`, quantity **10**, **Term (Months)
   36** (set it on the line, or on the quote header if every line follows
   the deal's term).
6. The cart shows **three rows** for this one product add: Year 1 at
   $100, Year 2 at $110, Year 3 at $121 — look for the ramp icon
   ("Edit multi-year ramp") on the line to open the ramp editor and see
   all three years, their quantities and the rule driving each one.
7. **Commit.** Three QuoteLineItems are written, sharing one
   `SegmentGroupKey__c`.

To see a manual override: open the ramp editor on Year 2, type a different
discount % or price, **Apply changes**. That year's `SegmentSource__c`
becomes `Manual`; Year 3 (still `Rule`) keeps compounding off whatever
Year 2's rule rate _would have been_, not off the rep's typed price —
re-open the editor after saving to confirm Year 3 didn't move.

### Scenario B — a term discount curve at two terms

1. **Product** `RT Term Product`, Active, Standard Price **$100**. Rate
   Plan: Recurring / Per unit / Monthly / **$100**, Active.
2. **Rules Studio tab → Term Discount Curves → New curve**: Name
   `RT Term Curve`, Status **Active**, Interpolation Mode **Step**.
3. Add three points: **12** months / **0**%, **24** months / **10**%,
   **36** months / **15**% → scope all three to `RT Term Product` → Save.
   (The same curve and points can be built on the plain **Term Discount
   Curve** tab + its Term Discount Points related list if Rules Studio
   isn't available yet — same object, same result.)
4. **Account** `RT Test Co` → **Opportunity** `RT Term Opp` → **New Quote**
   `RT Term Quote #1` → **Configure Products** → add `RT Term Product`,
   quantity **5**, Term (Months) **24**. Net price reads **$90** (10% off).
   **Commit.**
5. A second quote, same product/quantity, Term (Months) **36**. Net price
   reads **$85** (15% off). **Commit.**

To see linear mode: flip Interpolation Mode to **Linear** and quote the
same product at **30** months — expect **12.5%** off ($87.50), halfway
between the 24mo and 36mo points.

## Check it (Claude)

```bash
sf data query -o cpq-pkg -q "SELECT Name, DDCPQ__Status__c FROM DDCPQ__RampMatrix__c WHERE Name = 'RT Ramp Set'"

sf data query -o cpq-pkg -q "SELECT DDCPQ__Matrix__r.Name, DDCPQ__PeriodIndex__c, DDCPQ__AdjustmentType__c, DDCPQ__AdjustmentValue__c, DDCPQ__TargetProduct__r.Name FROM DDCPQ__RampSchedule__c WHERE DDCPQ__Matrix__r.Name = 'RT Ramp Set' ORDER BY DDCPQ__PeriodIndex__c"

sf data query -o cpq-pkg -q "SELECT Id, Product2.Name, Quantity, UnitPrice, DDCPQ__TermMonths__c, DDCPQ__DerivedListPrice__c, DDCPQ__RampAdjustedPrice__c, DDCPQ__NetPrice__c, DDCPQ__SegmentGroupKey__c, DDCPQ__SegmentIndex__c, DDCPQ__SegmentRoot__c, DDCPQ__SegmentSource__c FROM QuoteLineItem WHERE Quote.Name = 'RT Ramp Quote' ORDER BY DDCPQ__SegmentIndex__c"

sf data query -o cpq-pkg -q "SELECT Name, DDCPQ__Status__c, DDCPQ__InterpolationMode__c FROM DDCPQ__TermDiscountCurve__c WHERE Name = 'RT Term Curve'"

sf data query -o cpq-pkg -q "SELECT DDCPQ__Curve__r.Name, DDCPQ__TermMonths__c, DDCPQ__DiscountPct__c, DDCPQ__TargetProduct__r.Name FROM DDCPQ__TermDiscountPoint__c WHERE DDCPQ__Curve__r.Name = 'RT Term Curve' ORDER BY DDCPQ__TermMonths__c"

sf data query -o cpq-pkg -q "SELECT Id, Quantity, UnitPrice, DDCPQ__TermMonths__c, DDCPQ__DerivedListPrice__c, DDCPQ__TermAdjustedPrice__c, DDCPQ__TermDiscountPct__c, DDCPQ__NetPrice__c, DDCPQ__ARR__c FROM QuoteLineItem WHERE Quote.Name LIKE 'RT Term Quote%' ORDER BY DDCPQ__TermMonths__c"
```

All six ran clean against cpq-pkg on 2026-10-07 and returned the rows
below.

**Read-only price preview** (same pattern as the golden path — never call
`commit`, only `price`):

```json
{
  "quoteId": "<0Q0...>",
  "selections": [
    {
      "localKey": "p",
      "productId": "<RT Ramp Product 01t...>",
      "quantity": 10,
      "termMonths": 36
    }
  ]
}
```

```bash
sf api request rest "services/apexrest/DDCPQ/dd/v1/price?includeWaterfall=true" -o cpq-pkg --method POST --body @body.json
```

The response's `lines[]` will show three entries for one selection sent —
`localKey` **"p"** (Year 1), **"p#y2"**, **"p#y3"** — each with its own
`netPrice` and a `RampAdjustedPrice` waterfall stage. That is expected;
one selection fanning out to three preview lines is exactly what the
one-QLI-per-year split does before anything is written.

## Expected numbers

| Scenario                                                    | Input     | Result                                                                                                                                      |
| ----------------------------------------------------------- | --------- | ------------------------------------------------------------------------------------------------------------------------------------------- |
| 36-month ramp, +10%/+10%, qty 10 @ $100 list                | Year 1    | `RampAdjustedPrice` null, Net **$100**, `SegmentIndex` 0, `SegmentSource` Rule                                                              |
| same line                                                   | Year 2    | `RampAdjustedPrice` **110**, Net **$110**, `SegmentIndex` 1, `SegmentRoot` → Year 1's Id                                                    |
| same line                                                   | Year 3    | `RampAdjustedPrice` **121**, Net **$121**, `SegmentIndex` 2, note _"Year 3 of 3: +10% on Year 2"_                                           |
| Step curve 12mo=0% / 24mo=10% / 36mo=15%, qty 5 @ $100 list | Term 24mo | `TermAdjustedPrice` **90**, `TermDiscountPct` **10**, Net **$90**, ARR **$5,400**                                                           |
| same curve                                                  | Term 36mo | `TermAdjustedPrice` **85**, `TermDiscountPct` **15**, Net **$85**, ARR **$5,100**                                                           |
| `GET /dd/v1/ramp/overrides?productId=<RT Ramp Product>`     | —         | 2 rows, Period Index 1 and 2, both noting _"Period uses plan base matrix (rate-only ramp)"_ (no `UsageTierMatrixOverride__c` set on either) |

## Troubleshooting

| Symptom                                                                                   | Cause                                                                                                                                                                                               | Fix                                                                                                                                  |
| ----------------------------------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------ |
| Added a 36-month line and got **one** line, not three                                     | Term is ≤ 12 months, no Active Ramp Rule Set matches the product, or the line is a bundle parent/OneTime/Usage/amendment/renewal — all of which never segment                                       | Check the line's own Term (Months); confirm the Ramp Rule Set and its Period 1 schedule are both Active and target the right product |
| Years priced but didn't step — every year shows the same price                            | No schedule row at that `PeriodIndex`, so the engine carries the prior year's rate flat (by design, not a bug)                                                                                      | Add the missing Period Index row, or confirm `AdjustmentValue__c` isn't 0                                                            |
| Year 3's price looks wrong after I manually edited Year 2                                 | Compounding math is unaffected by a manual edit — Year 3 still compounds off what Year 2's **rule** would have produced, not off the rep's typed price                                              | Expected behaviour; to change Year 3 too, edit it directly or reset Year 2 to the rule                                               |
| A ramp line committed, but querying `RampSchedule__c` (the field) comes back blank        | Expected on every one-QLI-per-year line — that JSON field is only written by the older term-weighted-average path                                                                                   | Read `RampAdjustedPrice__c` per year instead; don't expect the JSON shape on segmented lines                                         |
| `TermAdjustedPrice__c` is null even though a curve covers the term                        | The matching point's criteria failed, or the curve is Draft/Inactive, or `BillingFrequency__c` on the point doesn't match the line's billing frequency                                              | Open the waterfall note — a skipped point names exactly which condition failed                                                       |
| Changing a Ramp Schedule's `AdjustmentValue__c` didn't change an already-committed quote  | Stored waterfalls don't repoll — see "Reprice to Today reads the _stored_ waterfall" above                                                                                                          | Reopen Configure Products (which recommits) or run Reprice to Today                                                                  |
| Two curves both seem to cover the product — unsure which one fired                        | `matchedCurveId` in the preview response, or the waterfall's `sourceRuleId`, names the winning curve; precedence is product-specific point > curve `Priority__c` > point `Priority__c` > curve name | Lower the intended curve's Priority, or scope the other curve off this product                                                       |
| Setting `RatePlan__c.TermDiscountCurve__c` didn't change pricing                          | That lookup is never read by the engine (see Objects and fields)                                                                                                                                    | Don't rely on it; author the curve's own points with `TargetProduct__c` instead                                                      |
| Ramp years don't line up with the quote's co-term date                                    | Co-term and ramp years both derive from `EffectiveDate__c` forward in 12-month blocks — if the line's own effective date is wrong, every year shifts                                                | Fix the line's/Quote's effective date, not the ramp rule                                                                             |
| 30-month deal on a Step curve only got the 24mo discount, not something between 24 and 36 | Step mode staircases — it never interpolates. That's Linear mode's job                                                                                                                              | Switch the curve's Interpolation Mode to Linear if a smooth value is wanted                                                          |

## Limits and gotchas

- **The field `QuoteLineItem.RampSchedule__c` and the object
  `RampSchedule__c` share a name but are unrelated data** — the field is a
  JSON blob on the line (and only populated by the pre-DDCPQ-59 averaged
  path); the object is the authored per-year rule row. Always qualify
  which one a query means.
- **Term Discount Curves are deprecated but not dead.** `TermDiscountService`
  still runs every Active curve on every price/commit, and Rules Studio's
  "Term Discount Curves" panel still saves them. New pricing-by-term
  should be built as separate Rate Plans per commitment length instead —
  tell a tester building new pricing this directly rather than letting
  them invest in a curve.
- **Ramps never apply to OneTime or bundle-parent lines**, same rule as
  system/volume discounts and margin floor — the "license line" of a
  bundle is priced flat; ramps apply to its priced children.
- **Not segmented in this build**: one-time and usage charges, 12-month-
  or-shorter terms, hybrid charge-profile siblings, and lines that change
  an existing Asset (amendment/renewal — those keep the term-averaged
  ramp instead of splitting).
- **A ramp and a term curve compose multiplicatively on the same line**:
  the curve discounts whatever price the ramp already produced for that
  specific year, not the Year‑1 list price.
- **Governor limits**: `RampService` and `TermDiscountService` each load
  their Active rows once per pricing run via `EngineCache` (keyed by
  currency), regardless of how many lines or ramp years are on the quote
  — one query per rule type per run, not per line.
- **`UsageTierMatrixOverride__c`** on a Ramp Schedule is a separate,
  optional feature (swap in a different Usage Tier structure for one
  ramp year) — it has nothing to do with the rate math in this module and
  is covered by `usage-pricing.md`.

## Questions testers ask

**"Why did adding one product give me three cart rows?"** — That is the
ramp working as designed: a Recurring line over 12 months with a matching
Ramp Rule Set becomes one QuoteLineItem per year (DDCPQ-59), each with its
own price, dates and discounts, not one line with a blended rate.

**"Can I still build 'discount by term length' the old way?"** — Yes,
Term Discount Curves still work and are still editable in Rules Studio,
but they're deprecated; the forward-looking way is a separate Rate Plan
per term length.

**"If I edit Year 2's price by hand, does Year 3 recompute from my new
number?"** — No. Year 3 (while still on `Rule`) compounds off what the
rule says Year 2 _would be_, not off the manual price typed into Year 2.

**"Where do I see which rule caused a step?"** — The waterfall's
`RampAdjustedPrice` row names the year and the adjustment, e.g. "Year 3
of 3: +10% on Year 2"; for term curves, the `TermAdjustedPrice` row names
step or interpolation math directly, e.g. "Step to 24mo point → 10.00% at
24mo."

**"Does a 30-month ramp get a partial third year?"** — Yes — years are
12 months each except the last, which absorbs whatever's left (a
30-month term is 12/12/6).

**"I set `MinCommitMonths__c` higher than the quoted term — does that
also lengthen a ramp?"** — Yes, `CommitService.commitmentTerm` raises the
effective term used for both segmentation and term-curve lookups to at
least the plan's `MinCommitMonths__c`, even if the rep typed a shorter
number on the line.

**"Why does `DDCPQ__RampSchedule__c` always come back blank on my ramp
lines?"** — Because those lines went through the one-QLI-per-year split,
which never writes that field. It's only used by the older single-line
averaged path (amendments, renewals, hybrid siblings).
