# Prepaid credit, minimum commitments, and commitment discount rules

> Installed build 0.1.0-9 · Verified 2026-10-06/07 in org **cpq-pkg** (0.1.0-9
> installed): 8% commitment discount dropped $100 → **$92** net on 1,000
> units; a $50K prepaid pool burned to **$23,000 remaining** at $27,000
> usage and **$5,000 overrun** at $55,000 usage; a $24,000/12mo
> MonthlyFloor commitment showed a **$1,500 projected shortfall** at $500
> hypothetical usage.

## What it is for

Three related but **separate** features live on this page, because DD CPQ
borrows the word "commitment" for all three:

1. **Prepaid credit (Snowflake pattern)** — a customer pre-buys a dollar
   pool (e.g. "$50K Snowflake Credits") and a Usage rate plan draws down
   against it until it is exhausted, then overflows to list-rate overage.
2. **Account-level minimum commitment floor (DDCPQ-046)** — a
   `CommitmentContract__c` promises a customer will spend at least
   `CommitAmount / TermMonths` per period, across one or more products.
   DD CPQ captures the _terms_; it never invents actual usage, so it never
   bills a shortfall itself.
3. **Commitment discount rules** — a locked discount rate a customer earns
   for _having_ an active commitment at all (the Anthropic/OpenAI
   "bigger spend commitment, bigger discount" rate card). This one DOES
   run live, in the pricing pipeline, on every quote.

A fourth, smaller "minimum commit" lives on `RatePlan__c` itself
(`MinimumCommitAmount__c` + `MinCommitMonths__c`) and feeds
`QuoteLineItem.CommittedUsageMRR__c` — a single-plan dollar floor, not the
account-wide `CommitmentContract__c`. See "How it works" below for how to
tell all four apart. (A fifth — `ProductOption__c.MinimumCommit__c`, a
per-line **quantity** floor consumed by `UsageTierService` — belongs to
the usage-pricing module, not this one; it never touches a dollar amount.)

## How it works

### 1. Prepaid credit / drawdown (DDCPQ-2027-G)

- A **deposit plan**: `RatePlan__c.IsPrepaidCredit__c = true`, almost
  always `RevenueNature = OneTime`, `BillingSchedule = OneTime` or
  `FlatFee` pricing. Committing it on a quote writes an ordinary QLI; no
  special ledger object exists.
- A **drawdown plan**: a Usage rate plan whose
  `RatePlan__c.DrawdownFromPlanId__c` points at the deposit plan's Id.
- **The runtime pricing pipeline is untouched by this.** Drawdown
  reconciliation ("how much of the pool is left") is a **billing-time**
  concern — at quote time there is no real usage yet. `PrepaidCreditService`
  is a read-only **what-if simulator**: give it hypothetical usage dollars
  and it tells you remaining balance, overrun, and per-usage-line
  attribution. It never runs inside commit or price.
- Math: for each pool, `remaining = deposit - drawdown` (floored at 0);
  `overrun = drawdown - deposit` (floored at 0); once `drawdown >= deposit`
  the pool `isExhausted`. Usage lines apply **in the order you send them** —
  the first dollar of overflow spills, not the biggest line.

### 2. Minimum commitment floor (DDCPQ-046, revised 2026-08-25)

- `CommitmentContract__c.FloorMode__c = 'MonthlyFloor'` + `CommitAmount__c`
  - `TermMonths__c` → **period floor** = `CommitAmount / TermMonths`.
    `CoveredProducts__c` (JSON array of Product2 Ids) scopes which products'
    spend counts toward the floor; **empty = every product on the account**.
- **This never runs at quote/commit time.** An earlier version
  auto-injected a synthetic "shortfall" QLI into the cart; Venkat cut that
  (2026-08-25) because DD CPQ has no real usage at quote time to measure a
  shortfall against — that is downstream billing's job
  (`bill = MAX(actualUsage, floor)` each period). `CommitmentFloorService`
  is pure math now: `evaluate(contract, hypotheticalDrafts)` and
  `evaluateAll(hypotheticalDrafts, accountId)`. No DML, no mutation, never
  wired into `CpqEngine.runPipeline` or `previewDraftsV2`.
