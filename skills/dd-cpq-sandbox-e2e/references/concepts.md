# How DD CPQ thinks

> Installed build 0.1.0-9 · Verified 2026-10-07 in cpq-pkg: priced "CN Concepts
> Widget" qty 100, Account Industry Healthcare, Opportunity Type Renewal, a
> $130 price rule, 5%+3% system discounts (AllStack), 15% volume tier at 100+
> → waterfall ran PricebookList(150) → ContractPrice(150, skip) →
> DerivedListPrice(130, rule) → RampAdjustedPrice(130, skip) →
> TermAdjustedPrice(130, skip) → UsageTierPrice(130, skip) →
> PricingModel(130, skip) → SystemDiscount(119.795) →
> PromotionDiscount(119.795, skip) → ChannelDiscount(119.795, skip) →
> VolumeDiscount(101.82575) → ManualDiscount(skip) →
> CommitDiscount(skip) → Net **$101.83**. ARR **$122,196**. Matches the
> golden quote exactly.

This is the one module every other module links to. Read it once, before any
feature module, so "pipeline stage", "waterfall", "rule set", "criteria" and
"RuleTarget" mean the same thing everywhere else in this skill.

## What it is for

DD CPQ is a server-side pricing and configuration engine for Salesforce
Quotes. A rep (or a bot) picks products in a cart; DD CPQ decides, in Apex,
what each line should cost and whether it is allowed at all — eligibility,
compatibility, list price, every discount, the final net. The cart is a
client. It renders what the engine returns and sends back what the rep
clicked. It never computes a price itself.

Every click in the cart ends up as one of two Apex entry points on
`CpqEngine`:

- `previewDrafts(request)` — prices a set of selections without writing
  anything. The cart calls this on every add, every quantity change, every
  discount typed. This is also what you call read-only through REST at
  `dd/v1/price` when you check a number from the CLI.
- `runPipeline(request)` — calls `previewDrafts` internally, then writes
  standard `QuoteLineItem` rows (parent + children via `ParentLine__c`) and
  an `AuditLog__c` row. This is Commit. The tester clicks it; you never call
  `dd/v1/commit`.

## The engine pipeline, in the order the installed code runs it

This is `CpqEngine.previewDraftsInRun` at commit 31444bb, collapsed to what a
tester actually sees on screen. Every stage below runs **inside Apex**; none
of it is JavaScript.

| #   | What happens in Apex                                                                                                                               | What the tester sees                                                                                                        |
| --- | -------------------------------------------------------------------------------------------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------- |
| 1   | Open the quote, load Account/Opportunity/User context                                                                                              | Clicks **Configure Products** on the Quote; the DD CPQ Cart opens                                                           |
| 2   | Load existing lines if this is Edit Lines                                                                                                          | Cart shows what was already committed                                                                                       |
| 3   | Filter the Pricebook entry + currency for each product                                                                                             | Add Products list only shows priced, active products                                                                        |
| 4   | (Guided Selling, optional, not covered here)                                                                                                       | —                                                                                                                           |
| 5   | **Eligibility** — `EligibilityService.filterEligible`: first matching Active `EligibilityRule__c` by priority wins; no match = eligible by default | A hidden product never appears in Add Products; one added another way (REST, an old quote) is flagged and refused at commit |
| 6   | **Selection** — the rep's picks become `CpqEngine.Selection` rows, bundle options expand, quantities resolve                                       | Products and bundle options appear in the cart grid                                                                         |
| 7   | **Compatibility** — `OptionConstraint__c` / `CompactabilityRule__c`: Requires / Excludes / MaxQty judged on every selection change                 | A red banner on the line the moment an excluded pair, a missing requirement, or a quantity over the cap appears             |
| 8   | **Pricing** — the waterfall below (steps 1–13 of it)                                                                                               | The price column updates live as the rep edits quantity or clicks the price to open the waterfall drawer                    |
| 9   | System + Volume discounts (part of the waterfall, multiplicative stack)                                                                            | Same live update                                                                                                            |
| 10  | Manual override + margin-floor approval if under floor                                                                                             | Rep types a % off, $ off, or a price; a floor breach shows "needs approval" but still lets the quote save                   |
| 11  | **Commit** — `runPipeline`: writes `QuoteLineItem` rows                                                                                            | Rep clicks **Commit**; the cart closes or shows the committed lines                                                         |
| 12  | **Audit** — `AuditService.logFinalize` writes `AuditLog__c`                                                                                        | Not visible in the cart; query `AuditLog__c` to see it                                                                      |

Do not reorder this in your head when troubleshooting: eligibility and
compatibility are judged before pricing runs, pricing runs before discounts,
and discounts run before the manual override and the margin floor — which is
the last word on price, enforced again at commit even if the cart let the
rep save a floor breach.

## The price waterfall — every stage this build can produce, in order

Every `WaterfallStage` the installed `CpqEngine` can add to
`QuoteLineItem.WaterfallJson__c`, in the order it runs. A stage that cannot
fire for this line (a OneTime charge hitting `RampAdjustedPrice`, for
example) still appears, marked "not-applicable" or with a skip note — it is
a pass-through, not a missing row. Click the price on any cart line to open
this list; it is also what `?includeWaterfall=true` on `dd/v1/price`
returns per line.

