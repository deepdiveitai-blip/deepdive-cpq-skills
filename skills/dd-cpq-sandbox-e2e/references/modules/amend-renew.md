# Amendments, renewals and the install base

> Installed build 0.1.0-9 · Verified 2026-10-07 in cpq-pkg: increased AR Widget
> Pro from 50 to 70 seats mid-month (ProrationFactor 0.516129, MrrDelta
> $2,000, ProratedFirstPeriodAmount $1,032.26); renewed 40 seats with CPI 3.2%
>
> - Adder 2% = 5.2% uplift computed and stamped on the line, but **not**
>   reflected in NetPrice__c — the rules diagnostic itself reports the uplift
>   rule outcome as "shadowed" (see Troubleshooting).

## What it is for

An **amendment** changes what an existing customer has mid-term: more or
fewer seats, a swap, a cancellation. A **renewal** picks, asset by asset,
what carries forward onto a new contract when the old one ends, and can
raise the price with a CPI-style escalator. Both start from a **Baseline** —
a sealed snapshot of what the account has right now — so the engine always
knows the "before" to compare the rep's "after" against.

This is a bigger slice of the product than CLAUDE.md's rule 11 "cut" list
suggests. Rule 11 cuts **Constitutional CPQ** and **Agreement Studio** — a
different, unrelated subsystem. Amendments, renewals, install base, lifecycle
rules and restructure (merge/split/transfer/rewrite/undo) are all present and
exercised by passing Apex tests in 0.1.0-9: `AmendmentServiceTest`,
`RenewalServiceTest`, `RenewalPricingServiceTest`, `RestructureServiceTest`,
`InstallBaseApiTest`. The repo's own `docs/AMEND_GUIDE.md` still says
"Renewals are slice 2 and not built" — that line is stale; `RenewalService`,
`RenewalDrafter` and `RenewalPricingService` are all in this build (commit
31444bb and before — `RenewalService` landed in `1ee551a`, an ancestor of
the installed commit). Trust the verified behaviour below over that one line
of the doc.

## How it works

**The pipeline**: Install Base 360 → seal a Baseline → open an Amendment or
Renewal quote pre-populated from it → the rep edits quantities → the engine
prices every line exactly like new business (steps 6–11 of the canonical
pipeline run unchanged) → Commit → **Accept**, which diffs the quote against
the Baseline and writes one `InstallBaseChange__c` per changed line → in
Owner mode the change is copied onto the `Asset` at once, or immediately if
its effective date is today or earlier.

**Where it starts**: every install base begins with an **accepted
new-business quote**. Accepting (`Accept and create what they have` in the
cart, or `accept` on `/dd/v1/installbase`) turns each line into an `Add` row
in the change ledger and, for the `Native` provider, an `Asset` — bundle
options nest under their bundle's Asset. A quote accepts exactly once; a
second `accept` call is refused. An Amendment or Renewal quote that was not
started from Install Base 360 has no Baseline and is refused.

**Lots and position**: every `Add` or `Increase` is a _lot_ — its own
quantity, start date, end date, price. What the customer has on any date is
the ledger replayed to that date, not a single number read off the Asset. A
decrease is split across lots in `DecreaseOrder__c` order (`NewestFirst`
default, `OldestFirst`, `HighestPriceFirst`).

**Pricing an existing line**: `LifecycleRuleService.forLine(productId,
lineAction, context)` picks the winning `LifecycleRule__c` — product-specific
beats catch-all, lowest `Priority__c` wins among ties — from the one
`LifecycleMatrix__c` whose status is Active. Its `PriceBasis__c` decides:
`HoldPrior` (the default, and the default when no rule matches at all) keeps
today's price so an untouched line never moves because the catalog did;
`Reprice` prices the line fresh through the waterfall. A brand-new line on
an amendment is always priced at today's catalog, like new business.