- **`AdjustmentProduct__c` on the contract is a leftover field.** It is
  still selected by SOQL in `CommitmentFloorService.evaluateAll` but
  nothing reads it any more — the synthetic adjustment-line code path that
  used to consult it was removed in the same refactor. Setting it does
  nothing in this build; don't spend tester time chasing it.
- `FloorMode__c = 'Drawdown'` (the default) means this contract is NOT a
  monthly floor at all — it is the commitment-discount kind, below.

### 3. Commitment discount rules (`CommitmentDiscountRule__c`)

- Runs live, as **pipeline stage 10.5** — after the manual discount, before
  payment-term discount and margin floor (step 10 "System + Volume
  Discounts" → 10.5 "Commitment Discount" → margin floor → commit).
- Skipped for: **OneTime** charges, **bundle parents**, any line whose
  price was typed by hand (`priceSetByHand`), and any line whose Account
  has no Active commitment covering it.
- For each eligible line, `CommitmentPricingService`:
  1. Loads the Account's Active commits (`Status='Active'`,
     `StartDate <= today <= EndDate`), newest/largest first.
  2. Picks the best commit for the line — a commit scoped to the line's
     product's `TargetCatalog` beats an unscoped one; ties go to the
     larger `CommitAmount`.
  3. Picks the first Active `CommitmentDiscountRule__c`, in
     `Priority__c` order (lowest first), whose **criteria match** AND
     whose `MinCommitAmount__c` tier gate is at or below the matched
     commit's `CommitAmount__c` AND whose `TargetCommitmentUnit__c`
     (if set) matches the commit's `CommitUnit__c`.
  4. Multiplies `netPrice` by `(1 - DiscountPct/100)`. **Multiplicative**
     with every other discount stage, not additive.
  5. Stamps `commitmentId` / `commitDiscountPct` / `commitDrawdownAmount`
     on the line. `commitDrawdownAmount = committedNetPrice × quantity` —
     a **record of this period's spend**, not a real-time balance.
- At commit, `CommitmentDrawdownService.reconcile` re-sums every
  non-deleted QLI's `CommitDrawdownAmount__c` grouped by `Commitment__c`
  and writes the total to `CommitmentContract__c.ConsumedAmount__c`. It
  locks the contract row (`FOR UPDATE`) first so two concurrent commits on
  the same Account can't double-book. This is a full re-sum each time
  (idempotent), not a delta increment.
- Uses the **shared 5-field criteria schema** (CLAUDE.md rule 5):
  `FieldApiName__c` / `Operator__c` / `Value__c` / `ValueType__c` +
  `Priority__c`. Empty criteria = catch-all.

### Tell the four "minimum" fields apart

| Field                                                       | Lives on                                  | Scope                                          | Runs when                                                       | Billing effect                                                           |
| ----------------------------------------------------------- | ----------------------------------------- | ---------------------------------------------- | --------------------------------------------------------------- | ------------------------------------------------------------------------ |
| `CommitmentContract__c.FloorMode='MonthlyFloor'`            | the Account                               | one or more products, via `CoveredProducts__c` | **what-if only**, REST `commit/floor/evaluate`                  | none — captures the term; billing enforces it                            |
| `RatePlan__c.MinimumCommitAmount__c` + `MinCommitMonths__c` | one Usage rate plan                       | that one plan                                  | **live**, in the pricing pipeline, feeds `CommittedUsageMRR__c` | guaranteed-revenue display metric only                                   |
| `ProductOption__c.MinimumCommit__c`                         | one bundle option                         | that one line's **quantity**                   | live, `UsageTierService`                                        | `billableQty = MAX(actualQty, minimumCommit)` — see usage-pricing module |
| `CommitmentDiscountRule__c`                                 | a rule row, gated by `MinCommitAmount__c` | whichever commit it's gated against            | **live**, stage 10.5                                            | real discount on `NetPrice__c`                                           |