| Stage name (exact, as shown) | Fed by                                                                                                                                                                    | Always shown?                                                       |
| ---------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------- |
| `PricebookList`              | the `PricebookEntry.UnitPrice` for this product+currency                                                                                                                  | Always                                                              |
| `ContractPrice`              | `ContractPriceRule__c` — a negotiated override for this Account+Product                                                                                                   | Always (skip note if none)                                          |
| `DerivedListPrice`           | `PricingRule__c` — Absolute / PctOfList / Adjustment / Variable, first match by priority                                                                                  | Always (falls back to list)                                         |
| `RampAdjustedPrice`          | `RampMatrix__c`/`RampSchedule__c` term-averaged rate, or per-year rate for a ramped line                                                                                  | Always (skip note if no ramp)                                       |
| `TermAdjustedPrice`          | `TermDiscountCurve__c`/`TermDiscountPoint__c` for this line's `TermMonths__c`                                                                                             | Always (skip note; "One-time charge" note on OneTime lines)         |
| `UsageTierPrice`             | `UsageTierMatrix__c`/`UsageTier__c` Volume/Graduated bracket + MinCommit + Overage                                                                                        | Always (skip note if no tiers)                                      |
| `PricingModel`               | `PricingModelService` — FlatFee/PerUnit/Tiered/Volume/Package/PercentOfBasis math                                                                                         | Always                                                              |
| `SystemDiscount`             | `SystemDiscountMatrix__c`/`SystemDiscountRule__c`, stacked per the matrix's `StackingStrategy__c`                                                                         | Always                                                              |
| `PromotionDiscount`          | `PromotionRule__c`, matched on `Quote.PromotionCode__c` or the request's `promoCode`                                                                                      | Always (skip note: no code, code didn't match, or promo exhausted)  |
| `ChannelDiscount`            | `ChannelDiscountRule__c`, matched on partner tier                                                                                                                         | Always (skip note: direct deal, or no tier match)                   |
| `VolumeDiscount`             | `VolumeDiscountTier__c` bracket for this line's quantity                                                                                                                  | Always                                                              |
| `ManualDiscount`             | the rep's typed % off, $ off, or (see `SalesPriceOverride` below) a typed price                                                                                           | Always ("No manual discount" when none)                             |
| `SalesPriceOverride`         | a rep-typed price — when present it REPLACES the net price and every stage above/below from `SystemDiscount` through `ManualDiscount` shows "Skipped — price set by hand" | Only when the rep set a price by hand                               |
| `BlockDiscount`              | a Deal Block's own discount, after the manual concession                                                                                                                  | Only on a line inside a Deal Block that has a discount              |
| `CommitDiscount`             | `CommitmentDiscountRule__c` against an Account's active `CommitmentContract__c`                                                                                           | Always (skip note if no active commit)                              |
| `PaymentTermDiscount`        | the winning `RatePlan__c.PaymentTermDiscountPct__c` for this line's payment terms                                                                                         | Only emitted when the plan names a discount %                       |
| `MarginFloor`                | `MarginFloorRule__c` against `QuoteLineItem.Cost__c` — clamps net up, or flags approval                                                                                   | Only emitted when a floor rule evaluated (needs a Cost on the line) |
| `Net`                        | the final per-unit price written to `QuoteLineItem.NetPrice__c`                                                                                                           | Always — always the LAST stage                                      |

Two things the gap-review history baked into this build, worth knowing when
a number looks "off by one stage":

- **The discount base is the first non-null of** `UsageTierPrice` →
  `TermAdjustedPrice` → `RampAdjustedPrice` → `DerivedListPrice`. A line with
  no usage tier, no term curve and no ramp discounts straight off
  `DerivedListPrice` — which is exactly the golden quote's path.
- **The discount stack is multiplicative, not additive.** 5% + 3% is not
  8% — it is `1 − (0.95 × 0.97) = 7.85%`. `QuoteLineItem.SystemDiscount__c`
  stores that combined 7.85, not the two inputs.
- Not-in-this-build: DDCPQ-69 (the waterfall naming the rule behind each
  step in friendlier text) and DDCPQ-112 (discount % shown to more decimal
  places) are both trunk-only. The rule Id is there (`sourceRuleId`); the
  prose note is the older wording.

## Rule sets vs rules

Six areas use the same two-level shape: a **matrix** (the rule set,
master) and its **rule rows** (detail, master-detail to the matrix). The
matrix carries `Status__c` — **Draft** (default on save), **Active**
(the only status the engine reads when pricing), **Inactive** (switched off
without deleting). A rule set saved in Rules Studio starts Draft; you must
switch it to Active yourself.