**Proration**: `ProrationFactor__c` is remaining days / days in the
effective-date's calendar month for `Immediate` + `Daily` proration; `1` for
`Immediate` + `MonthlyRoundUp`/`None`; `0` for `Immediate` +
`MonthlyRoundDown`, `NextPeriod`, or `AtRenewal`. `ProratedFirstPeriodAmount__c`
= `MrrDelta__c` × `ProrationFactor__c` (negative is a credit). A change
dated before the line it changes actually started is clamped to that start
date and flagged `LIFECYCLE_EFFECTIVE_CLAMPED`.

**Drift**: before `accept` finalizes, the engine re-reads every provider.
If anything on the account moved since the Baseline was sealed, nothing is
written — the response carries `drift.lines` (product, when it started, now,
what changed) and the caller must resend with `acceptDrift: true`, or start a
fresh amendment.

**Renewal**: `RenewalService.start` reads every renewable line on the named
Contract(s) as of the day before the new term starts, lets the caller pick
asset by asset what renews (a bundle header decides for its whole bundle; a
component can be dropped but never renews without its header), seals that
subset as a new Baseline, and opens a Renewal quote. A line with no end date
reads as **evergreen** (unless its rate plan is marked `IsEvergreen__c`, or
its revenue nature is `OneTime`, both of which skip it as `oneTime`) and
never renews or expires — it simply continues. Folding a second contract in
early credits only its unused, already-paid time.

**Renewal uplift**: when a renewal line carries a prior net price (from its
Baseline line), `RenewalPricingService` computes `newPrice = priorNetPrice ×
(1 + MIN(CPI% + Adder%, MaxCap%)/100)` using the winning Active
`RenewalUpliftRule__c` (by criteria, then the product-specific rule beating a
catch-all) and the latest Active `CPIIndex__c` row for the currency on or
before today. It is gated purely on `Quote.TransactionType__c == 'Renewal'`
— nothing else — so it never fires on an Amendment quote even if the line
carries the same Baseline linkage.

## Objects and fields

| Object label (API name)                      | Field                                                                                                                                                                                                                                                                                                                                                                                                   | Meaning / values                                                                                      |
| -------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------- |
| Baseline (`Baseline__c`)                     | `Account__c`, `Status__c`                                                                                                                                                                                                                                                                                                                                                                               | One sealed snapshot of an account's install base at a point in time                                   |
| Baseline Line (`BaselineLine__c`)            | `Baseline__c`, `Product__c`, `Quantity__c`, `SourceAsset__c`, `RatePlan__c`, `Contract__c`, `ParentBaselineLine__c`, `LegacyComponent__c`                                                                                                                                                                                                                                                               | One line of what the account had when the Baseline was sealed; `UnitNetPrice__c` feeds renewal uplift |
| Quote                                        | `TransactionType__c` (New Business / Renewal / Amendment / Cancellation / Restructure), `Baseline__c`                                                                                                                                                                                                                                                                                                   | Deal-level fact; blank reads as New Business. Drives which cart view opens                            |
| QuoteLineItem                                | `BaselineQuantity__c`, `QuantityDelta__c`, `MrrDelta__c`, `LineAction__c` (New/Increase/Decrease/Unchanged/Swap/Renewed/Removed), `ProrationFactor__c`, `ProratedFirstPeriodAmount__c`, `ChangeTiming__c` (Immediate/NextPeriod/AtRenewal), `SourceAsset__c`, `PriorNetPrice__c`, `RenewalUpliftPct__c`, `RenewalUpliftBreakdown__c`                                                                    | Every amendment/renewal fact lives on the line, stamped by the engine, not typed by the rep           |
| Install Base Change (`InstallBaseChange__c`) | `ChangeType__c` (Add/Increase/Decrease/Swap/Renew/PriceChange/Remove), `QuantityDelta__c`, `QuantityAfter__c`, `MrrDelta__c`, `MrrAfter__c`, `ProratedAmount__c`, `EffectiveDate__c`, `Timing__c`, `AppliedStatus__c` (Pending/Applied/Rejected), `AppliedRef__c`, `RatePlan__c`, `Lot__c`, `ProjectedAt__c`, `ReversalOf__c`, `ParentChange__c`, `CounterpartChange__c`                                | One row per accepted change — the permanent ledger                                                    |
| Lifecycle Rule Set (`LifecycleMatrix__c`)    | `Status__c`, `EffectiveFrom__c`, `EffectiveTo__c`                                                                                                                                                                                                                                                                                                                                                       | One Active set whose dates cover today governs                                                        |
| Lifecycle Rule (`LifecycleRule__c`)          | `Matrix__c`, `AllowedAction__c` (Increase/Decrease/Cancel/Swap/Renew/Suspend/Resume/Rewrite/Merge/Transfer/Split/Expire), `IsAllowed__c`, `ProrationMethod__c` (Daily/MonthlyRoundUp/MonthlyRoundDown/None), `MinNoticeDays__c`, `MaxDecreasePercent__c`, `RequiresApproval__c`, `PriceBasis__c` (HoldPrior/Reprice), `DecreaseOrder__c`, `Priority__c`, `Product__c`, plus the shared 5-field criteria | What each lifecycle action is allowed to do and how it is priced                                      |
| Renewal Uplift Rule (`RenewalUpliftRule__c`) | `TargetProduct__c`, `Target__c`, `AdditionalUpliftPct__c`, `MaxCapPct__c`, `UseCPI__c`, `Priority__c`, `Status__c`, plus the shared 5-field criteria                                                                                                                                                                                                                                                    | The Adder% and MaxCap% a renewal escalates by                                                         |
| CPI Index (`CPIIndex__c`)                    | `Currency__c`, `EffectiveDate__c`, `IndexValue__c`, `Source__c`, `Locked__c`, `Status__c`                                                                                                                                                                                                                                                                                                               | Versioned inflation input; latest Active row on/before today, for the QLI currency, wins              |

