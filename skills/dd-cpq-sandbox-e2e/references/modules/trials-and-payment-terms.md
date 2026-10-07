# Free trials and payment terms

> Installed build 0.1.0-9 · Verified 2026-10-06 in cpq-pkg: a Monthly $20
> plan with a 14-day free trial commits at NetPrice $20 (trial never touches
> price) and stamps TrialEndDate = today + 14 days; a $180/12-month Upfront
> plan with Prepaid 5% early-pay commits at NetPrice $171, NMR $14.25/mo,
> ARR $342; a $100 Monthly plan with Net-30 2% early-pay commits at NetPrice
> $98, ARR $3,528.

## What it is for

Two independent rate-plan dimensions that make a plan match how a real deal
is actually sold:

- **Free trial** (2027-D) — the customer uses the product at a reduced or
  zero rate for a number of days before the plan flips to its list rate.
  Think "14-day free trial" or "$1 first month."
- **Payment terms** (2027-F) — when the customer owes the invoice (On
  Receipt, Net-15/30/45/60/90, Prepaid), optionally paired with an
  **early-pay discount %** that rewards the customer for favorable terms
  (the "2/10 Net 30" pattern treasury teams use, or a prepay discount).

Both live entirely on the **Rate Plan**, not on the Quote or the Quote Line.
Every line priced under a given plan inherits that plan's trial and payment
terms; there is no per-line override.

Billing schedule (Monthly / Quarterly / Semi-Annual / Annual / Upfront For
Term / On Event / One-Time) is a third, related dimension that controls
invoice cadence and interacts with both trials and payment terms: it
decides how MRR/ARR are normalized, and Upfront For Term is where
`MinCommitMonths` is mandatory.

## How it works

### Free trial (2027-D)

- Fields live on **Rate Plan**: `TrialDays__c` (days) and
  `TrialUnitPrice__c` (price per unit during the trial; 0 = free, non-zero =
  reduced-rate like "$1 first month"). `TrialDays__c` ≤ 0 or null = no trial.
- Only **Recurring** plans can carry a trial. It is wired into every
  pricing model (Per Unit, Tiered, Volume, Package) but only the first
  year of a **ramp** gets the trial — `TrialService` skips any line whose
  `segmentIndex > 0`.
- `TrialService` runs **last** in the pipeline, after every pricing and
  discount stage, and it is metadata-only: it **never changes `NetPrice__c`**.
  The trial window is a transient invoicing-timing fact, not a price. On
  commit it stamps `QuoteLineItem.TrialEndDate__c` =
  `(EffectiveDate__c ?? today) + TrialDays__c`.
- **MRR/ARR reflect steady-state, post-trial revenue** — the Bessemer /
  Meritech convention. A line with a 14-day free trial still reports its
  full MRR and ARR the moment it is committed; the trial is a transient
  discount on _cash_, not on the metric that values the deal.
- **Round-trip is idempotent.** Once a quote line has a `TrialEndDate__c`,
  every later reprice preserves it instead of recomputing
  `today + TrialDays__c` — otherwise revisiting the cart would silently
  slide the trial forward and extend the customer's trial for free.
- The **Invoice Timeline** (see below) is the one place the trial actually
  changes a displayed number: it blends the trial rate into the affected
  months and reports a **"Trial savings"** line in its sidebar, with a note
  that ARR stays at the steady-state figure.
- The generic `dd/v1/price` preview endpoint's waterfall does **not** show
  a trial stage — trial fields never reach the waterfall array because they
  never touch `netPrice`. Nothing is wrong if you don't see one.

### Payment terms (2027-F)

- Fields live on **Rate Plan**: `PaymentTerms__c` (On Receipt, Net-15/30/45/
  60/90, Prepaid) and `PaymentTermDiscountPct__c` (0–100, optional).
- **The two are independent.** A plan can declare "we bill Net-30" with no
  discount — the term is informational, captured for downstream billing,
  and does not touch price. The discount **only fires when the % is > 0**.
- `PaymentTermDiscountService` applies the % **multiplicatively on
  `netPrice`**, late in the pipeline — after every rule-based discount
  (system, promo, channel, volume, manual, commit) and **before** the
  `MarginFloor` and final `Net` waterfall stages. The exact installed order
  is: … → `CommitDiscount` → **`PaymentTermDiscount`** → `MarginFloor` →
  `Net`. This means nothing downstream can push a price back below the
  margin floor after the early-pay discount lands, and the waterfall's
  `Net` row is always the true final number.
- It composes with everything else by simple multiplication:
  `netPrice × (1 − pct/100)`, rounded to 6 decimal places internally. A
  discount over 100 is clamped to 100 (free).
