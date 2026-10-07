# Bundles

> Installed build 0.1.0-9 · Verified 2026-10-07 in cpq-pkg (BN-prefixed data):
> RollUp header shows its own pricebook price unrated (999) while a Match-the-
> bundle-quantity option follows parent qty 10→5 at ×0.5 multiplier; Header
> Priced bundle rates $500/unit × qty 2 = $1,000 with its Included option
> forced to $0; Mixed bundle rates $300/unit × qty 2 = $600 header + $20/unit
> × qty 2 = $40 Priced extra; Match-with-Other-Option ×2 off a base of 4 → 8;
> a Variable constant of 7 drives an option to quantity 7; nesting a bundle
> inside a bundle is refused with `BUNDLE_NESTING_REFUSED` at Bundle Builder
> save.

## What it is for

A bundle is one product (the **header**, a.k.a. the bundle parent) sold
together with a fixed structure of other products (its **options**), grouped
into **features** (the questions a rep answers: "which edition", "which
add-ons"). DD CPQ bundles are **exactly two levels deep, on purpose** — a
header and its options, nothing nested inside an option. Three independent
decisions make a bundle behave the way a given catalogue needs it to:

- **Is the bundle itself priced, or do only its options carry a price?**
  (`PricingMode__c` — RollUp / HeaderPriced / Mixed.)
- **How many options can/must the rep pick per feature, and how are they
  shown?** (`ProductFeature__c.DisplayType__c` / `MinSelections__c` /
  `MaxSelections__c`.)
- **For each option: is it charged, included for free, or just informational
  — and how is its quantity derived?** (`ProductOption__c.PricingRole__c` /
  `QuantityRule__c`.)

**Not this module:** compatibility rules between products, including
cross-option "requires"/"excludes" constraints a bundle can also carry
([compatibility.md](compatibility.md)); eligibility ("who may buy this
bundle at all") ([eligibility.md](eligibility.md)); the rate plans that price
a header or an option once it is Priced
([pricing-models.md](pricing-models.md)); ramps and term curves applied to a
header or option ([ramps-and-term-curves.md](ramps-and-term-curves.md)); the
cart's general grouping/columns/Deal Blocks behaviour that is not
bundle-specific ([cart.md](cart.md)).

## How it works

### With no `BundleDefinition__c` at all

A product can have `ProductOption__c` rows (so it has structure) with **no**
`BundleDefinition__c` record pointing at it. The engine still recognises it
as a bundle parent — any line with no parent and with children in the same
request is a bundle header — but with no definition the pricing mode
defaults to **RollUp**, the behaviour every bundle had before
`BundleDefinition__c` existed. This is a deliberate backward-compatibility
fallback (DDCPQ-BB-07), not a gap: a header with no definition is never
itself rated, and every one of its options prices on its own regardless of
`PricingRole__c`. Build a `BundleDefinition__c` whenever the header needs to
be priced, ramped, or discounted as a line in its own right.

### The pricing modes — exact math

`BundleDefinition__c.PricingMode__c` decides whether the header is
**rateable** — whether the full pricing pipeline (rate plan, ramp, term
curve, discounts, margin floor) runs on the header's own line at all, as
opposed to the header just echoing its Pricebook list price with none of
that math applied.

| `PricingMode__c`   | Label (Bundle Builder)   | Header rateable?                                                                                                                        | What the options do                                                                                                                                                                                                                                              |
| ------------------ | ------------------------ | --------------------------------------------------------------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `RollUp` (default) | Options carry the price  | **No.** The header's `NetPrice__c` = its `DerivedListPrice__c` = its own Pricebook list price, untouched by any pricing stage.          | Every option prices independently off its own rate plan, by its own `PricingRole__c`.                                                                                                                                                                            |
| `HeaderPriced`     | One price for the bundle | **Yes.** The header is rated by its own rate plan (any `PricingModel__c`), ramped, term-curved and discounted like any standalone line. | Options are expected to be `PricingRole__c = Included` (forced to `NetPrice__c = 0`) — the header's price already covers them. An option left `Priced` charges on top, which the validator flags as `HEADER_PRICED_COMPONENT_ALSO_PRICED` (see Troubleshooting). |
| `Mixed`            | Bundle price + extras    | **Yes**, same as HeaderPriced.                                                                                                          | A mix: some options `Included` (free), some left `Priced` and charged on top of the header's own price. This is the only mode where `HEADER_PRICED_COMPONENT_ALSO_PRICED` does not fire for a Priced option — it is the point of the mode.                       |

**Verified (cpq-pkg, `BN Test Quote`):**

| Scenario     | Header plan              | Qty | Header net                          | Option                                     | Option role | Option net                 |
| ------------ | ------------------------ | --- | ----------------------------------- | ------------------------------------------ | ----------- | -------------------------- |
| RollUp       | none (no plan on header) | 3   | **999** (= Pricebook list, unrated) | BN RU Option A (PerUnit $100)              | Priced      | **100**/unit (qty 3 → 300) |
| RollUp       | "                        | "   | "                                   | BN RU Option B (PerUnit $50, static qty 2) | Priced      | **50**/unit (qty 2 → 100)  |
| HeaderPriced | PerUnit $500             | 2   | **500**/unit → **1,000** total      | BN HP Included Opt                         | Included    | **0**                      |
| Mixed        | PerUnit $300             | 2   | **300**/unit → **600** total        | BN MX Included Opt                         | Included    | **0**                      |
| Mixed        | "                        | "   | "                                   | BN MX Priced Opt (PerUnit $20)             | Priced      | **20**/unit (qty 2 → 40)   |

A RollUp header is not forced to $0 — it still shows whatever its own
Pricebook entry says, which is exactly why `ROLLUP_HEADER_HAS_PRICE` exists
(see Troubleshooting): if that Pricebook price is non-zero, the rep sees a
number on the header row that nothing ever subtracts, on top of the options'
own prices.

### Discounts, and the bundle total in the cart

System/volume/promo/channel discounts and margin floors run on whichever
lines are rateable. In RollUp, that is every option, never the header — so
discounting the bundle means discounting its options (a volume tier keyed
on the header's own quantity still applies per-option, since the rate plan
and quantity live on each option). In HeaderPriced/Mixed, the header is a
normal rateable line and takes the same discounts any standalone product
would; `Included` options, having `NetPrice__c` forced to `0`, are
unaffected by a discount (there is nothing to discount).

The cart shows a **bundle total** as the sum of every line in the group
(header + options) at their current net prices — it is a display rollup,
not a stored field. For RollUp, that total already excludes double-counting
because the header's own price is meant to be zero (or the admin is meant
to zero it); DD CPQ does not enforce that automatically, which is the gap
`ROLLUP_HEADER_HAS_PRICE` warns about.

### Quantity rules — exact math

`ProductOption__c.QuantityRule__c` decides where an option's quantity comes
from. `BundleQuantityResolver` computes it server-side at commit/preview
time, and it is **authoritative** — a client-supplied quantity on a
non-Static rule is overridden unless `QuantityEditable__c` is also true and
the rep actually touched the stepper. Verified: a `Match with Parent`
option sent `quantity: 10` in the request still resolved to `5` because its
rule said otherwise.

| `QuantityRule__c`         | Driver value                                                                                   | Then            |
| ------------------------- | ---------------------------------------------------------------------------------------------- | --------------- |
| `Static` (default)        | none — uses `Quantity__c` (or the selection's own quantity if `Quantity__c` is blank) verbatim | nothing further |
| `Match with Parent`       | the bundle header's own quantity                                                               |                 |
| `Variable`                | `VariableService.resolveOne(QuantityVariable__c, …)`                                           |                 |
| `Match with Other Option` | another option's **already-resolved** quantity (topologically — the target resolves first)     |                 |

For the three non-Static rules, the full pipeline is:

```
driver = driverValue × (QuantityMultiplier__c ?? 1)
result = QuantityOperator__c applied to (driver, Quantity__c as target):
   '='  (default/null) → driver verbatim
   '<=' → MIN(target, driver)        '<' → MIN(target, driver − 1)
   '>=' → MAX(target, driver)        '>' → MAX(target, driver + 1)
result = clamp to [MinQty__c, MaxQty__c] if set
result = MAX(1, result)   — never less than 1, never null
```

**Verified (cpq-pkg, `BN Qty Bundle`, header quantity 10):**

| Option                   | Rule                                                            | Setup                        | Resolved qty                     |
| ------------------------ | --------------------------------------------------------------- | ---------------------------- | -------------------------------- |
| BN Qty Match Parent x0.5 | Match with Parent                                               | multiplier 0.5, operator `=` | header 10 × 0.5 = **5**          |
| BN Qty Static Base       | Static                                                          | `Quantity__c` = 4            | **4** (unaffected by header qty) |
| BN Qty Match Other x2    | Match with Other Option → the Static option above, multiplier 2 |                              | 4 × 2 = **8**                    |
| BN Qty Variable Driven   | Variable → a Constant-type Variable = 7                         |                              | **7**                            |

A **cycle** in `Match with Other Option` (A drives off B, B drives off A,
directly or transitively) is detected and refused with
`BundleQuantityCycleException`: _"Bundle quantity cycle involving
ProductOption\_\_c \<Id\>. One option transitively depends on itself via
Match with Other Option."_ — fix by breaking the loop, not by retrying.

`QuantityEditable__c` only matters on a non-Static rule: it shows the rep an
editable stepper next to the rule-computed chip. The engine still records
both the rule-computed and the rep's final value in the audit log either
way.

### Nesting — refused, not silently flattened

DD CPQ enforces exactly two levels: **a bundle is a header + its options,
and an option can never itself be a bundle.** This used to fail silently —
dropping a bundle product as another bundle's option rendered as an
ordinary flat leaf in the configurator, with its own options invisible,
because configure reads one hop and commit is a two-phase insert. Since
DDCPQ-BB-07, Bundle Builder's validator catches it at save time:

> _`BUNDLE_NESTING_REFUSED`: "\<option name\>" is itself a bundle, and DD CPQ
> is deliberately two levels: a bundle and its options, nothing deeper. Add
> that bundle's own options to this one directly, or sell it as a separate
> line._

**Verified** (cpq-pkg, via `bundle.validate`): setting `BN Nest Outer
Bundle`'s only option to the product `BN Nest Inner Bundle` (itself a
bundle header with its own feature/option) returned exactly that finding,
`severity: error`, scope `option`. The same validate call also returned
`ROLLUP_HEADER_HAS_PRICE` (the outer bundle's own Pricebook price was
non-zero) and `NO_RATE_PLAN` for the nested option — three independent
checks firing together on one bad draft, which is normal; fix the nesting
first, the others may resolve themselves once the structure is sound.

### Commit shape

A committed bundle writes **one `QuoteLineItem` per header plus one per
option** — never a single collapsed row. **Verified** (`BN Mixed Bundle`,
qty 2, header + 2 options → 3 `QuoteLineItem`s inserted in one commit call):

| Field                  | Header row                     | Option row                                                |
| ---------------------- | ------------------------------ | --------------------------------------------------------- |
| `ParentLine__c`        | blank                          | the header's own `QuoteLineItem` Id                       |
| `SourceBundle__c`      | the bundle's own `Product2` Id | the **same** bundle `Product2` Id (not the header QLI)    |
| `UnitPrice` (standard) | the rated price                | `0` for an Included option, its own rated price otherwise |

`SourceBundle__c` is a lookup to **Product2**, not to the header line — so
two different quotes, or two instances of the same bundle on one quote, all
point their options at the same bundle Product2 record; only `ParentLine__c`
tells you which specific header **line** an option belongs to.

### Visual collapse and sibling linkage — a different feature, do not confuse them

`SiblingGroupKey__c` / `SiblingRoot__c` and the cart's "collapse into one
row with chips" behaviour (DDCPQ-057/059) are about a **hybrid product**
that fans out into multiple charge-type lines (e.g. a Recurring base +
a one-time setup fee on the _same_ product) — not about bundle options.
A bundle option's siblings-under-one-header are tracked by
`ParentLine__c` + `SourceBundle__c` as above; `SiblingGroupKey__c` stays
null on ordinary bundle lines unless an option itself happens to be a
hybrid product, in which case both groupings apply independently and
compose normally (a hybrid option inside a bundle gets a `ParentLine__c`
pointing at the bundle header **and** a `SiblingGroupKey__c` shared with its
own OneTime/Usage sibling rows).

### The Bundle Builder tab

One workspace, not a wizard: a **Setup rail** on the left, a **canvas** in
the centre (the bundle's structure — header, features, options, rules), and
an **inspector** on the right (whatever you've selected). There is no List
price field anywhere — a product's price always comes from its rate plan.

- **Open…** in the header searches saved bundles; each result opens with
  **Edit** or **Start a copy** (clones the whole draft with every Id
  stripped — the save path then creates new records instead of updating).
  **New** starts a fresh bundle and asks before discarding unsaved work.
- **Choosing the bundle product**: **Use an existing product** (search by
  name/code; results are tagged **Inactive**, **Bundle** — cannot be an
  option elsewhere — or **In this bundle**) or **Create a new product**
  (Name, Product code, Family, Description; the product and its Pricebook
  entry are created only on save, never before). Choosing a product that is
  already a bundle opens _that_ bundle for editing instead of nesting it.
- **Pricing** (inspector): the three mode buttons from How it works —
  **Options carry the price** / **One price for the bundle** / **Bundle
  price + extras** — each with a one-line hint. "More settings" hides
  Bundle type / Header treatment / Visible to reps.
- **+ Add feature**: name it, then **Pick one** (Radio) / **Pick several**
  (Checkbox) / **Dropdown**, plus **At least** / **At most** (min/max
  selections). The feature's canvas header states the rule in words, e.g.
  "pick exactly one".
- **+ Add option** under a feature: search by name/code, or "Create '\<text\>'
  as a new product" inline. A new option product asks for its role —
  **Charged separately** / **Included in the price** / **Informational** —
  and, if charged, an inline rate plan (**Charge type** Recurring / One-time
  / Usage, **Price**, **Priced as** Per unit or flat fee, **Billed**
  Monthly/Quarterly/Annual/Upfront). These rate plans save as Active and
  are the Rate Plan Editor's own records — tiers, trials and credit
  schedules are added there afterward, not here.
- An option missing a plan shows a **No plan** or **Draft plan** badge;
  click it (or the inspector's **Add a rate plan**) to open the same inline
  rate plan editor.
- **Quantity** (inspector, per option): **Fixed quantity** / **Match the
  bundle quantity** / **From a variable** / **Match another option** — the
  four `QuantityRule__c` values, with Min/Max quantity and "The rep can
  change the quantity" (`QuantityEditable__c`).
- **+ Add rule** on the canvas: option compatibility — require/exclude,
  **Block the rep** or **Warn only**, plus a message shown to the rep.
- The rail's **checklist** (Bundle product · Pricing · Features & options ·
  Rate plans · Ready) lights up per step; **Checks** below it lists errors
  (red, block the save) and warnings (amber, do not) — click any finding to
  jump to the offending row. With errors present, **Save anyway, as a draft
  to finish later** still saves (it cannot override a product that could
  not be found).
- **Create bundle** (new) / **Save changes** (editing) / **Save as new
  bundle** (from Start a copy) commits everything — header, features,
  options, new products, new rate plans — in one transaction; a failure
  writes nothing.

**Validation codes** (from `docs/BUNDLE_BUILDER_GUIDE.md`, cross-checked
against `BundleDraftValidator.cls`): `NO_FEATURES`, `FEATURE_NO_NAME`,
`FEATURE_NO_OPTIONS`, `NO_RATE_PLAN`, `NEW_PRODUCT_INCOMPLETE`,
`NEW_PRODUCT_CODE_TAKEN`, `NEW_PRODUCT_CODE_DUPLICATED_IN_DRAFT`,
`RATE_PLAN_ONLY_DRAFT`, `OPTION_NO_PRODUCT`, `INLINE_RATE_PLAN_INCOMPLETE`,
`MIN_ABOVE_MAX`, `MIN_UNSATISFIABLE`, `BUNDLE_NESTING_REFUSED`,
`RATE_PLAN_NOT_NEEDED`, `RATE_PLAN_ALREADY_ACTIVE`, plus the two
pricing-mode warnings from How it works: `ROLLUP_HEADER_HAS_PRICE` and
`HEADER_PRICED_COMPONENT_ALSO_PRICED`. Full wording for each is in the
guide; `BUNDLE_NESTING_REFUSED` and `ROLLUP_HEADER_HAS_PRICE`'s wording is
quoted verbatim above and was re-verified live against cpq-pkg.

**Not in the builder**: Tiered/Volume/Package/Percent pricing, trials,
prepaid credit and usage tiers (all added in the Rate Plan Editor after
save); changing a saved bundle's product (use Start a copy instead); bulk
operations; product images.

### The cart's bundle configurator

Adding a bundle to a quote opens a drawer (`cpqBundleConfigurator`), not
the Bundle Builder. It shows the header's name and a **Bundle quantity**
stepper, then each feature as a row of chip buttons — one per option,
labelled with the option's product name. A selected chip shows a
checkmark; a **Required** option carries a **Required** badge; an option
blocked by an unmet compatibility rule is disabled and shows **∅ \<reason\>**;
an option with an unmet "requires" shows **⚠ Requires \<other option\>**.
Selecting an option reveals its own quantity input only if its
`QuantityRule__c` allows editing. Changing the bundle quantity re-resolves
every `Match with Parent` / `Match with Other Option` / `Variable` option
live, the same math as "Quantity rules" above — the rep sees the chip's
quantity update without leaving the drawer.

## Objects and fields

| Object label (API name)                   | Field                                   | Meaning / values                                                                                                                                                                                                                                                     |
| ----------------------------------------- | --------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Bundle Definition (`BundleDefinition__c`) | `BundleProduct__c`                      | Lookup to `Product2` — the header. At most one Active definition per product.                                                                                                                                                                                        |
|                                           | `BundleType__c`                         | `Configurable` (default, rep picks within min/max) / `Static` (descriptive only in v1 — the configurator already honours `MinSelections`/`MaxSelections`, so a "Static" bundle is just one whose features are fully constrained).                                    |
|                                           | `PricingMode__c`                        | `RollUp` (default) / `HeaderPriced` / `Mixed` — see How it works.                                                                                                                                                                                                    |
|                                           | `HeaderTreatment__c`                    | `RealSku` (default, a real sellable product) / `VirtualGrouping` (a marketing grouping only). Descriptive — changes no engine math.                                                                                                                                  |
|                                           | `Status__c`                             | `Draft` (default) / `Active` (only these are read by the engine) / `Inactive`. A Draft or Inactive definition leaves the bundle on the legacy RollUp path — a safe one-field rollback.                                                                               |
| Product Feature (`ProductFeature__c`)     | `ParentProduct__c`                      | Lookup to `Product2` — the bundle this feature belongs to.                                                                                                                                                                                                           |
|                                           | `DisplayType__c`                        | `Radio` (default, pick one) / `Checkbox` (pick several) / `Dropdown` (pick one from a list).                                                                                                                                                                         |
|                                           | `MinSelections__c` / `MaxSelections__c` | How many options the rep must/can pick from this feature. Number(3,0), optional.                                                                                                                                                                                     |
|                                           | `SortOrder__c`                          | Required. Display order among the bundle's features.                                                                                                                                                                                                                 |
| Product Option (`ProductOption__c`)       | `ParentProduct__c` / `ChildProduct__c`  | The bundle and the option product, both lookups to `Product2`.                                                                                                                                                                                                       |
|                                           | `Feature__c`                            | Lookup to `ProductFeature__c` — which feature this option belongs to.                                                                                                                                                                                                |
|                                           | `Required__c`                           | Checkbox. Must be selected when the bundle is added.                                                                                                                                                                                                                 |
|                                           | `DefaultSelected__c`                    | Checkbox. Pre-selected when the configurator opens — an admin's starting position, not a computed recommendation.                                                                                                                                                    |
|                                           | `PricingRole__c`                        | `Priced` (default, rated normally) / `Included` (forced `NetPrice__c = 0`, contributes no MRR — the Steelbrick `Bundled` equivalent) / `Informational` (shown, never rated). Blank is **not** exempt from needing a rate plan — only `Included`/`Informational` are. |
|                                           | `InclusionReason__c`                    | `Included` (default) / `Promotional` / `ZeroRated` / `Trial` — why a component prices at zero. Only meaningful when `PricingRole__c` is Included or Informational.                                                                                                   |
|                                           | `QuantityRule__c`                       | `Static` (default) / `Match with Parent` / `Variable` / `Match with Other Option` — see Quantity rules.                                                                                                                                                              |
|                                           | `Quantity__c`                           | Default/static quantity, or the clamp target for a non-`=` operator.                                                                                                                                                                                                 |
|                                           | `QuantityMultiplier__c`                 | Multiplied against the driver value before the operator. Null = 1. Must be > 0 when set.                                                                                                                                                                             |
|                                           | `QuantityOperator__c`                   | `=` (default) / `<=` / `<` / `>=` / `>` — how the driver and `Quantity__c` combine. Null = legacy `=`.                                                                                                                                                               |
|                                           | `QuantityVariable__c`                   | Lookup to `Variable__c`, required for the `Variable` rule. Lookup filter requires the Variable to be **Active**.                                                                                                                                                     |
|                                           | `QuantityDrivenByOption__c`             | Self-lookup to a sibling `ProductOption__c`, required for `Match with Other Option`.                                                                                                                                                                                 |
|                                           | `MinQty__c` / `MaxQty__c`               | Clamp bounds on the resolved quantity.                                                                                                                                                                                                                               |
|                                           | `QuantityEditable__c`                   | Checkbox. Lets the rep override a rule-computed quantity in the configurator; both values are audited.                                                                                                                                                               |
|                                           | `Discountable__c`                       | Checkbox, default true. If false, discount matrices skip this option's line.                                                                                                                                                                                         |
|                                           | `AllocationWeight__c`                   | Number(5,2). This component's share of a header-priced bundle for ASC 606 revenue allocation — stored and reported, **not yet applied** by the engine.                                                                                                               |
|                                           | `SortOrder__c`                          | Display order of options within a feature.                                                                                                                                                                                                                           |
| Quote Line Item (standard, extended)      | `ParentLine__c`                         | Lookup to the header `QuoteLineItem` — blank on the header itself, set on every option row.                                                                                                                                                                          |
|                                           | `SourceBundle__c`                       | Lookup to `Product2` — the bundle header's **product**, on both the header row and every option row.                                                                                                                                                                 |
|                                           | `SiblingGroupKey__c` / `SiblingRoot__c` | Hybrid-charge fan-out, not bundle structure — see "a different feature" above.                                                                                                                                                                                       |

## Build it (tester)

Do this after the golden path, using your own products — do not touch CRM
Suite Pro or its options. This builds all three pricing modes side by side.

1. **Products tab → New** three times: `BN RollUp Demo`, `BN Header Demo`,
   `BN Mixed Demo` (each is a bundle header). **Active** ticked, each gets
   a Standard Price (any value — RollUp's will show through unrated, so set
   it to `0` there to avoid the `ROLLUP_HEADER_HAS_PRICE` warning; `150` is
   fine for the other two, it gets overridden by their own rate plan once
   rated). Also create two option products, `BN Seat` and `BN Support`,
   Standard Price `150` each.
2. **Bundle Builder tab → New.** **Use an existing product** → `BN RollUp
Demo`. In the inspector under "Pricing", leave **Options carry the
   price** selected. **+ Add feature** → name it `Core`, **Pick several**.
   **+ Add option** twice → `BN Seat`, `BN Support`; for each, role
   **Charged separately**, on by default, **Add a rate plan** → Recurring,
   Price `100`, Per unit, Monthly. **Create bundle**.
3. Repeat, opening a **new** bundle on `BN Header Demo`: in the inspector
   choose **One price for the bundle**. Click the bundle card itself (not
   an option) → **Add a rate plan** → Recurring, Price `500`, Per unit,
   Monthly. Add one option `BN Seat` with role **Included in the price**
   (no rate plan needed — the Checks panel should not ask for one).
   **Create bundle**.
4. Repeat once more on `BN Mixed Demo`, pricing mode **Bundle price +
   extras**: header rate plan Recurring/$300/Per unit/Monthly, one option
   `BN Seat` **Included in the price**, one option `BN Support` **Charged
   separately** with its own rate plan (Recurring/$20/Per unit/Monthly).
5. Build a quote on an Account/Opportunity of your own (e.g. `BN Test Co`).
   **Configure Products** → add each of the three bundles at quantity `2`.
   Read each header's price and each option's price in the cart against
   "Expected numbers" below.
6. Try the nesting refusal yourself: open Bundle Builder on a fourth
   product, add a feature, and try **+ Add option** → search for
   `BN RollUp Demo` (a product that is already a bundle). The search result
   is tagged **Bundle** and cannot be added as an option — this is the UI's
   first line of defense before the validator's `BUNDLE_NESTING_REFUSED`
   even runs.

## Check it (Claude)

```bash
sf data query -o cpq-pkg -q "SELECT DDCPQ__BundleProduct__r.Name, DDCPQ__PricingMode__c, DDCPQ__Status__c FROM DDCPQ__BundleDefinition__c WHERE DDCPQ__BundleProduct__r.Name LIKE 'BN %'"
sf data query -o cpq-pkg -q "SELECT DDCPQ__ParentProduct__r.Name, DDCPQ__ChildProduct__r.Name, DDCPQ__PricingRole__c, DDCPQ__QuantityRule__c, DDCPQ__QuantityMultiplier__c, DDCPQ__Required__c, DDCPQ__DefaultSelected__c FROM DDCPQ__ProductOption__c WHERE DDCPQ__ParentProduct__r.Name LIKE 'BN %' ORDER BY DDCPQ__SortOrder__c"
sf data query -o cpq-pkg -q "SELECT Product2.Name, Quantity, UnitPrice, DDCPQ__DerivedListPrice__c, DDCPQ__NetPrice__c, DDCPQ__ParentLine__r.Product2.Name, DDCPQ__SourceBundle__c FROM QuoteLineItem WHERE Quote.Name = 'BN Test Quote'"
```

Expect: each `BundleDefinition__c` `Active` with the `PricingMode__c` you
chose; each `Included` option's committed `NetPrice__c` = `0` regardless of
its standard `UnitPrice`; every option row's `ParentLine__r.Product2.Name`
equal to the header's product name; every row (header and options) sharing
the same `SourceBundle__c` Id.

A read-only price preview (no commit), for a header + one option — swap in
your own quote and product/option Ids:

```json
{
  "quoteId": "<0Q0…>",
  "selections": [
    {
      "localKey": "p",
      "productId": "<header 01t…>",
      "quantity": 2,
      "sourceBundleId": "<header 01t…>"
    },
    {
      "localKey": "c1",
      "parentLocalKey": "p",
      "productId": "<option 01t…>",
      "quantity": 2,
      "sourceBundleId": "<header 01t…>",
      "productOptionId": "<ProductOption__c Id>"
    }
  ]
}
```

```bash
sf api request rest "services/apexrest/DDCPQ/dd/v1/price?includeWaterfall=true" -o cpq-pkg --method POST --body @body.json
```

`productOptionId` is what lets the engine read `PricingRole__c` and apply a
`QuantityRule__c` — a preview sent without it treats the line as an
ordinary standalone product (no $0 forcing, no quantity resolution). Look
at `lines[].netPrice` and `lines[].quantity` against "Expected numbers".
**Never** call `dd/v1/commit` — that is the tester's step.

## Expected numbers

Verified in cpq-pkg, `BN Test Quote`, 2026-10-07:

| Scenario       | Line                                          | Qty sent           | Qty resolved | Net price                                                                                         |
| -------------- | --------------------------------------------- | ------------------ | ------------ | ------------------------------------------------------------------------------------------------- |
| RollUp         | header (no plan)                              | 3                  | 3            | 999 (Pricebook list, unrated)                                                                     |
| RollUp         | Option A, Match with Parent                   | 3                  | 3            | 100                                                                                               |
| RollUp         | Option B, Static qty 2                        | 2                  | 2            | 50                                                                                                |
| HeaderPriced   | header, PerUnit $500                          | 2                  | 2            | 500                                                                                               |
| HeaderPriced   | Included option                               | 2                  | 2            | 0                                                                                                 |
| Mixed          | header, PerUnit $300                          | 2                  | 2            | 300                                                                                               |
| Mixed          | Included option                               | 2                  | 2            | 0                                                                                                 |
| Mixed          | Priced extra, PerUnit $20                     | 2                  | 2            | 20                                                                                                |
| Quantity rules | header                                        | 10                 | 10           | 999                                                                                               |
| Quantity rules | Match with Parent ×0.5                        | 10 (sent, ignored) | **5**        | 10                                                                                                |
| Quantity rules | Static base = 4                               | 4                  | 4            | 10                                                                                                |
| Quantity rules | Match with Other Option ×2 (off the base)     | 4 (sent, ignored)  | **8**        | 10                                                                                                |
| Quantity rules | Variable = constant 7                         | 1 (sent, ignored)  | **7**        | 10                                                                                                |
| Nesting        | outer bundle, option = another bundle product | —                  | —            | refused: `BUNDLE_NESTING_REFUSED` (+ `ROLLUP_HEADER_HAS_PRICE`, `NO_RATE_PLAN` on the same draft) |

Commit shape (Mixed scenario, 3 lines inserted in one call): header
`ParentLine__c` blank, both options' `ParentLine__c` = the header's QLI Id;
all three rows' `SourceBundle__c` = the bundle's Product2 Id.

## Troubleshooting

| Symptom                                                                                                            | Cause                                                                                                                                                                                                                                                                                                                                                       | Fix                                                                                                                                 |
| ------------------------------------------------------------------------------------------------------------------ | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------- |
| A RollUp header shows a nonzero price and the bundle total looks double-counted                                    | `ROLLUP_HEADER_HAS_PRICE`: the header's own Pricebook entry is nonzero, and RollUp never zeroes it automatically. Exact message: _"This is a roll-up bundle, so the options carry the price — but the bundle product itself is listed at \<price\>. The rep will be charged for both. Either zero the header price or switch the bundle to Header Priced."_ | Zero the header's Standard Price, or switch `PricingMode__c` to `HeaderPriced` if the header really should carry a price.           |
| A HeaderPriced/Mixed option is charged on top when it shouldn't be                                                 | It is left `PricingRole__c = Priced` under a mode that expects `Included`. Validator: `HEADER_PRICED_COMPONENT_ALSO_PRICED` — _"\<option\> still prices separately while the bundle is Header Priced, so the customer pays for it twice. Mark it Included, or set the bundle to Mixed if charging on top is intended."_                                     | Set the option's role to `Included`, or switch the bundle to `Mixed` if the extra charge is intentional.                            |
| Adding a bundle-inside-a-bundle silently does nothing, or save refuses it                                          | Two levels only. Pre-BB-07 this rendered the inner bundle as a flat leaf with its options invisible; now the save is refused outright with `BUNDLE_NESTING_REFUSED`.                                                                                                                                                                                        | Flatten: add the inner bundle's own options directly to the outer bundle, or sell the inner bundle as its own separate quote line.  |
| A priced option commits at $0, or won't commit at all ("...cannot be priced... Each needs an Active rate plan...") | Its `PricingRole__c` is blank or `Priced` but it has no Active rate plan — blank is **not** exempt from the rate plan gate, only `Included`/`Informational` are.                                                                                                                                                                                            | Add an Active rate plan to the option's product, or if it is genuinely free, set `PricingRole__c` to `Included` or `Informational`. |
| An option's quantity doesn't match what you typed in the configurator                                              | `QuantityRule__c` is not `Static` and the engine's resolved value differs from your input — resolution is server-side and authoritative.                                                                                                                                                                                                                    | Check the option's rule, multiplier, operator and Min/MaxQty; if the rep should be able to override, tick `QuantityEditable__c`.    |
| Quantity rule throws a cycle error                                                                                 | A chain of `Match with Other Option` options loops back on itself (A→B→A, directly or through more hops). Exact message: _"Bundle quantity cycle involving ProductOption\_\_c \<Id\>. One option transitively depends on itself via Match with Other Option."_                                                                                              | Repoint one option in the chain at a Static (or otherwise non-cyclic) source.                                                       |
| `NO_RATE_PLAN` / `RATE_PLAN_ONLY_DRAFT` in Bundle Builder's Checks                                                 | The option (or header, if header-rated) has no plan, or only a Draft one, and will need one to price.                                                                                                                                                                                                                                                       | Click the option/header, **Add a rate plan**, or activate the existing Draft plan in the Rate Plan Editor.                          |
| `RATE_PLAN_NOT_NEEDED` warning                                                                                     | A rate plan was attached to an option marked `Included` — it will be ignored.                                                                                                                                                                                                                                                                               | Remove the plan, or change the option's role to `Priced` if it should actually charge.                                              |
| New product's code is rejected (`NEW_PRODUCT_CODE_TAKEN` / `_DUPLICATED_IN_DRAFT`)                                 | The product code already exists in the org, or two new products in the same draft share a code.                                                                                                                                                                                                                                                             | Pick a different code, or use the offered link to reuse the existing product.                                                       |
| A feature's selection rule can never be satisfied (`MIN_UNSATISFIABLE`)                                            | `MinSelections__c` exceeds the number of options actually in the feature.                                                                                                                                                                                                                                                                                   | Lower the minimum, or add more options.                                                                                             |
| An option search result is grayed out and tagged **Bundle**                                                        | That product is itself a bundle header — the UI blocks adding it as an option before the nesting validator even runs.                                                                                                                                                                                                                                       | Pick a non-bundle product, or add that bundle's own options to this one directly.                                                   |

## Limits and gotchas

- **Two levels, no exceptions.** There is no `MaxDepth__c` setting — depth
  is always exactly 2 (header → options). This is a deliberate product
  decision (locked by Venkat during DDCPQ-BB-07), not a configurable limit.
- **A header with no `BundleDefinition__c`, or a Draft/Inactive one, is
  always RollUp** — changing `Status__c` to `Inactive` is a safe one-field
  rollback if a pricing-mode change misbehaves.
- **`AllocationWeight__c` is stored but not applied.** The engine does not
  yet spread a header's price across its components on a relative
  standalone-selling-price basis (ASC 606) — the field exists for
  reporting/export today.
- **Discounts never see a RollUp header** — a volume tier, system discount
  or margin floor keyed only off the header's own quantity still has to be
  authored against the options, because the header itself never enters the
  pricing pipeline in that mode.
- **The rate-plan gate still applies to `Priced` and blank-role options**
  under every pricing mode. Only `Included`/`Informational` are exempt.
  Blank reads as charged, not as "not yet decided."
- **Match with Other Option resolves in topological order, not declaration
  order** — define the chain in any order; the resolver sorts it, and only
  a genuine cycle is refused.
- **The configurator's own quantity input only appears when the rule
  permits editing** (`QuantityEditable__c`); on a locked rule, the rep sees
  the computed chip with no way to change it from the UI.
- **Bundle Builder's new-product and new-plan creation is deferred to
  save** — canceling a bundle under construction leaves the catalogue
  exactly as it was, even if you filled in a new product's fields.
- **Old React Bundle Builder (`ui/BundleBuilder.tsx`) is retired** — the
  LWC `cpqBundleBuilder` replaced it (DDCPQ-BB-01) and is what ships in
  0.1.0-9. Any doc or memory describing the three-step wizard, "Edit in
  Builder" button, or `batchUpdateBundle`/`batchCreateBundle` as separate
  admin-app actions is describing the pre-rebuild UI; the underlying Apex
  (`CpqAdminController.batchCreateBundle`/`batchUpdateBundle`) is still the
  save path underneath the new LWC, so the data model notes still apply.
- **Visual collapse / sibling fields are a hybrid-charge feature, not a
  bundle feature** — do not expect `SiblingGroupKey__c` to be set on an
  ordinary bundle option; it stays null unless that option is itself a
  hybrid product.

## Questions testers ask

**Q: My RollUp bundle's header shows $999 even though I never gave it a
rate plan — is that a bug?**
No. A RollUp header is never rated, so it just echoes its own Pricebook
list price untouched. If that number should be $0, zero the Pricebook
entry (and expect the `ROLLUP_HEADER_HAS_PRICE` warning until you do).

**Q: I set an option to Included but it still shows a price in the cart.**
Check its `PricingRole__c` is exactly `Included` (not blank, not
`Informational` unless that's what you meant), and that the bundle's
`PricingMode__c` is `HeaderPriced` or `Mixed` — Included has no special
effect under `RollUp`, since the option is never covered by anything there.

**Q: Can I nest a bundle inside a bundle "just for this one case"?**
No — it is refused outright (`BUNDLE_NESTING_REFUSED`), not a soft warning.
Flatten the inner bundle's options into the outer one, or sell it as a
separate line on the quote.

**Q: Why did the quantity I typed in the configurator get overwritten?**
Because the option's `QuantityRule__c` isn't `Static` — the server always
recomputes a ruled quantity and wins unless the option is also marked
`QuantityEditable__c` and you used the stepper.

**Q: Does a bundle's `AllocationWeight__c` actually change the price?**
No, not yet. It is captured for revenue-recognition reporting only; the
pricing engine does not read it.

**Q: What is `SourceBundle__c` actually pointing at — the header line, or
the header product?**
The **product**. Use `ParentLine__c` when you need the specific header
**line** (for a quote with the same bundle added more than once).

**Q: My new option product's code got rejected even though I've never seen
it before.**
Check whether another new product _in the same bundle draft_ already used
that code (`NEW_PRODUCT_CODE_DUPLICATED_IN_DRAFT`) before assuming it's a
clash with existing catalogue data (`NEW_PRODUCT_CODE_TAKEN` — the Checks
panel offers a link to reuse the existing product in that case).

**Q: Is the old three-step Bundle Builder wizard still around anywhere?**
No — 0.1.0-9 ships the single-workspace LWC described above. If a doc or
screenshot shows separate Identity/Build/Review steps, it predates the
DDCPQ-BB-01 rebuild.