## Build it (tester)

**Scenario**: increase an existing customer's seat count mid-term, then
renew their contract a year later with a price uplift.

1. Finish the golden path first (one accepted quote gives you an Asset and
   a Contract to work from). On the **Acme Corp** Account record, open the
   **DeepDive CPQ** app and go to the Account's **Install Base 360**
   related tab (or the `Account_Install_Base_Record_Page` Lightning page —
   other apps keep the standard Account page).
2. On the **What they have** tab, find the **CRM Suite Pro** group (or
   whichever row you want to change). Tick the row (or leave nothing
   ticked to scope the whole account) and click **Amend contract**
   (header level) or **Amend** on a single row.
3. In the **Amend Acme Corp** dialog: set **Effective date** (today or a
   future date), pick **Timing**:
   - **Immediately - prorate the first period by the lifecycle rule** (default)
   - **From the next billing period - no proration**
   - **At renewal - schedule it for the new term**

   Click **Create amendment quote**.

4. The new Quote opens in the DD CPQ cart on the **Amendment** view. Columns
   (field set `DD_CPQ_Cart_Amendment`): Product · **What they had** · **What
   they will have** (editable) · **What the line does** · **Quantity
   change** · **Net price** · **MRR change** · **Charged or credited this
   period** · **Timing** · **End date**.
5. Type the new quantity into **What they will have**. Watch **What the
   line does** switch to `Increase` or `Decrease` and the MRR change and
   proration amounts fill in.
6. **Commit**, then click **Accept amendment** below the grid (disabled
   until the commit is clean). If the account's record changed since the
   Baseline was sealed, the cart shows **The customer's record changed
   since this amendment started** with **Cancel** / **Accept anyway**.
7. Back on Install Base 360, check **Pending changes** — anything not
   applied by the `NativeAsset` applier waits here for **Mark applied** or
   **Reject** (a reason is required to reject).
8. To renew: on **Contracts**, click **Renew...** next to the contract.
   The renewal picker shows **Qty today**, **MRR**, a **Renew / Drop**
   toggle per asset, an editable **Qty on renewal**, and term months. Click
   **Continue**, then **Create renewal quote**.
9. The Renewal quote opens the same Amendment-style cart. Commit, then
   **Accept amendment** (the button is not renamed for a renewal).

## Check it (Claude)