`CommittedUsageMRR__c = MinimumCommitAmount / MinCommitMonths` (not
`TermMonths` — a plan-level setting, separate from the quote's term). Per
[[pattern_a_max_semantics]], the cart shows **MAX(committed floor,
projected usage)**, never additive — a $100/mo floor plus $12K of actual
usage bills $12K, not $12,100. That MAX logic lives in the cart's JS
(`cpqCart.js` / `cpqCartLine.js`), not on the QLI — `MRR__c` itself stays
recurring-only (Bessemer-clean); `CommittedUsageMRR__c` is the separate
field that carries the floor.

## Objects and fields

**Commitment Contract** (`CommitmentContract__c`, autonumber `COMMIT-{00000}`)

| Field                         | Meaning / values                                                                                                    |
| ----------------------------- | ------------------------------------------------------------------------------------------------------------------- |
| `Account__c`                  | Customer who owns this commitment                                                                                   |
| `CommitAmount__c`             | Total commitment size over the whole term, in `CommitUnit__c`                                                       |
| `CommitUnit__c`               | `Dollar` / `Seat` / `Request` / `DBU (Databricks Unit)` / `API Call` / `GB-Hour`                                    |
| `TermMonths__c`               | Length of the commitment in months                                                                                  |
| `StartDate__c` / `EndDate__c` | Only commits active today (`Start<=today<=End`) count                                                               |
| `FloorMode__c`                | `Drawdown (prepaid burn)` (default — commitment-discount lines) or `Monthly Floor (Snowflake style)` (what-if only) |
| `CoveredProducts__c`          | JSON array of Product2 Ids the MonthlyFloor sums across; empty = every product                                      |
| `AdjustmentProduct__c`        | **Dead field** in this build — see "How it works"                                                                   |
| `RolloverPolicy__c`           | `None (Use or Lose)` / `Partial (see RolloverPct)` / `Full (100%)`                                                  |
| `RolloverPct__c`              | % of unused commit that rolls over when Partial                                                                     |
| `TargetCatalog__c`            | Optional — scopes the commit to one Catalog; empty = every product for the Account                                  |
| `Status__c`                   | `Draft` / `Active` / `Consumed` / `Expired` — only Active participates                                              |
| `ConsumedAmount__c`           | Running total, re-summed from QLIs by `CommitmentDrawdownService` on every commit                                   |

**Commitment Discount Rule** (`CommitmentDiscountRule__c`, autonumber `CDR-{00000}`)

| Field                                                           | Meaning / values                                                                          |
| --------------------------------------------------------------- | ----------------------------------------------------------------------------------------- |
| `DiscountPct__c`                                                | Locked discount %, multiplicative with the rest of the stack                              |
| `MinCommitAmount__c`                                            | Tier gate — rule only fires if the matched commit's `CommitAmount__c` is at or above this |
| `TargetCommitmentUnit__c`                                       | Optional — only commits in this `CommitUnit` qualify; blank = any unit                    |
| `TargetProduct__c`                                              | Apply to one product; blank + `Target__c` blank = every product                           |
| `Target__c`                                                     | Rules-Studio-written JSON scoping (All / Products / Criteria / Group)                     |
| `FieldApiName__c` / `Operator__c` / `Value__c` / `ValueType__c` | Shared 5-field criteria (rule 5); empty = catch-all                                       |
| `Priority__c`                                                   | Evaluation order, lowest first; first match wins                                          |
| `Status__c`                                                     | `Draft` / `Active` / `Archived` — only Active fires                                       |

**Rate Plan** (`RatePlan__c`) — prepaid + drawdown additions

| Field                    | Meaning / values                                                                  |
| ------------------------ | --------------------------------------------------------------------------------- |
| `IsPrepaidCredit__c`     | Checkbox — this plan is a deposit pool                                            |
| `DrawdownFromPlanId__c`  | Lookup(RatePlan__c) — this (Usage) plan draws against the referenced pool         |
| `MinimumCommitAmount__c` | Single-plan dollar floor (Snowflake $25K/yr style) — feeds `CommittedUsageMRR__c` |
| `MinCommitMonths__c`     | Denominator for that floor; defaults to 12 if blank                               |

**Quote Line Item** — commitment-stamped fields

| Field                     | Meaning / values                                                                                      |
| ------------------------- | ----------------------------------------------------------------------------------------------------- |
| `Commitment__c`           | The `CommitmentContract__c` this line drew its discount from                                          |
| `CommitDiscountPct__c`    | The locked % that fired, from `CommitmentDiscountRule__c`                                             |
| `CommitDrawdownAmount__c` | `NetPrice × Quantity` after the commit discount — this period's booked spend                          |
| `CommittedUsageMRR__c`    | `RatePlan.MinimumCommitAmount__c / MinCommitMonths__c`, zero unless that plan sets a floor            |
| `IsFloorAdjustment__c`    | **Always false in this build** — the MonthlyFloor auto-injection path that used to set it was removed |

## Build it (tester)

Scenario: a customer with a $100,000 Dollar commitment gets a locked 8%
off one product, as long as their commitment is at least $50,000.

1. **Account** `PC Commit Co`.
2. **Product** `PC Commit Product`, Active, Standard Price **$100**.
   Give it a Rate Plan: Recurring / Per unit / Monthly / **$100**, Active.
3. **Commitment Contracts tab → New**:
   - Account `PC Commit Co`
   - Commit Amount **100000**, Commit Unit **Dollar**
   - Term (Months) **12**
   - Start Date today, End Date 12 months out
   - Floor Mode **Drawdown (prepaid burn)**
   - Status **Active** → Save.
4. **Commitment Discount Rules tab → New**:
   - Applies To Product `PC Commit Product`
   - Discount % **8**
   - Min Commit Amount **50000**
   - Status **Active** → Save.
5. **Opportunity** `PC Commit Opp` on that account → **New Quote**
   `PC Commit Quote` → **Configure Products**.
6. **Add products** → `PC Commit Product`, quantity **1,000**.
7. Read the line: net price **$92.00** (list $100 → −8% commit discount).
   Open the waterfall — the last two rows before Net both read 92, with a
   `CommitDiscount` row noting _"Commit COMMIT-00001 locked rate 8.00%
   off"_.
8. **Commit.** The QLI's `Commitment__c` now points at the contract,
   `CommitDrawdownAmount__c` = 92 × 1000 = **92,000**.

For prepaid credit, build the Snowflake pattern instead: one product +
Rate Plan with **Is Prepaid Credit Pool** ticked (price $50,000, FlatFee,
OneTime), and a second Usage product/plan with **Draws From Plan** set to
the first. There is no cart-side "balance" to watch — drawdown is a
what-if simulation Claude can run for you (below), not something the
commit screen shows as a running total.

For the account-level MonthlyFloor: Commitment Contract with Floor Mode
**Monthly Floor (Snowflake style)**, a Commit Amount and Term Months, and
optionally a Covered Products picker (on the Commitment Contract's own
record page — add `cpqCoveredProductsPicker` there via Lightning App
Builder if it isn't already placed). There is nothing to click on the
quote for this one; it is a contract term, checked by Claude's what-if
query below, not by anything in the cart.

## Check it (Claude)

```bash
sf data query -o cpq-pkg -q "SELECT Name, DDCPQ__Account__r.Name, DDCPQ__CommitAmount__c, DDCPQ__CommitUnit__c, DDCPQ__FloorMode__c, DDCPQ__Status__c, DDCPQ__ConsumedAmount__c FROM DDCPQ__CommitmentContract__c WHERE DDCPQ__Account__r.Name = 'PC Commit Co'"

sf data query -o cpq-pkg -q "SELECT Name, DDCPQ__TargetProduct__r.Name, DDCPQ__DiscountPct__c, DDCPQ__MinCommitAmount__c, DDCPQ__Status__c FROM DDCPQ__CommitmentDiscountRule__c WHERE DDCPQ__TargetProduct__r.Name = 'PC Commit Product'"

sf data query -o cpq-pkg -q "SELECT Product2.Name, Quantity, UnitPrice, DDCPQ__NetPrice__c, DDCPQ__Commitment__c, DDCPQ__CommitDiscountPct__c, DDCPQ__CommitDrawdownAmount__c FROM QuoteLineItem WHERE Quote.Name = 'PC Commit Quote'"

sf data query -o cpq-pkg -q "SELECT DDCPQ__TargetProduct__r.Name, DDCPQ__IsPrepaidCredit__c, DDCPQ__DrawdownFromPlanId__r.DDCPQ__DisplayLabel__c, DDCPQ__MinimumCommitAmount__c, DDCPQ__MinCommitMonths__c FROM DDCPQ__RatePlan__c WHERE DDCPQ__IsPrepaidCredit__c = true OR DDCPQ__DrawdownFromPlanId__c != null"
```

**Read-only what-ifs** — these are the two safe REST calls for this
feature (both are GET-equivalent in effect: no DML, no commit). Write the
body to a file first.

Minimum-commitment floor projection:

```json
{
  "accountId": "001...",
  "hypotheticalUsage": [{ "productId": "01t...", "amount": 500 }]
}
```

```bash
sf api request rest "services/apexrest/DDCPQ/dd/v1/commit/floor/evaluate" -o cpq-pkg --method POST --body @body.json
```

Prepaid-credit burndown — either load a committed quote's deposit lines
(`{"quoteId": "0Q0...", "hypotheticalUsage": [...]}`) or run pure math
with no quote at all:

```json
{
  "deposits": [
    { "ratePlanId": "a0X...", "ratePlanLabel": "Pool", "amount": 50000 }
  ],
  "hypotheticalUsage": [
    {
      "ratePlanId": "a0X...(the USAGE plan's Id)",
      "amount": 12000,
      "localKey": "jan"
    }
  ]
}
```

```bash
sf api request rest "services/apexrest/DDCPQ/dd/v1/commit/credit/simulate" -o cpq-pkg --method POST --body @body.json
```

Send the **usage plan's** Id in `hypotheticalUsage[].ratePlanId`, not the
pool's — the service resolves it to the pool internally
(`PrepaidCreditService.resolveUsagePlansToPools`). This is the correct,
documented contract; a few in-app tooltips print the endpoint path
without the `/commit` segment (`/dd/v1/credit/simulate`) — the real path
always needs it: `/dd/v1/commit/credit/simulate`.

## Expected numbers

| Scenario                                                                           | Input                          | Result                                                                                                                                           |
| ---------------------------------------------------------------------------------- | ------------------------------ | ------------------------------------------------------------------------------------------------------------------------------------------------ |
| Commitment discount, 8% off, qty 1,000 @ $100                                      | commit $100,000 ≥ $50,000 gate | NetPrice **$92.00**, waterfall note _"Commit COMMIT-00001 locked rate 8.00% off"_, `totalArr` on the preview line **1,104,000** (92 × 1000 × 12) |
| Prepaid pool $50,000, usage $12,000 + $15,000                                      | under budget                   | `totalDrawdown` **27,000**, `totalRemaining` **23,000**, `totalOverrun` **0**, `isExhausted` **false**                                           |
| Prepaid pool $50,000, usage $30,000 + $25,000                                      | overrun                        | `totalDrawdown` **55,000**, `totalRemaining` **0**, `totalOverrun` **5,000**, `isExhausted` **true**, second line `spilled` **5,000**            |
| MonthlyFloor $24,000 / 12 mo, hypothetical usage $500                              | covers 1 product               | `periodFloor` **2,000**, `coveredTotal` **500**, `shortfall` **1,500**                                                                           |
| Same contract, hypothetical usage $2,500                                           | —                              | `shortfall` **0** (covered ≥ floor)                                                                                                              |
| Same contract, zero hypothetical usage                                             | —                              | `shortfall` **2,000** (the full period floor)                                                                                                    |
| Same contract, `CoveredProducts` scoped to a different product than the usage line | —                              | `coveredTotal` **0**, `shortfall` **2,000** — uncovered spend never counts                                                                       |

## Troubleshooting

| Symptom                                                                                                        | Cause                                                                                                                                                                                   | Fix                                                                                                                                          |
| -------------------------------------------------------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------- |
| Net price doesn't drop despite an Active discount rule                                                         | Rule's `MinCommitAmount__c` gate is above the contract's `CommitAmount__c`, or `TargetCommitmentUnit__c` doesn't match the contract's `CommitUnit__c`                                   | Lower the gate, or match the unit, or raise the contract's Commit Amount                                                                     |
| Still no discount                                                                                              | The line's charge type is OneTime, or it's a bundle parent, or the rep typed the price by hand                                                                                          | Commitment discounts only touch Recurring/Usage/Overage lines nobody has manually priced — by design                                         |
| Still no discount after that                                                                                   | No Active `CommitmentContract__c` on the Account covering today's date                                                                                                                  | Check `Status='Active'` and `StartDate <= today <= EndDate`                                                                                  |
| `floor/evaluate` returns `"accountId is required"` (400)                                                       | Request body missing `accountId`                                                                                                                                                        | Always send `{"accountId": "001..."}`, even with empty `hypotheticalUsage`                                                                   |
| `covered-products` returns `"contractId is required"` (400)                                                    | Missing `contractId` in body                                                                                                                                                            | `{"contractId": "a08...", "productIds": [...]}`                                                                                              |
| `credit/simulate` returns `"Either quoteId or deposits[] is required."` (400)                                  | Body has neither                                                                                                                                                                        | Send one or the other — never both are required, but one is                                                                                  |
| Usage amount shows up in `unmatchedUsage[]` instead of being drawn down                                        | `hypotheticalUsage[].ratePlanId` doesn't resolve to a pool — either it's not a real Rate Plan, or that plan's `DrawdownFromPlanId__c` is blank                                          | Confirm the usage plan's Drawdown From Plan field is set, and that you sent the usage plan's Id (not a random Id)                            |
| Tester sets `AdjustmentProduct__c` on a MonthlyFloor contract expecting an adjustment line to appear on commit | Dead field in this build — the auto-injection path was removed 2026-08-25                                                                                                               | Nothing to fix; the field does nothing. Don't build a scenario around it                                                                     |
| Commit fails with error code `FLOOR_BLOCKED`                                                                   | This is the **Margin Floor** feature (a different "floor" — see margin-and-manual-discounts.md), not the minimum-commitment floor in this module                                        | Don't confuse the two; a commitment-floor shortfall never blocks a commit, it's informational only                                           |
| `cpqCommitmentPanel` / `cpqCoveredProductsPicker` not visible anywhere                                         | Neither ships on a default page layout — both are `lightning__RecordPage`-exposed components the admin places manually                                                                  | Lightning App Builder: drop `cpqCommitmentPanel` on the Quote record page, `cpqCoveredProductsPicker` on the Commitment Contract record page |
| `CommittedUsageMRR__c` stays 0 even though `MinimumCommitAmount__c` is set on the plan                         | `RevenueNature__c` isn't the issue (committedUsageMrr is computed independent of nature) — check `MinimumCommitAmount__c` is actually > 0 and `MinCommitMonths__c` isn't accidentally 0 | Re-check the plan's two fields; 0 or negative `MinimumCommitAmount__c` always yields 0                                                       |

## Limits and gotchas

- **Nothing here runs inside `CpqEngine.runPipeline`'s commit path except
  the commitment discount rules.** Prepaid-credit drawdown and
  MonthlyFloor shortfalls are both read-only what-if simulators called
  directly over REST/MCP — there is no live "balance remaining" anywhere
  in the cart UI for either. If a tester expects to watch a credit pool
  drain as they add usage lines, tell them plainly: that is not built;
  run the simulator instead.
- **No `CommitmentLedger__c` object exists.** `ConsumedAmount__c` on the
  contract is a direct re-sum of QLIs by `CommitmentDrawdownService`, not
  a ledger of discrete events. Don't go looking for ledger rows.
- **`AdjustmentProduct__c` and `IsFloorAdjustment__c` are vestigial** in
  this build — schema from the original (pre-refactor) DDCPQ-046 that
  auto-injected shortfall lines. The refactor (2026-08-25) kept the
  fields but removed every code path that wrote to them from a real
  commit. `IsFloorAdjustment__c` will always read `false` on anything you
  commit in this build.
- **A commitment that matches two Active discount rules only ever fires
  one** — the first match by `Priority__c` (lowest first) wins; it does
  not stack multiple commitment-discount rules on the same line.
- **The commitment discount is multiplicative, not a replacement** — it
  stacks on top of whatever system/volume/promotion/channel/manual
  discount already landed on the line. A line that is already at $0 stays
  at $0; 8% of $0 is $0.
- **Usage attribution order in the simulator is caller-supplied, not
  chronological** — if you send February's usage before January's, the
  simulator burns it in that order. There's no date field on
  `hypotheticalUsage` entries.
- **`CommitUnit__c` has no "USD" value** — it's `Dollar`. A seed script or
  REST body that sends `"USD"` will fail picklist validation.
- **Governor limits:** `CommitmentDrawdownService.reconcileContracts`
  does one `FOR UPDATE` SOQL + one aggregate SOQL + one `update` per
  commit, regardless of how many lines reference the contract — safe at
  any cart size. The what-if simulators do zero SOQL when given
  `deposits[]` directly (pure math); `simulateForQuote` does one SOQL
  against QuoteLineItem.

## Questions testers ask

**"Why didn't my 8% discount show up?"** — Check, in order: is the
Commitment Contract Active and dated to cover today; is the Commitment
Discount Rule Active; does the rule's Min Commit Amount gate sit at or
below the contract's Commit Amount; is the line Recurring/Usage/Overage
(not OneTime, not a bundle parent, not hand-typed).

**"Where do I watch the prepaid credit balance drain as I add usage?"** —
Nowhere in this build. It's a what-if simulator you run on demand, not a
live cart balance. Ask Claude to run it for you with your real numbers.

**"I set the Covered Products on my floor contract and nothing changed on
the quote."** — Correct, nothing should. Covered Products only affects
what the what-if simulator counts toward the floor; the quote itself
never shows a floor line.

**"Why does the Account-level commitment floor never block or warn me on
the quote?"** — By design. Minimum-commitment enforcement is downstream
billing's job (`MAX(actualUsage, floor)` per period); DD CPQ only ever
captures the contract term.

**"Is `FLOOR_BLOCKED` the same floor as this module?"** — No. That error
code belongs to the Margin Floor feature (a minimum-margin guardrail on
discounts), unrelated to `CommitmentContract__c`.

**"Can one commitment discount rule apply to every product?"** — Yes:
leave `TargetProduct__c` and `Target__c` both blank, or set `Target__c` to
mode `All` via Rules Studio.

**"What happens if I set `MinimumCommitAmount__c` on a Recurring plan
instead of a Usage plan?"** — Nothing useful; `CommittedUsageMRR__c` is
computed independent of `RevenueNature__c`, but the field only has
meaning as a usage guardrail. Recurring plans already have a real
MRR/ARR; don't double up.

**"I committed a quote with a prepaid deposit line — where's the
drawdown recorded?"** — Nowhere automatically. The deposit QLI is an
ordinary committed line; running `credit/simulate` with `quoteId` set
reads it back (`RatePlan.IsPrepaidCredit__c = true`) as one deposit for
the what-if math. There's no live reconciliation against it.