- It **skips** bundle parent lines and synthetic margin-floor adjustment
  lines, and it never re-applies to a line the rep priced by hand
  (`priceSetByHand = true` wins).
- Payment terms are **not stored on the QuoteLineItem** — there is no
  `PaymentTerms__c` or `PaymentTermDiscountPct__c` field on QLI. The only
  way to see which term a committed line used is to follow
  `QuoteLineItem.RatePlan__c` back to the Rate Plan.

### Billing schedule and billing timing

- `BillingSchedule__c` (Rate Plan, required): **Monthly / Quarterly /
  Semi-Annual / Annual / Upfront For Term (Prepay) / On Event / One-Time**.
  The Rate Plan Editor's "Billed" control only offers Monthly, Quarterly,
  Semi-Annual, Annual, Upfront — **On Event** and **One-Time** are set
  automatically (On Event for Usage/Overage plans, One-Time for a
  One-Time revenue-nature plan) and cannot be hand-picked from that
  control.
- `BillingTiming__c`: **In Advance** (default — customer pays at the start
  of the period) or **In Arrears** (default for Usage/Overage — you rate
  consumption after it happens). Shown in the editor as "Invoiced" and only
  for non-One-Time plans.
- **Normalized Monthly Rate (NMR)** is the atomic MRR primitive and is
  **per unit**: `NMR = UnitPrice__c / periodMonths`, where `periodMonths` is
  1 / 3 / 6 / 12 for Monthly/Quarterly/Semi-Annual/Annual, and
  **`MinCommitMonths__c`** for Upfront For Term. `MRR = NMR × Quantity`,
  `ARR = MRR × 12`. This is what fixes the "Zoom prepay" problem: a
  $180-for-12-months Upfront plan is **not** $180/month — it normalizes to
  $15/mo per unit before quantity.
- **`MinCommitMonths__c`** is the minimum commitment length in months.
  For Upfront For Term it is **the term the single upfront invoice covers
  and the divisor NMR uses** — if it is left blank on an Upfront plan, NMR
  resolves to 0 ("undetermined period") and MRR/ARR silently show $0.
  Switching a plan's "Billed" segment to **Upfront** in the mini rate-plan
  editor auto-fills `MinCommitMonths = 12` if it was empty — but it is a
  real field you can change, not a fixed 12.
- Trial, payment terms and billing schedule are three independent axes —
  a plan can be Monthly + 14-day trial + Net-30 2% all at once; the engine
  applies them as three separate, order-fixed pipeline stages.

## Objects and fields

| Object label (API name)           | Field                       | Meaning / values                                                                                                      |
| --------------------------------- | --------------------------- | --------------------------------------------------------------------------------------------------------------------- |
| Rate Plan (`RatePlan__c`)         | `TrialDays__c`              | Number. Days at the trial rate. 0/null = no trial.                                                                    |
| Rate Plan (`RatePlan__c`)         | `TrialUnitPrice__c`         | Currency. Price per unit during trial; 0 = free.                                                                      |
| Rate Plan (`RatePlan__c`)         | `PaymentTerms__c`           | Picklist: `OnReceipt` (default), `Net15`, `Net30`, `Net45`, `Net60`, `Net90`, `Prepaid`.                              |
| Rate Plan (`RatePlan__c`)         | `PaymentTermDiscountPct__c` | Number(5,2), 0–100. Early-pay % off `NetPrice__c`, multiplicative. Null/0 = no discount.                              |
| Rate Plan (`RatePlan__c`)         | `BillingSchedule__c`        | Picklist: `Monthly` (default), `Quarterly`, `SemiAnnual`, `Annual`, `UpfrontForTerm`, `OnEvent`, `OneTime`. Required. |
| Rate Plan (`RatePlan__c`)         | `BillingTiming__c`          | Picklist: `InAdvance` (default), `InArrears`.                                                                         |
| Rate Plan (`RatePlan__c`)         | `MinCommitMonths__c`        | Number(3,0). Minimum term; the Upfront-For-Term divisor for NMR.                                                      |
| Rate Plan (`RatePlan__c`)         | `MaxCommitMonths__c`        | Number. Optional ceiling on term length.                                                                              |
| Rate Plan (`RatePlan__c`)         | `IsEvergreen__c`            | Checkbox. "Renews until cancelled" in the editor.                                                                     |
| Rate Plan (`RatePlan__c`)         | `RevenueNature__c`          | `Recurring` / `OneTime` / `Usage` / `Overage`. Only `Recurring` plans can carry a trial.                              |
| Quote Line Item (`QuoteLineItem`) | `TrialEndDate__c`           | Date. Stamped at commit = EffectiveDate (or CreatedDate) + TrialDays. Null when the plan has no trial.                |
| Quote Line Item (`QuoteLineItem`) | `RatePlan__c`               | Lookup to Rate Plan. The only way to read back which Payment Terms a committed line used.                             |
| Quote Line Item (`QuoteLineItem`) | `NormalizedMonthlyRate__c`  | Currency. Per-unit NMR = NetPrice / periodMonths.                                                                     |
| Quote Line Item (`QuoteLineItem`) | `MRR__c`                    | Currency. `NMR × Quantity`. Steady-state, ignores trial.                                                              |
| Quote Line Item (`QuoteLineItem`) | `ARR__c`                    | Currency. `MRR × 12`.                                                                                                 |
| Quote Line Item (`QuoteLineItem`) | `TermMonths__c`             | Number. Term length carried onto the line (e.g. 12 for the Upfront example).                                          |
| Quote Line Item (`QuoteLineItem`) | `NetPrice__c`               | Currency. Final price after every discount stage including Payment Term Discount — unaffected by trial.               |