```bash
sf data query -o dd-e2e -q "SELECT Id, DDCPQ__TransactionType__c, DDCPQ__Baseline__c, Status, ContractId FROM Quote WHERE Id='<quoteId>'"
sf data query -o dd-e2e -q "SELECT Id, Product2Id, Quantity, UnitPrice, DDCPQ__NetPrice__c, DDCPQ__BaselineQuantity__c, DDCPQ__LineAction__c, DDCPQ__QuantityDelta__c, DDCPQ__MrrDelta__c, DDCPQ__ProrationFactor__c, DDCPQ__ProratedFirstPeriodAmount__c, DDCPQ__ChangeTiming__c FROM QuoteLineItem WHERE QuoteId='<quoteId>'"
sf data query -o dd-e2e -q "SELECT Id, DDCPQ__ChangeType__c, DDCPQ__QuantityDelta__c, DDCPQ__QuantityAfter__c, DDCPQ__MrrDelta__c, DDCPQ__MrrAfter__c, DDCPQ__ProratedAmount__c, DDCPQ__EffectiveDate__c, DDCPQ__AppliedStatus__c, DDCPQ__AppliedRef__c, DDCPQ__ProjectedAt__c FROM DDCPQ__InstallBaseChange__c WHERE DDCPQ__Quote__c='<quoteId>'"
sf data query -o dd-e2e -q "SELECT Id, Quantity, Price, Status, UsageEndDate FROM Asset WHERE AccountId='<accountId>'"
sf data query -o dd-e2e -q "SELECT Id, DDCPQ__Status__c, DDCPQ__AllowedAction__c, DDCPQ__PriceBasis__c, DDCPQ__Product__c FROM DDCPQ__LifecycleRule__c"
sf data query -o dd-e2e -q "SELECT Id, DDCPQ__Status__c, DDCPQ__AdditionalUpliftPct__c, DDCPQ__MaxCapPct__c, DDCPQ__UseCPI__c, DDCPQ__TargetProduct__c FROM DDCPQ__RenewalUpliftRule__c"
```

Expect: `AppliedStatus__c = 'Applied'` with `ProjectedAt__c` **blank** when
the change's `EffectiveDate__c` is in the future — the Asset itself will not
show the new quantity until that date (or an explicit **Re-project Assets**)
even though the status already reads Applied. Check `ProjectedAt__c`, not
just `AppliedStatus__c`, before telling a tester the Asset is wrong.

A read-only preview (never call `accept` from here — the tester commits and
accepts, not Claude):

```json
POST services/apexrest/DDCPQ/dd/v1/price?includeWaterfall=true
{
  "quoteId": "<amendment or renewal quoteId>",
  "mode": "reconcile",
  "wholeQuote": true,
  "selections": [
    { "localKey": "p", "productId": "<id>", "quantity": 70,
      "sourceAssetId": "<02i…>", "baselineLineId": "<a01…>" }
  ]
}
```

## Expected numbers

All from a real run in cpq-pkg, account "AR Test Co", product "AR Widget
Pro" ($100/unit, PerUnit, Monthly, Recurring).

| Step                                                                             | Field                              | Value                                                                                                      |
| -------------------------------------------------------------------------------- | ---------------------------------- | ---------------------------------------------------------------------------------------------------------- |
| New business: 50 units                                                           | NetPrice__c / ARR                  | 100 / $60,000 ($5,000 MRR × 12)                                                                            |
| Accept                                                                           | Asset Quantity / Price / Status    | 50 / 100 / Purchased                                                                                       |
| Amendment, effective 16 Oct 2026 (31-day October), 50 → 70                       | LineAction__c                      | Increase                                                                                                   |
| same                                                                             | QuantityDelta__c                   | 20                                                                                                         |
| same                                                                             | MrrDelta__c                        | 2,000.00 (20 × 100)                                                                                        |
| same                                                                             | ProrationFactor__c                 | 0.516129 — (31 − 16 + 1) / 31                                                                              |
| same                                                                             | ProratedFirstPeriodAmount__c       | 1,032.26                                                                                                   |
| same, NetPrice__c (PerUnit, HoldPrior)                                           | NetPrice__c                        | 100 (unchanged — unit price held, not repriced)                                                            |
| Accept amendment                                                                 | InstallBaseChange__c.ChangeType__c | Increase, QuantityAfter 70, MrrAfter 7,000.00, AppliedStatus Applied, ProjectedAt **blank** (future-dated) |
| New business #2: 40 units, 12-month term                                         | Contract StartDate/EndDate         | 2026-10-07 / 2027-10-06                                                                                    |
| Renew that contract, no overrides                                                | Renewal quote StartDate / term     | 2027-10-07 / 12 months                                                                                     |
| same                                                                             | renewing[].mrr (dry run)           | 4,000.00                                                                                                   |
| same, with a RenewalUpliftRule (Adder 2%, MaxCap 8%, UseCPI) + CPIIndex 3.2% USD | RenewalUpliftPct__c / Breakdown    | 5.20 / "CPI 3.20% + Adder 2.00% = 5.20% (AR test index)"                                                   |
| same                                                                             | NetPrice__c on the renewed line    | 100 (uplift stamped but **not** applied — see below)                                                       |