| Rule set (matrix) label                                                                                   | Rule row label                                 | What it decides                       |
| --------------------------------------------------------------------------------------------------------- | ---------------------------------------------- | ------------------------------------- |
| Price Rule Set (`PricingMatrix__c`)                                                                       | Pricing Rule (`PricingRule__c`)                | `DerivedListPrice`                    |
| Eligibility Rule Set (`EligibilityMatrix__c`)                                                             | Eligibility Rule (`EligibilityRule__c`)        | Who may buy what                      |
| Compatibility Rule Set (`CompactabilityMatrix__c`) — yes, that is the real spelling, see CLAUDE.md rule 4 | Compatibility Rule (`CompactabilityRule__c`)   | Requires / Excludes / MaxQty          |
| Discount Rule Set (`SystemDiscountMatrix__c`)                                                             | System Discount Rule (`SystemDiscountRule__c`) | `SystemDiscount`                      |
| Lifecycle Rule Set (`LifecycleMatrix__c`)                                                                 | Lifecycle Rule (`LifecycleRule__c`)            | Amend/renew policy against an `Asset` |
| Ramp Rule Set (`RampMatrix__c`)                                                                           | Ramp Schedule (`RampSchedule__c`)              | `RampAdjustedPrice`                   |

Volume (`VolumeDiscountMatrix__c` → `VolumeDiscountTier__c`) and Usage
(`UsageTierMatrix__c` → `UsageTier__c`) follow the same master-detail shape
but key off quantity brackets (`MinQuantity__c`/`MaxQuantity__c`), not the
5-field criteria schema below — a volume tier has no "Account.Industry
equals X" row of its own; the matrix header plus `TargetProduct__c` is as
conditional as it gets. **An Active Volume/Usage matrix requires a
`TargetProduct__c`** — the engine refuses to activate one with no product
(confirmed in cpq-pkg: "Target Product is required on Active
volume-discount matrices").

### The shared 5-field criteria schema (CLAUDE.md rule 5)

One Apex class, `CriteriaEvaluator`, judges every rule row above (and
Promotion/Channel/Commitment/RenewalUplift rules, which reuse the same
fields without a matrix parent). Empty `FieldApiName__c` or `Operator__c` =
**catch-all, matches everything**.

| Field             | Type        | Notes                                                                                                                                                                                             |
| ----------------- | ----------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `FieldApiName__c` | Text        | Dotted path, e.g. `Account.Industry`, `Line.Quantity`                                                                                                                                             |
| `Operator__c`     | Picklist    | every value installed: `equals`, `not-equals`, `in`, `not-in`, `greater`, `less`, `greater-or-equal`, `less-or-equal`, `between`, `contains`, `starts-with`, `includes`, `is-null`, `is-not-null` |
| `Value__c`        | Text        | Comma-separated for `in`/`not-in`/`between`/`includes`; can reference a `Variable__c`                                                                                                             |
| `ValueType__c`    | Picklist    | `string`, `number`, `picklist`, `date`, `boolean`, `id`                                                                                                                                           |
| `Priority__c`     | Number(3,0) | Evaluation order, low → high; lower wins on a tie                                                                                                                                                 |

A rule that cannot be evaluated (bad literal, unreadable field) is treated
as **false**, never as an error — one broken rule cannot stop a quote from
pricing.

### `Conditions__c` — several conditions with logic

When a rule needs more than one condition, Rules Studio writes
`Conditions__c` instead of the five single fields, as JSON:

```json
{
  "logic": "1 AND (2 OR 3)",
  "conditions": [
    {
      "n": 1,
      "field": "Account.Industry",
      "op": "equals",
      "value": "Healthcare",
      "type": "string"
    },
    {
      "n": 2,
      "field": "Opportunity.Type",
      "op": "equals",
      "value": "Renewal",
      "type": "string"
    },
    {
      "n": 3,
      "field": "Opportunity.Type",
      "op": "equals",
      "value": "New Business",
      "type": "string"
    }
  ]
}
```

When `Conditions__c` is filled it **replaces** the single
`FieldApiName__c`/`Operator__c`/`Value__c` on that row — they are ignored.

### What field paths are allowed

Not just the six legacy fields — any readable field under these roots,
loaded once per transaction and validated hop-by-hop against the describe
so a rule can never read more than the pricing user can:

`Account.*` · `Opportunity.*` · `Quote.*` · `User.*` · `Product.*` (the
line's product) · `Variable.*` (a resolved `Variable__c`) · `Line.*` — the
line itself: `Quantity`, `TermMonths`, `BillingFrequency`, `LineAction`,
`ChargeType`, `PricingModel`, `ListPrice`, `NetPrice`, `Cost`,
`EffectiveDate`, `EndDate` · `Block.*` — the line's Deal Block: `Kind`,
`Name`, `Location`, `TermMonths`, `IsOptional`, `ParentKind`.

### First-match vs all-match, per rule type

| Rule type                                                     | Behaviour                                                                                                                                                                       |
| ------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Pricing (`PricingRule__c`)                                    | First Active match by priority wins (product-named rule beats a field/group match at equal priority); no match falls back to Pricebook list                                     |
| Eligibility (`EligibilityRule__c`)                            | First Active match by priority wins; **no match = eligible** (default allow — admins exclude, they do not allow)                                                                |
| Compatibility (`CompactabilityRule__c`/`OptionConstraint__c`) | Every matching Require/Exclude/MaxQty rule is enforced independently — not first-match; a line can carry more than one violation                                                |
| System Discount (`SystemDiscountRule__c`)                     | Every matching rule under one matrix's `StackingStrategy__c`: `AllStack` multiplies every match, `TakeBest` keeps only the largest                                              |
| Volume (`VolumeDiscountTier__c`)                              | The one bracket whose `MinQuantity__c`/`MaxQuantity__c` contains the line's quantity                                                                                            |
| Promotion / Channel                                           | One winning rule is picked (`PromotionService.pickWinningRule`), then its own discount applies; channel rules of the same tier can still stack with each other multiplicatively |

### Targeting: `TargetProduct__c` vs `Target__c`

Every rule row has a simple `TargetProduct__c` lookup to `Product2` — "the
product whose price/eligibility/etc. this row decides" — kept for backward
compatibility and the common case. Rules Studio's richer targeting lives in
`Target__c` (a JSON LongTextArea, label "Products Covered"):

```json
{"mode":"All|Products|Criteria|Group","products":["01t..."],
 "criteria":{"logic":"...","conditions":[...]},"groupId":"a0...","exclude":["01t..."]}
```

Blank `Target__c` means "the single `TargetProduct__c` field decides, as
before." `RuleTarget.keepCoveringMostSpecificFirst` (seen in
`PricingService.derive`) is what makes a rule naming the exact product beat
one that covers it only by field criteria or group membership, before
`Priority__c` breaks further ties.

## Object map

Every custom object in the installed package, grouped by area. Every
`__c` carries `DDCPQ__` in a query against cpq-pkg; this table uses the
unprefixed name, per CLAUDE.md rule 1.

**Catalog, products, rate plans**

| Object                             | Purpose                                                                                                                                                                                                                    |
| ---------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `Catalog__c` / `CatalogProduct__c` | A curated product set a rep picks, independent of pricing                                                                                                                                                                  |
| `ProductFeature__c`                | A section inside a bundle ("Core", "Storage")                                                                                                                                                                              |
| `ProductOption__c`                 | Joins a bundle parent to one child product inside a feature                                                                                                                                                                |
| `ProductGroup__c`                  | A named group of products a rule can target as one unit                                                                                                                                                                    |
| `ProductTier__c`                   | An ordinal rank on a product so "tier 2 or higher" is arithmetic, not string matching                                                                                                                                      |
| `RatePlan__c`                      | The pricing model for one product: `RevenueNature__c` (Recurring/OneTime/Usage/Overage), `PricingModel__c` (FlatFee/PerUnit/Tiered/Volume/Package/PercentOfBasis), `BillingSchedule__c`, `UnitPrice__c`, `PlanGroupKey__c` |
| `ProductChargeProfile__c`          | One charge line a product can fan out into (base + usage + overage on one hybrid SKU)                                                                                                                                      |

**Rules**

| Object                                                         | Purpose                                                                                    |
| -------------------------------------------------------------- | ------------------------------------------------------------------------------------------ |
| `PricingMatrix__c` / `PricingRule__c`                          | Sets `DerivedListPrice`                                                                    |
| `EligibilityMatrix__c` / `EligibilityRule__c`                  | Who may buy what                                                                           |
| `CompactabilityMatrix__c` / `CompactabilityRule__c`            | Requires / Excludes / MaxQty between products                                              |
| `SystemDiscountMatrix__c` / `SystemDiscountRule__c`            | Standing entitlement discounts                                                             |
| `VolumeDiscountMatrix__c` / `VolumeDiscountTier__c`            | Quantity-break discounts                                                                   |
| `UsageTierMatrix__c` / `UsageTier__c`                          | Consumption brackets (Volume/Graduated)                                                    |
| `PromotionRule__c` / `PromotionRedemption__c`                  | Promo-code discounts and their usage ledger                                                |
| `ChannelDiscountRule__c`                                       | Partner-tier discount                                                                      |
| `MarginFloorRule__c`                                           | The last-word floor against `Cost__c`                                                      |
| `ContractPriceRule__c`                                         | Negotiated Account+Product override, read before `DerivedListPrice`                        |
| `CommitmentContract__c` / `CommitmentDiscountRule__c`          | Prepaid commitment + its locked rate                                                       |
| `RenewalUpliftRule__c` / `CPIIndex__c`                         | Renewal-quote price uplift and the CPI table it reads                                      |
| `TermDiscountCurve__c` / `TermDiscountPoint__c`                | Whole-contract-term discount curve                                                         |
| `RampMatrix__c` / `RampSchedule__c`                            | Per-period ramp pricing                                                                    |
| `LifecycleMatrix__c` / `LifecycleRule__c`                      | Amend/renew policy against an `Asset`                                                      |
| `OptionConstraint__c`                                          | Require/Exclude between two `ProductOption__c` rows in the same bundle                     |
| `QuoteLineRequirement__c` / `QuoteLineRequirementCompanion__c` | Cross-line "if X then at least one of Y" rule                                              |
| `Variable__c` / `VariableFilter__c`                            | Reusable resolved value (Aggregate/Formula/ApexMethod/Constant) a rule's RHS can reference |

**Bundles**

| Object                | Purpose                                                                                  |
| --------------------- | ---------------------------------------------------------------------------------------- |
| `BundleDefinition__c` | Declares a `Product2` as a bundle header and how it is priced (roll-up vs header-priced) |

**Quoting and deal structure**

| Object                                  | Purpose                                                                |
| --------------------------------------- | ---------------------------------------------------------------------- |
| `DealBlock__c` / `DealBlockTemplate__c` | A named group of lines with its own window/discount inside a quote     |
| `OrderItemRatePlan__c`                  | Links a committed `OrderItem` back to the `RatePlan__c` that priced it |

**Lifecycle / install base**

| Object                            | Purpose                                                        |
| --------------------------------- | -------------------------------------------------------------- |
| `Baseline__c` / `BaselineLine__c` | A snapshot of a customer's owned lines, for amend/renew deltas |
| `InstallBaseChange__c`            | One recorded change against the install base                   |

**Admin, audit, telemetry**

| Object                                         | Purpose                                                                            |
| ---------------------------------------------- | ---------------------------------------------------------------------------------- |
| `Setting__c`                                   | DD CPQ Settings (org-wide switches — do not edit without explicit task scope)      |
| `UserPreference__c`                            | Per-user cart preferences                                                          |
| `AuditLog__c`                                  | Append-only log of overrides, matrix activations, finalize events                  |
| `RuleHistory__c`                               | Append-only version history for rule/matrix records (Create/Update/Delete/Restore) |
| `RuleStat__c` / `EngineRun__c` / `ErrorLog__c` | Engine Health telemetry — rule hit counts, run cost per stage, errors              |

## Custom fields that matter — QuoteLineItem

In the installed build, `QuoteLineItem` is the standard object that
carries the pricing math. The fields a tester will actually look at:

| Field                                                                                                                                    | Meaning                                                                          |
| ---------------------------------------------------------------------------------------------------------------------------------------- | -------------------------------------------------------------------------------- |
| `DerivedListPrice__c`                                                                                                                    | List price after `PricingRule__c`, before any discount                           |
| `NetPrice__c`                                                                                                                            | The final per-unit price — the `Net` waterfall stage                             |
| `MRR__c` / `ARR__c`                                                                                                                      | Monthly / annual recurring revenue for this line                                 |
| `TermMonths__c` / `BillingFrequency__c`                                                                                                  | The line's own term and invoice cadence                                          |
| `ParentLine__c`                                                                                                                          | Points at the bundle parent `QuoteLineItem`; null on a standalone or parent line |
| `WaterfallJson__c`                                                                                                                       | The full stage-by-stage waterfall, exactly what REST returns                     |
| `SystemDiscount__c` / `VolumeDiscount__c` / `PromotionDiscount__c` / `ChannelDiscount__c` / `ManualDiscount__c` / `CommitDiscountPct__c` | The effective % each stage applied                                               |
| `Cost__c`                                                                                                                                | Unit cost, read by `MarginFloorRule__c`                                          |
| `SalesPriceOverride__c`                                                                                                                  | A rep-typed price; when set it IS the net price                                  |
| `RatePlan__c`                                                                                                                            | The winning `RatePlan__c` for this line                                          |
| `SourceBundle__c`                                                                                                                        | The bundle product Id this child line came from                                  |
| `EffectiveDate__c` / `EndDate__c`                                                                                                        | The line's own window                                                            |
| `BlockedByFloor__c` / `FloorApprovalRequired__c`                                                                                         | Margin-floor outcome                                                             |

## Custom fields that matter — Quote

The installed build also adds deal-level header fields to Quote:
`StartDate__c`, `TermMonths__c`, `BillingFrequency__c`, `CoTermMode__c`,
`PromotionCode__c`, `TransactionType__c`, `Baseline__c`, `SourceAsset__c`.

## Permission model

| Permission set        | Custom permissions it grants          | What it's for                                                                                                                      |
| --------------------- | ------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------- |
| `DD_CPQ_Engine_Admin` | `Administer_Pricing` + `Build_Quotes` | Configure rate plans, catalogs, rules, variables — plus everything a quoting user can do. No Modify All; sharing rules still apply |
| `DD_CPQ_Engine_User`  | `Build_Quotes` only                   | Price and commit quotes; reads pricing config but cannot change it                                                                 |

DD CPQ checks these two **custom permissions** itself, in Apex, independent
of profile and object permissions — **even a System Administrator** is
refused with "Working on a quote needs the "Build Quotes" permission"
without one of these permission sets assigned. There is no third tier;
`Administer_Pricing` implies `Build_Quotes`.

The app is **DeepDive CPQ** (App Launcher). Its tabs: `DD_CPQ_Settings`,
standard Quote/Opportunity/Account/Product2/Pricebook2, `DD_CPQ_Cart`,
`Product_360`, `Rate_Plan_Editor`, `Bundle_Builder`, `Rules_Studio`,
`Engine_Health`, plus a direct tab for every rule/matrix object in the Rules
section above.

**How the cart opens**: the package ships a Quick Action
("Configure Products") that an admin must drag onto the Quote page layout
themselves (it cannot ship pre-placed on a layout it doesn't own). Clicking
it calls `CpqCartController.getCartSurface` and either navigates to the
`DD CPQ Cart` tab with the quote Id in page state (`c__quoteId`) — the
default, "page" surface — or opens the cart as a modal dialog over the
record ("popup" surface, an admin's choice per view). Either way the cart
receives the same quote Id; nothing about pricing differs between the two.

## What DD CPQ deliberately does not do

- **No Order, OrderItem, Asset or Contract creation.** Salesforce's native
  Sync-to-Order/Contract turns a committed Quote into those; DD CPQ writes
  only `QuoteLineItem` and reads the rest.
- **No AI in the runtime quoting path.** Rule authoring in English happens
  at design time (Rules Studio's "Ask Claude" style helpers); the engine
  that actually prices a quote executes stored deterministic rules only.
  Nothing between Add Product / Edit Lines / Commit makes an `HttpRequest`
  to an AI service.
- **In 0.1.0-9, pricing math lives on `QuoteLineItem`.** `Quote` gets a
  small set of deal-level header fields (above); the installed build adds no
  fields to `Account`, `Product2`, `Pricebook2`, `Order`, `OrderItem`, `Asset`
  or `Contract`. (Since 2026-10-07 the rules allow a feature to add fields
  to any standard object, so later builds may.)
- **No Subscription object.** Term/recurring math (`TermMonths__c`,
  `BillingFrequency__c`, `MRR__c`, `ARR__c`) lives on the `QuoteLineItem`
  itself; the post-sale "owned product" is the standard `Asset`.
- **Minimum-commitment floors are not enforced at quote time.** There is no
  actual usage yet to compare against a floor; `CommitmentContract__c`
  captures the contractual terms and a what-if simulator exists, but the
  real floor check is a billing-time concern, not a CPQ one.

## Money math

- **MRR** = `NetPrice__c × Quantity`, normalized to a month regardless of
  `BillingFrequency__c` (an Annual line's MRR is its annual charge ÷ 12).
- **ARR** = MRR × 12. The golden line: `101.83 × 100 × 12 = 122,196`.
- **TCV** (total contract value) = MRR × `TermMonths__c`, before any
  one-time/usage charges — those are tracked separately per chargeType.
- **Rounding**: `System.RoundingMode.HALF_UP` throughout. The FINAL scale
  depends on who owns the price: a PerUnit/Volume line's `NetPrice__c` is a
  real billed rate and rounds to **2 decimal places** (this is what makes
  $101.83 exact); a FlatFee/Tiered/Package line's net price is a synthetic
  per-unit figure (a total divided by quantity) and keeps **6 decimal
  places** so the recovered total doesn't lose cents — only the line TOTAL
  is rounded to currency. Sub-$1 per-unit rates (token/message/GB pricing)
  also keep 6dp so a $0.000015/token rate doesn't round to $0.00.
- Every intermediate waterfall value is carried at 6 decimal places and
  only the final `Net` stage is scale-normalized — so don't be alarmed to
  see `101.82575` on the `VolumeDiscount` row above `101.83` on `Net`.

## Build it (tester)

This module has no feature of its own to build — [golden-path.md](golden-path.md)
IS the worked build: six products, a bundle, two rate plans, three rule
sets, one quote. Do that first; every number and label in this module came
from reading the same installed code that golden path proves end to end.
When a feature module sends you here, it is because the tester asked "how
does the engine actually work" or "what is a waterfall stage" — explain
from the tables above, then send them back to that module's own "Build it".

## Check it (Claude)

Read-only queries that work against any quote, once the golden path's quote
exists (swap in another Quote Name once the tester has built their own):

```bash
# The full pipeline's output on one quote's lines
sf data query -o dd-e2e -q "SELECT Product2.Name, Quantity, DDCPQ__DerivedListPrice__c, DDCPQ__NetPrice__c, DDCPQ__MRR__c, DDCPQ__ARR__c, DDCPQ__ParentLine__r.Product2.Name FROM QuoteLineItem WHERE Quote.Name = 'Q-0042 Acme'"

# One line's full waterfall, stage by stage
sf data query -o dd-e2e -q "SELECT Product2.Name, DDCPQ__WaterfallJson__c FROM QuoteLineItem WHERE Quote.Name = 'Q-0042 Acme' AND Product2.Name = 'Sales Cloud'"

# Which rule sets are Active right now
sf data query -o dd-e2e -q "SELECT Name, DDCPQ__Status__c FROM DDCPQ__PricingMatrix__c"
sf data query -o dd-e2e -q "SELECT Name, DDCPQ__Status__c, DDCPQ__StackingStrategy__c FROM DDCPQ__SystemDiscountMatrix__c"

# The audit trail for a commit
sf data query -o dd-e2e -q "SELECT DDCPQ__EventType__c, CreatedDate FROM DDCPQ__AuditLog__c WHERE DDCPQ__Quote__r.Name = 'Q-0042 Acme' ORDER BY CreatedDate DESC"

# Which permission set this user actually has
sf data query -o dd-e2e -q "SELECT PermissionSet.Name FROM PermissionSetAssignment WHERE Assignee.Username = '<username>' AND PermissionSet.NamespacePrefix = 'DDCPQ'"
```

A read-only price preview, independent of the cart (never call `dd/v1/commit`):

```bash
sf api request rest "services/apexrest/DDCPQ/dd/v1/price?includeWaterfall=true" -o dd-e2e --method POST --body @body.json
```

```json
{
  "quoteId": "<0Q0...>",
  "selections": [{ "localKey": "p", "productId": "<01t...>", "quantity": 100 }]
}
```

## Expected numbers

From the verification run in cpq-pkg (own CN-prefixed data, not the golden
quote): "CN Concepts Widget", Pricebook list $150, qty 100, Account
Industry = Healthcare, Opportunity Type = Renewal.

| Stage                                                                 | Value                                                          |
| --------------------------------------------------------------------- | -------------------------------------------------------------- |
| PricebookList                                                         | 150                                                            |
| ContractPrice                                                         | 150 (no match, skip)                                           |
| DerivedListPrice                                                      | 130 (Absolute price rule)                                      |
| RampAdjustedPrice / TermAdjustedPrice / UsageTierPrice / PricingModel | 130 (all skip — no ramp, curve or tiers authored)              |
| SystemDiscount                                                        | 119.795 (5% Healthcare x 3% Renewal, AllStack, 7.85% combined) |
| PromotionDiscount / ChannelDiscount                                   | 119.795 (skip — no code, no partner tier)                      |
| VolumeDiscount                                                        | 101.82575 (15% tier at qty >= 100)                             |
| ManualDiscount / CommitDiscount                                       | 101.82575 (skip)                                               |
| Net                                                                   | 101.83                                                         |
| totalNet (REST)                                                       | 10183 (cents)                                                  |
| totalArr (REST)                                                       | 122196                                                         |
| systemDiscountPct on the draft                                        | 7.85                                                           |
| volumeDiscountPct on the draft                                        | 15                                                             |

This matches the golden quote's Sales Cloud line exactly (same inputs,
different product) — the two independent builds agree.

## Troubleshooting

| Symptom                                                                       | Cause                                                                                                | Fix                                                                                                                                                   |
| ----------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------- |
| A number looks right at DerivedListPrice but wrong by the time it reaches Net | One of the discount stages fired when it should not (or did not fire when it should)                 | Open the waterfall drawer and read each stage's note — every skipped stage says why in plain English                                                  |
| Volume/Usage matrix will not go Active                                        | "Target Product is required on Active volume-discount matrices"                                      | Set TargetProduct__c on the matrix header before switching Status to Active                                                                           |
| A rule you just activated never fires                                         | Still shows DDCPQ__Status__c = Draft                                                                 | Rules Studio always saves new rule sets as Draft; switch to Active explicitly                                                                         |
| Two discounts you expected to add (5%+3%=8%) instead gave 7.85%               | StackingStrategy__c = AllStack multiplies, it does not add                                           | Expected behaviour — 1 - (0.95 x 0.97) = 0.0785                                                                                                       |
| A criteria condition never matches even though the value looks right          | Wrong ValueType__c (e.g. a number compared as string) or the field path is not under an allowed root | Check ValueType__c matches the field's real type; only Account._, Opportunity._, Quote._, User._, Product._, Line._, Block._, Variable._ are readable |
| A product priced fine in preview but Commit refuses it                        | No Active RatePlan__c for that product (the plan gate)                                               | Every priced product needs an Active rate plan before commit, not just before preview                                                                 |
| "Working on a quote needs the Build Quotes permission" for a System Admin     | DD CPQ checks its own custom permission, not profile/object access                                   | Assign DD_CPQ_Engine_Admin or DD_CPQ_Engine_User                                                                                                      |
| A line's net price never changes no matter what you edit                      | SalesPriceOverride__c is set — a typed price is final and skips every discount stage                 | Clear the override to let the engine price it again                                                                                                   |
| A margin-floor row never appears in the waterfall                             | MarginFloorService only emits a stage when a floor rule evaluated, which needs a Cost__c on the line | Put a Cost on the line and an Active MarginFloorRule__c covering it                                                                                   |

## Limits and gotchas

- CompactabilityMatrix__c / CompactabilityRule__c keep the missing second
  "i" on purpose (CLAUDE.md rule 4) — do not "fix" it in a query or a rule
  name.
- DDCPQ-68 (rule tester removed from Rules Studio) and DDCPQ-104 (promotion
  code length) are trunk-only — not in 0.1.0-9.
- DDCPQ-67 (2,000-line cart cap, background commit) is also trunk-only;
  this build has no stated line-count ceiling of its own, but ordinary
  Apex governor limits (SOQL rows, CPU time, heap) still apply to a very
  large request — DdCpqLimits.requireWithinLimits is the guard that refuses
  an oversized price/commit request before Salesforce's own CPU-time
  exception, which names nothing useful.
- A beta package (0.1.0-9) cannot be upgraded in place; do not let a tester
  try — see SKILL.md.
- Every criteria-driven rule reads context that is loaded once per
  transaction and cached (CriteriaContextService). Editing a Quote or
  Account field mid-session and repricing without reopening the cart can
  look like stale data — the cache resets per previewDrafts/runPipeline
  call, not per click.

## Glossary

| Term                     | Meaning                                                                                                             |
| ------------------------ | ------------------------------------------------------------------------------------------------------------------- |
| Pipeline                 | The fixed, ordered sequence of engine stages from Eligibility through Audit                                         |
| Waterfall                | The per-line, per-stage price trace from PricebookList to Net                                                       |
| Stage                    | One named step in the waterfall (e.g. SystemDiscount)                                                               |
| Rule set / matrix        | The parent record (e.g. PricingMatrix__c) carrying Status; "matrix" is the API-name word, "rule set" is the UI word |
| Rule                     | One criteria row under a matrix (e.g. PricingRule__c)                                                               |
| Criteria                 | The 5-field schema (FieldApiName__c / Operator__c / Value__c / ValueType__c / Priority__c) one rule row carries     |
| Catch-all                | A rule with blank FieldApiName__c / Operator__c — matches every line                                                |
| Conditions__c            | The advanced multi-condition JSON that replaces the single criteria fields when filled                              |
| Target__c                | The JSON "Products Covered" targeting mode (All / Products / Criteria / Group)                                      |
| TargetProduct__c         | The simple single-product lookup most rules use instead                                                             |
| RuleTarget               | The Apex helper that resolves which rules cover a product and in what precedence                                    |
| First-match              | Only the highest-priority matching rule applies (Pricing, Eligibility)                                              |
| All-match / stacking     | Every matching rule applies, combined per the matrix's stacking strategy (System Discount)                          |
| AllStack                 | Multiply every matching discount together                                                                           |
| TakeBest                 | Keep only the single largest matching discount                                                                      |
| Derived List Price       | List price after the Pricing Rule stage, before any discount                                                        |
| Net Price                | The final per-unit price, after every stage                                                                         |
| Bundle parent / header   | The top line of a bundle; roll-up by default (not rated — children carry the price)                                 |
| Roll-up vs header-priced | Whether a BundleDefinition__c rates the parent line itself or leaves it at a pass-through price                     |
| Included / Informational | A bundle option's pricing role — Included = covered by the parent's price, never rated on its own                   |
| Rate Plan                | RatePlan__c — a product's pricing model: revenue nature, pricing math, billing schedule                             |
| Revenue nature           | Recurring / OneTime / Usage / Overage — drives whether a line counts toward MRR                                     |
| Pricing model            | FlatFee / PerUnit / Tiered / Volume / Package / PercentOfBasis — the math, not the schedule                         |
| Plan gate                | The rule that refuses Commit for a priced product with no Active Rate Plan                                          |
| Margin floor             | The policy-level minimum net price against unit cost; the pipeline's last word                                      |
| Deal Block               | A named sub-group of lines inside one quote, with its own window/discount                                           |
| Co-term                  | Aligning a line's end date to the deal's own end date                                                               |
| Baseline                 | A snapshot of a customer's owned lines, used to compute amend/renew deltas                                          |
| Variable                 | A reusable, dynamically-resolved value a rule's RHS or price can reference                                          |
| AuditLog                 | The append-only record of overrides, activations and finalize events                                                |
| Custom permission        | Administer_Pricing / Build_Quotes — checked by DD CPQ itself, independent of profile                                |
| Charge profile           | One priced facet of a hybrid product (base + usage + overage) that can fan out into sibling lines                   |
| Ramp                     | Per-period price steps over a term (e.g. year 1 cheaper than year 2)                                                |
| Term discount curve      | A whole-contract-length discount, distinct from a ramp's per-period steps                                           |
| Commitment contract      | A prepaid dollar/unit commitment that unlocks a locked discount rate                                                |

## Questions testers ask

**"Why does the waterfall show so many stages that all say the same price?"**
Every stage in the fixed order always appears, even when nothing changed —
a pass-through with a note explaining why it skipped. A missing row would
look like a bug, so every stage is shown.

**"I set a 5% discount and a 3% discount — why is the combined discount not 8%?"**
Discounts stack multiplicatively under AllStack: 1 - (0.95 x 0.97) =
7.85%, not 5 + 3.

**"Why did Commit refuse a line that priced fine a second ago?"**
The plan gate only blocks Commit, not preview — a product needs an Active
Rate Plan before you can write it to the quote, even if the engine was
happy to show you a price for it.

**"I'm a System Admin — why am I locked out?"**
DD CPQ checks its own Build Quotes / Administer Pricing custom permissions
in Apex. Profile and Modify All do not satisfy them; only the
DD_CPQ_Engine_Admin / DD_CPQ_Engine_User permission set does.

**"Is any of this AI?"**
No — not in the path from Add Product to Commit. Plain-English rule
authoring, where offered, happens at design time only, and converts to the
same deterministic rule rows this module describes.

**"Which object do I edit to change a product's price without touching a rule?"**
None of the above — that is the plain PricebookEntry.UnitPrice, which feeds
the very first waterfall stage, PricebookList.

**"Why is the Compatibility object spelled Compactability?"**
A locked-in managed-package typo (CLAUDE.md rule 4) — not a mistake you can
fix by renaming it.

**"The cart opened as a popup instead of a full tab — is that a bug?"**
No — an admin can choose "page" or "popup" surface per view; both call the
same engine and price identically.