There is **no** `PaymentTerms__c` or `PaymentTermDiscountPct__c` field, and
no `BillingSchedule__c` field, on `QuoteLineItem` itself — only on the Rate
Plan it points at.

## Build it (tester)

Build three standalone products (no bundle needed) in a fresh scratch org,
each with its own Active Recurring Per-unit plan. Use your own prefix on
every record so it never collides with another tester's data.

**Product A — free trial.** Products tab → New → `<PFX> Trial Seat`,
Active. Add a Standard Price of $20. Open **Rate Plan Editor** → pick the
product → **Blank plan**. In **Shape**: Charge **Recurring**, Priced
**Per unit**, price **20**, Billed **Monthly**. In **Terms**: tick
**Trial period** → **Days** `14`, **Price during trial** leave blank
(= free). Leave **Payment terms** at its default. Set Status **Active** →
Save.

**Product B — Upfront prepay with an early-pay discount.** New product
`<PFX> Prepay Annual`, Standard Price $180. New blank plan: Charge
**Recurring**, Priced **Per unit**, price **180**, Billed **Upfront** (the
Term field appears, showing "months, paid upfront") → Term `12`. In
**Terms**: **Payment terms** → **Prepaid**, **Early-pay discount %** → `5`.
Activate → Save.

**Product C — Net-30 early-pay discount.** New product
`<PFX> Net30 Service`, Standard Price $100. New blank plan: Charge
**Recurring**, Priced **Per unit**, price **100**, Billed **Monthly**. In
**Terms**: **Payment terms** → **Net 30**, **Early-pay discount %** → `2`.
Activate → Save.

Build a quote, **Configure Products**, add all three at quantities
**10 / 2 / 3**. Expect:

- `<PFX> Trial Seat` — a purple **"Free trial · 14 day(s) left · through
  <date+14>"** badge next to the name; net price stays **$20**.
- `<PFX> Prepay Annual` — an amber **"Prepaid · 5% off"** badge; net price
  **$171** (180 × 0.95); term shows 12 months.
- `<PFX> Net30 Service` — an amber **"Net-30 · 2% off"** badge; net price
  **$98** (100 × 0.98).

Open the price column's waterfall drawer on Prepay Annual or Net30
Service — a **`PaymentTermDiscount`** row appears just before **Net**,
with a note like `PaymentTermDiscount: Prepaid → 5.00% off (180.000000 ×
0.95 = 171.000000)`. The Trial Seat line's waterfall has **no** trial row
at all — look for the badge, not the waterfall.

Switch the cart to the **Timeline** view (button next to Table at the top
of the cart). The Trial Seat row's first two months shade differently and
the sidebar shows a **"Trial savings"** line; **ARR** in the sidebar still
reads the steady-state figure, not the trial-discounted one. **Commit.**

## Check it (Claude)

```bash
sf data query -o dd-e2e -q "SELECT Product2.Name, Quantity, UnitPrice, DDCPQ__DerivedListPrice__c, DDCPQ__NetPrice__c, DDCPQ__NormalizedMonthlyRate__c, DDCPQ__MRR__c, DDCPQ__ARR__c, DDCPQ__TrialEndDate__c, DDCPQ__TermMonths__c, DDCPQ__RatePlan__r.DDCPQ__PaymentTerms__c, DDCPQ__RatePlan__r.DDCPQ__PaymentTermDiscountPct__c, DDCPQ__RatePlan__r.DDCPQ__BillingSchedule__c FROM QuoteLineItem WHERE Quote.Name = '<their quote name>'"
```