## Troubleshooting

| Symptom                                                                                                                                 | Cause                                                                                                                                                                                                                                                                                                                                                                                                              | Fix                                                                                                                                     |
| --------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ | --------------------------------------------------------------------------------------------------------------------------------------- |
| `"Nothing is picked to renew. To let everything lapse, do nothing: lines end with their contract."`                                     | Every renewable asset was excluded, or every line read as evergreen (no `TermMonths__c` ever set, so no end date)                                                                                                                                                                                                                                                                                                  | Confirm the original line was sold with a term; an evergreen line is correct to skip — it is not supposed to renew                      |
| Renewal picker shows a product's row as **Follows its bundle**                                                                          | It is a bundle option; the header's Renew/Drop decision carries its options                                                                                                                                                                                                                                                                                                                                        | Toggle the bundle header, not the option row                                                                                            |
| Renewal picker's **Continue** button stays disabled                                                                                     | `nothingRenews` — nothing is marked to renew yet                                                                                                                                                                                                                                                                                                                                                                   | Toggle at least one asset's segmented control to Renew                                                                                  |
| "A Renewal or Amendment quote that was not started from Install Base 360 is refused"                                                    | The quote has no `Baseline__c`                                                                                                                                                                                                                                                                                                                                                                                     | Always start from Install Base 360's **Amend** / **Renew...**, never a plain New Quote with TransactionType hand-set                    |
| `AppliedStatus__c = 'Applied'` but the Asset's Quantity did not change                                                                  | `EffectiveDate__c` is in the future; `ProjectedAt__c` is blank                                                                                                                                                                                                                                                                                                                                                     | Expected — the daily `AssetProjectionSchedulable` job (02:00) or **Re-project Assets** copies it onto the Asset on its date, not before |
| Renewal line shows a correct `RenewalUpliftPct__c` and `RenewalUpliftBreakdown__c`, but `NetPrice__c` did not move from the prior price | The lifecycle's `PriceBasis__c` (default `HoldPrior`, or an explicit `Reprice` rule) re-pins the price to the Baseline's prior price **after** `RenewalPricingService` computed the uplift — the rules diagnostic reports the uplift rule's outcome as `"shadowed"`. This reproduces even with an explicit Active `LifecycleRule__c` (`AllowedAction__c = Renew`, `PriceBasis__c = Reprice`) targeting the product | Known limitation in 0.1.0-9 — report it; do not assume a config mistake                                                                 |
| Cart's accept button on a Renewal quote says **Accept amendment**, not "Accept renewal"                                                 | `isAmendment` in `cpqLedger.js` treats cartMode `amendment` and `renewal` identically                                                                                                                                                                                                                                                                                                                              | Cosmetic; the underlying call (`AmendmentService.accept`) is correct for both                                                           |
| Accept refused a second time with no clear message                                                                                      | A quote accepts exactly once                                                                                                                                                                                                                                                                                                                                                                                       | Start a new Amendment/Renewal from Install Base 360; it builds from a fresh Baseline overlaid with anything still Pending               |
| The cart shows **The customer's record changed since this amendment started**                                                           | Drift — something on the account moved since the Baseline was sealed                                                                                                                                                                                                                                                                                                                                               | Review the drift rows; **Accept anyway** to proceed as quoted, or **Cancel** and start a fresh amendment from current data              |

## Limits and gotchas

- Renewal uplift is gated only on `Quote.TransactionType__c == 'Renewal'` —
  not on the line's `LineAction__c` or anything else — so every renewable
  line with a prior Baseline price gets evaluated against Active
  `RenewalUpliftRule__c` rows, even bundle children (bundle parents are
  explicitly skipped, as are OneTime/Usage/Overage charge types).
- The uplift computing-but-not-charging behaviour above means a renewal
  quote's **ARR total today reflects the held/reprice price, not the CPI
  escalator**, even though the chip/field data looks right. Verify the
  actual `NetPrice__c`, not just the uplift fields, before trusting a
  renewal number.
- Only two install base providers read anything in this build: `Native`
  (the ledger, replayed) and `Asset` (any Asset as it stands, lower
  confidence). `RevenueCloud`, `LegacySBQQ`, `ServiceContract`,
  `ExternalRest` and `Import` provider rows exist on the metadata but are
  not implemented — each contributes zero lines and is reported as a
  finding.
- Governor-limit shape: `RenewalService.start` and `AmendmentService.start`
  each do their SOQL in bulk collections (BaselineLine, RatePlan, Contract)
  before any DML; a renewal that folds in several contracts is still one
  pass. Avoid scripting a loop of many single-contract renew calls back to
  back in the same transaction.
- `docs/AMEND_GUIDE.md`'s own "Known limits" section says "Renewals are
  slice 2 and not built" — that line predates `RenewalService` landing and
  is stale in this checkout; everything above it in the doc about
  `RenewalPricingService` / `RenewalUpliftRule__c` is current and matches
  what was verified here.
- CLAUDE.md rule 11's "cut" list (Constitutional CPQ, Agreement Studio,
  Amendments [constitutional], Migrator, DocRaptor, Deal Scoring, runtime AI
  services) is about a different, retired policy-engine subsystem, not this
  lifecycle feature — do not read it as "amend/renew isn't built."
- A line with `TermMonths__c` never set never gets a `UsageEndDate` on its
  Asset, reads as evergreen, and is excluded from both expiry and renewal —
  by design, not a bug, but easy to mistake for one when testing.

## Questions testers ask

**Q: I changed a quantity but Net Price didn't move — is that a bug?**
No. `PriceBasis__c` defaults to `HoldPrior`: an untouched or quantity-changed
PerUnit/Tiered/Volume line keeps today's price. Only a `Package` line, or a
line whose `LifecycleRule__c` says `Reprice`, moves.

**Q: Why does the renewal uplift percentage show on the line but the price
is the same as before?**
Confirmed limitation in 0.1.0-9 — see Troubleshooting. The percentage and
breakdown are computed and stored correctly; the net price is not updated to
match.

**Q: Can I amend a quote I haven't accepted yet?**
No — amend and renew both start from an **accepted** quote's install base.
A draft quote has nothing in the ledger to amend against.

**Q: What happens to options when I amend their bundle?**
The header decides. Dropping a bundle header in the renewal picker drops its
options with it; an option can never renew without its header.

**Q: Why is a product I sold with no end date never showing up to renew?**
It reads as evergreen (no `TermMonths__c`, no `IsEvergreen__c` rate plan
flag, not OneTime) — evergreen lines never renew and never expire; they just
continue.

**Q: I rejected a pending change — can I undo that?**
Not from Install Base 360. `AppliedStatus__c` becomes `Rejected` with your
note; there's no reversal action on this screen. A fresh amendment is the
way forward.

**Q: Does accepting an amendment touch Orders or Contracts directly?**
No — it writes `InstallBaseChange__c` rows and, in Owner mode, updates
`Asset`. CLAUDE.md rule 8 still holds: native Sync-to-Order handles
Order/OrderItem/Asset/Contract creation from the Quote; DD CPQ never creates
them directly.