Expect `TrialEndDate__c` set only on the trial line, equal to today +
`TrialDays__c` (first commit) and unchanged on later reprices.
`PaymentTerms__c`/`PaymentTermDiscountPct__c` come back via the
`RatePlan__r` relationship — there is nothing to query directly on the
QLI for them.

```bash
sf data query -o dd-e2e -q "SELECT DDCPQ__TargetProduct__r.Name, DDCPQ__Status__c, DDCPQ__TrialDays__c, DDCPQ__TrialUnitPrice__c, DDCPQ__PaymentTerms__c, DDCPQ__PaymentTermDiscountPct__c, DDCPQ__BillingSchedule__c, DDCPQ__MinCommitMonths__c, DDCPQ__UnitPrice__c FROM DDCPQ__RatePlan__c WHERE DDCPQ__Status__c = 'Active'"
```

Confirms the authored plan before troubleshooting a wrong number — check
`Status__c = Active` first; a Draft plan is invisible to the engine.

**Read-only price preview** (never commit on the tester's behalf):

```json
{
  "quoteId": "<0Q0…>",
  "selections": [
    {
      "localKey": "l1",
      "productId": "<Prepay Annual 01t…>",
      "ratePlanId": "<its a0c…>",
      "quantity": 2,
      "termMonths": 12
    }
  ]
}
```

```bash
sf api request rest "services/apexrest/DDCPQ/dd/v1/price?includeWaterfall=true" -o dd-e2e --method POST --body @body.json
```

The response's `netPrice` already reflects the Payment Term Discount; its
`waterfall` array shows the `PaymentTermDiscount` stage with the exact
multiplication in `note`. It does **not** show anything trial-related —
that is expected, not a bug (see "How it works").

## Expected numbers

Verified 2026-10-06 in `cpq-pkg` — three standalone products, no rules, no
bundle, committed quote lines:

| Product               | List | Plan                               | Net price | NMR   | MRR             | ARR      | TrialEndDate (today 2026-10-06) |
| --------------------- | ---- | ---------------------------------- | --------- | ----- | --------------- | -------- | ------------------------------- |
| TP Trial Seat         | 20   | Monthly, 14-day free trial         | 20.00     | 20.00 | 200.00 (qty 10) | 2,400.00 | 2026-10-20 (today+14)           |
| TP Reduced Trial Seat | 50   | Monthly, 30-day trial @ $1         | 50.00     | 50.00 | 250.00 (qty 5)  | 3,000.00 | 2026-11-05 (today+30)           |
| TP Prepay Annual      | 180  | Upfront/12mo, Prepaid 5% early-pay | 171.00    | 14.25 | 28.50 (qty 2)   | 342.00   | — (no trial)                    |
| TP Net30 Service      | 100  | Monthly, Net-30 2% early-pay       | 98.00     | 98.00 | 294.00 (qty 3)  | 3,528.00 | — (no trial)                    |

Quote totals from the same preview: `totalNet` **1,086.00**
(200 + 250 + 342 + 294), `totalArr` **9,270.00**
(2,400 + 3,000 + 342 + 3,528) — both match the per-line math above exactly,
confirming NMR/MRR/ARR and the Net price are independent of trial and
driven purely by `NetPrice__c / periodMonths × Quantity × 12`.

## Troubleshooting

| Symptom                                                            | Cause                                                                                   | Fix                                                                                                                        |
| ------------------------------------------------------------------ | --------------------------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------- |
| Trial badge never appears                                          | Plan's `RevenueNature__c` is not `Recurring`, or `TrialDays__c` is 0/blank              | Trial only wires into Recurring plans. Open the plan, tick **Trial period**, set Days > 0.                                 |
| Price didn't change during the trial                               | Expected — Trial Service never mutates `NetPrice__c`                                    | Not a bug. The trial shows up as a badge and in the Timeline view, not in `NetPrice__c` or the waterfall.                  |
| `PaymentTermDiscount` row missing from the waterfall               | `PaymentTermDiscountPct__c` is null or 0 on the plan                                    | A payment **term** alone (e.g. Net-30 with no %) never adds a waterfall row — only a discount > 0 does.                    |
| Net price looks like list even though a discount % was set         | The plan is still **Draft**                                                             | `PaymentTermDiscountService` only runs on `Active` plans the engine actually selected. Activate it.                        |
| Every line shows an "On Receipt" badge even though you set nothing | `PaymentTerms__c` defaults to `OnReceipt`, not blank                                    | Expected. Every Recurring plan carries some payment term; `OnReceipt` with no discount is the quiet default, not an error. |
| Upfront plan's MRR/ARR show $0                                     | `MinCommitMonths__c` is blank on a Billed-Upfront plan                                  | NMR's divisor is undefined without it. Set a value (commonly 12) in **Minimum term (months)**.                             |
| Discount applied twice as big as expected                          | Quantity confused with the discount, or pct typed as a fraction (`0.05` instead of `5`) | `PaymentTermDiscountPct__c` is a whole percent, e.g. `5` = 5%, not `0.05`.                                                 |
| Discount > 100 made the line free instead of erroring              | By design — `PaymentTermDiscountService` clamps at 100                                  | Not a bug; the engine protects against a negative price rather than failing the quote.                                     |
| TrialEndDate slides forward every time you reopen the cart         | A client bug would look like this, but the installed service is idempotent              | If it does slide, capture the quote Id and the two TrialEndDate values — that is a real regression, file it.               |
| Commit refused needing an Active rate plan                         | No Active plan exists for the product at all (same root cause as the golden path)       | See golden-path.md Stage 6 — trial/payment-term fields live on the plan, but the plan still needs `Status__c = Active`.    |
| Ramp Year 2+ still shows a trial in the Timeline                   | Should not happen — `TrialService` skips `segmentIndex > 0`                             | If a later ramp year shows the trial rate, that is a regression; capture the quote and ramp schedule.                      |

## Limits and gotchas

- Trial and payment terms are **plan-level only** — there is no way to
  give one customer a trial and another customer the same plan without a
  trial; clone the plan instead.
- A trial only applies to the **first year of a ramp** (`segmentIndex == 0`
  or null); later ramp years always bill at the ramp's own rate.
- The generic `dd/v1/price` preview response carries no trial fields at
  all (no `trialDays`, `trialUnitPrice`, `trialEndDate` in its line
  objects) — only the full cart bootstrap (what the LWC loads) and the
  committed `QuoteLineItem.TrialEndDate__c` carry it. Don't expect to see
  trial in a raw REST price preview.
- `PaymentTermDiscountService` skips bundle **parent** lines and synthetic
  margin-floor adjustment lines by design — a parent's own early-pay
  discount, if it has one, never shows on the header line.
- `MaxCommitMonths__c` exists on the plan but nothing in the installed
  build enforces it against the quote's actual term — it is informational
  only in 0.1.0-9.
- Billing Schedule's **On Event** and **One-Time** values exist on the
  object but are not reachable from the "Billed" segmented control in
  either Bundle Builder or the Rate Plan Editor — they are set
  automatically from the plan's Charge type.
- Trial + Payment Term Discount can both be set on the same plan; they are
  independent stages and do not interact (the trial still makes `NetPrice__c`
  unaffected either way).

## Questions testers ask

**Q: Why doesn't the price change while the customer is in their free
trial?**
A: By design. `NetPrice__c`, `MRR__c` and `ARR__c` always reflect the
steady-state, post-trial rate (the Bessemer/Meritech convention investors
use to value SaaS). The trial shows up as a badge and in the Invoice
Timeline, not in the price.

**Q: Where do I see what a customer actually pays during the trial
window?**
A: The Timeline view's cells for the trial months, and the sidebar's
"Trial savings" line. There is no dedicated trial-rate field on the QLI
beyond `TrialEndDate__c` — the trial _rate_ lives on the Rate Plan
(`TrialUnitPrice__c`), not the line.

**Q: Can I give a discount for Net-30 terms without changing the invoice
due date logic?**
A: Yes — set `PaymentTermDiscountPct__c` without worrying about dunning;
actual Net-30 collections logic is a downstream billing engine's job, not
DD CPQ's.

**Q: My Upfront plan's ARR looks wrong — too small.**
A: Check `MinCommitMonths__c`. NMR divides `UnitPrice__c` by it; a 12-month
$180 plan with `MinCommitMonths` blank computes NMR = 0, not $15.

**Q: Does a trial extend every time I reopen the cart?**
A: No — the server preserves the already-stamped `TrialEndDate__c` on
every reprice. If you see it moving forward, that is a bug worth reporting.

**Q: Can two products in the same bundle have different payment terms?**
A: Yes — each option can point at its own Rate Plan with its own
`PaymentTerms__c`/discount; the bundle parent's own line is skipped from
the Payment Term Discount stage either way.

**Q: Is there a field that says "this plan offers a trial" I can filter
on?**
A: Query `DDCPQ__RatePlan__c` where `DDCPQ__TrialDays__c > 0` — there is no
separate boolean; the number itself is the flag.
