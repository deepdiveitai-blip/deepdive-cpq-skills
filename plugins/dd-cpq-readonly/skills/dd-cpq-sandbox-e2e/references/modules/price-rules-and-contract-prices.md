# Price rules and contract prices

> Installed build 0.1.0-9 · Verified 2026-10-06 in an installed org (cpq-pkg): Absolute → $150, PctOfList 90% of $200 → $180, Adjustment −$35 → $165, Variable (Constant 175) → $175, lower Priority wins over higher even on the same product, a contract price of $160 plus a 90% PctOfList rule → $144 (the rule bases off the contract price, not the list price).

## What it is for

Two different ways to change what a line charges before discounts run:

- **Price rules** (`DDCPQ__PricingMatrix__c` → `DDCPQ__PricingRule__c`) are
  deal-shaped: "when this quote looks like X, this product's list price is
  Y." They run for every quote that matches, on every product they target.
- **Contract prices** (`DDCPQ__ContractPriceRule__c`) are account-shaped:
  "this one customer pays this price for this product," independent of any
  deal condition. They exist because a negotiated price is a fact about the
  account, not a rule about the deal.

Both feed **Step 9** of the engine pipeline and both write into
`DDCPQ__DerivedListPrice__c` — the number the waterfall calls "Derived List
Price," one step before system/volume discounts stack on top of it.

**Variables** (`DDCPQ__Variable__c`) are a third piece that price rules (and
other rules) can reach into: a single named value — computed, constant, or
looked up — that any rule's condition or price can reference instead of a
literal.

## How it works

### Pipeline order

```
PricebookEntry list price
  → Contract Price (DDCPQ__ContractPriceRule__c) — replaces the list price
    as the seed for the next step, if a rule matches
  → Price Rule (DDCPQ__PricingRule__c) — computes DerivedListPrice__c from
    that seed, if a rule matches
  → (no match at either step = pass through unchanged)
```

Contract price runs **first**. When it matches, its override price becomes
the base that a price rule's `PctOfList` or `Adjustment` math works from —
**not** the pricebook list price. Verified: list $200, contract override
$160, a 90% `PctOfList` rule → **$144** (0.90 × 160), not $180 (0.90 × 200).

### Price rule math — every `DDCPQ__PriceType__c`

| Price Type (label)                                  | Math                                                                | Verified                            |
| --------------------------------------------------- | ------------------------------------------------------------------- | ----------------------------------- |
| `Absolute` ("A fixed price")                        | `DerivedListPrice = PriceValue`                                     | list $200, rule value 150 → **150** |
| `PctOfList` ("A % of list (or of contract price)")  | `DerivedListPrice = seed × PriceValue / 100`                        | list $200, rule value 90 → **180**  |
| `Adjustment` ("List price plus or minus an amount") | `DerivedListPrice = seed + PriceValue` (negative subtracts)         | list $200, rule value −35 → **165** |
| `Variable` ("The value of a Variable")              | `DerivedListPrice = resolved value of DDCPQ__PriceValueVariable__c` | Constant Variable = 175 → **175**   |

`PriceValue__c` is ignored when `PriceType__c = Variable`; only
`PriceValueVariable__c` matters then. If the named Variable cannot resolve
(inactive, missing dependency, resolver error), the engine falls back to the
pricebook/contract seed and marks the line `VariableUnresolved` — the
waterfall note says so; it never throws and blocks the whole quote.

### Which rule wins — precedence

Every active `DDCPQ__PricingRule__c` across **every** Active
`DDCPQ__PricingMatrix__c` is pooled into one candidate list for the product
being priced — the matrix a rule belongs to only decides whether the rule
is in play (its `Status__c` must be Active), not which rule wins. Verified:
two Active rule sets ("PR Set A", "PR Set B"), each holding one rule for
the same product — the pool, not the set, decides.

Within that pool, the winner is picked in this order:

1. **Most specific target wins.** A rule naming the product directly beats
   one that covers it through conditions or a Product Group, which beats
   one that covers "every product."
2. **Within the same specificity, lowest `Priority__c` (Evaluation Order)
   wins** — 1 runs before 2. A rule with no Priority value ranks last.
3. The winning rule's criteria must still hold (see Conditions, below). The
   **first candidate (by rules 1–2) whose criteria hold wins** — this is a
   first-match pick, not "all matching rules combine."

Verified: two catch-all rules target the same product — Priority 20 (value 999) and Priority 10 (value 155). Result: **155** — the lower Priority
number wins even though its rule was inserted after the other.

No match at all → the line keeps the seed price (pricebook or contract)
unchanged; the waterfall stage is `PricebookFallback`.

### Currency

`Currency__c` on a `DDCPQ__PricingRule__c` is blank by default, meaning the
rule applies to every currency. A non-blank value (e.g. `USD`) scopes the
rule to quotes in that currency only — a quote in another currency skips
the rule as if it didn't exist, it is not an error.

### Conditions — the shared 5-field schema, and the multi-condition upgrade

Every price rule carries the shared criteria fields (hard rule 5):
`FieldApiName__c`, `Operator__c`, `Value__c`, `ValueType__c`,
`Priority__c`. Blank `FieldApiName__c`/`Operator__c` = catch-all, always
matches.

`Conditions__c` is the advanced form Rules Studio writes when a rule needs
**more than one** condition, as JSON:

```json
{
  "logic": "1 AND (2 OR 3)",
  "conditions": [
    {
      "n": 1,
      "field": "Account.Industry",
      "op": "equals",
      "value": "Technology",
      "type": "string"
    },
    {
      "n": 2,
      "field": "Account.Name",
      "op": "equals",
      "value": "PR Test Co",
      "type": "string"
    },
    {
      "n": 3,
      "field": "Account.Name",
      "op": "equals",
      "value": "Nobody",
      "type": "string"
    }
  ]
}
```

When `Conditions__c` is filled, it **replaces** the single flat
field/operator/value — they are ignored. Blank `logic` means every
condition must hold (implicit AND). Verified: the JSON above against
Account Industry = Technology, Name = "PR Test Co" evaluates
`true AND (true OR false)` = **true**, and the rule's $120 Absolute price
applied.

Operators: `equals`, `not-equals`, `in`, `not-in`, `greater`, `less`,
`greater-or-equal`, `less-or-equal`, `between`, `contains`, `starts-with`,
`includes` (multi-select picklist), `is-null`, `is-not-null`. A condition
that cannot be read (bad literal, field missing from context) evaluates to
**false** — it never throws and never blocks the quote.

### Target — `Target__c` vs `TargetProduct__c`

`TargetProduct__c` is the simple single-product lookup. `Target__c` is the
richer JSON Rules Studio writes when you pick anything beyond one product:

```json
{"mode":"All"}
{"mode":"Products","products":["01t…","01t…"]}
{"mode":"Criteria","criteria":{"logic":"1","conditions":[{"n":1,"field":"Product.Family","op":"equals","value":"Collaboration","type":"picklist"}]}}
{"mode":"Group","groupId":"a0X…"}
```

A blank `Target__c` falls back to `TargetProduct__c`. For `PricingRule__c`
specifically, a **blank `TargetProduct__c` with a blank `Target__c` matches
nothing** (hard rule: a Price Rule with no product never applies) — unlike
some other rule objects where blank means "every product." Any mode can
carry `"exclude":[ids]` to carve out specific products.

### Contract prices — precedence inside `ContractPriceRule__c` itself

When more than one Active contract price rule could cover a line, the
winner is: the rule naming the account is required (`TargetAccount__c`
always required); among those, the one naming the product most precisely
wins (same 3/2/1 specificity as price-rule Target), then the **later**
`StartDate__c` (nulls lose to a real date), then the lower record Id as a
final tiebreak. `StartDate__c`/`EndDate__c` are inclusive windows; either
left blank means open-ended on that side. **Bundle parents are skipped** —
a contract override never applies to a bundle parent line, only to its
children, so there is no ambiguity about whether an override "includes"
the bundle's options.

### 2D matrix — a different "matrix," do not confuse the two

`docs/DD_CPQ_2D_MATRIX_TRAIL.html` is about **Charge Profiles** on a Rate
Plan (Charge Type × Pricing Model — Recurring/Usage/Overage/OneTime ×
Flat/PerUnit/Tiered/Volume/Package). That is a **different** feature,
covered in [pricing-models.md](pricing-models.md). `DDCPQ__PricingMatrix__c`
in this module is unrelated — it is just a named container ("Rule Set") for
Price Rules, with no grid or two axes of its own. If a tester lands on the
2D matrix trail looking for Price Rules, point them back here.

## Objects and fields

| Object label (API name)                      | Field                                                                                           | Meaning / values                                                                                                                                  |
| -------------------------------------------- | ----------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------- |
| Price Rule Set (`PricingMatrix__c`)          | `Status__c`                                                                                     | Draft / Active / Inactive. Only Active rule sets contribute rules to pricing.                                                                     |
| Price Rule (`PricingRule__c`)                | `Matrix__c`                                                                                     | Master-detail to its Price Rule Set.                                                                                                              |
|                                              | `TargetProduct__c`                                                                              | "Applies To Product" — the simple single-product form.                                                                                            |
|                                              | `Target__c`                                                                                     | "Products Covered" — JSON for All/Products/Criteria/Group + exceptions; blank falls back to `TargetProduct__c`.                                   |
|                                              | `PriceType__c`                                                                                  | "Sets List Price To" — `Absolute` / `PctOfList` / `Adjustment` / `Variable`.                                                                      |
|                                              | `PriceValue__c`                                                                                 | "Value (Price or %)" — the fixed price, the percent, or the +/− amount. Ignored when `PriceType__c = Variable`.                                   |
|                                              | `PriceValueVariable__c`                                                                         | "Price Value Variable" — lookup to an **Active** `Variable__c`; used only when `PriceType__c = Variable`.                                         |
|                                              | `FieldApiName__c` / `Operator__c` / `Value__c` / `ValueType__c`                                 | "Condition Field" / "Condition" / "Condition Value" / "Value Type" — the single flat condition. Ignored when `Conditions__c` is filled.           |
|                                              | `Conditions__c`                                                                                 | "Conditions (Advanced)" — multi-condition JSON with AND/OR logic.                                                                                 |
|                                              | `Priority__c`                                                                                   | "Evaluation Order" — lower runs first; blank runs last.                                                                                           |
|                                              | `Currency__c`                                                                                   | ISO code; blank = every currency.                                                                                                                 |
| Contract Price Rule (`ContractPriceRule__c`) | `TargetAccount__c`                                                                              | The account this negotiated price belongs to. Always required in practice.                                                                        |
|                                              | `TargetProduct__c` / `Target__c`                                                                | Same product-targeting pair as Price Rule.                                                                                                        |
|                                              | `OverridePrice__c`                                                                              | The negotiated price — required.                                                                                                                  |
|                                              | `StartDate__c` / `EndDate__c`                                                                   | Inclusive validity window; blank = open on that side.                                                                                             |
|                                              | `Status__c`                                                                                     | Draft / Active / Inactive.                                                                                                                        |
|                                              | `Conditions__c`                                                                                 | Optional WHEN conditions on top of account + product + dates.                                                                                     |
| Variable (`Variable__c`)                     | `ApiName__c`                                                                                    | The name rules reference, e.g. in `Variable.PR_Negotiated_Rate`.                                                                                  |
|                                              | `Type__c`                                                                                       | `Aggregate` (SOQL aggregate) / `Formula` (combines other variables) / `ApexMethod` (registered `VariableResolver`) / `Constant` (static literal). |
|                                              | `OutputType__c`                                                                                 | Number / String / Boolean / Date / Id — what shape the resolved value takes.                                                                      |
|                                              | `Scope__c`                                                                                      | Quote / Account / Opportunity / Global — the implicit record the variable is about.                                                               |
|                                              | `ConstantValue__c`                                                                              | The literal, when `Type__c = Constant`.                                                                                                           |
|                                              | `AggregateFunction__c` / `AggregateField__c` / `SourceObject__c`                                | Sum/Avg/Max/Min/Count/CountDistinct/First over a SOQL filter, when `Type__c = Aggregate`.                                                         |
|                                              | `FormulaExpression__c`                                                                          | Arithmetic over other Variables, when `Type__c = Formula`.                                                                                        |
|                                              | `ApexClassName__c` / `ApexParameters__c`                                                        | The registered `VariableResolver` class and its JSON params, when `Type__c = ApexMethod`.                                                         |
|                                              | `CacheStrategy__c`                                                                              | `PerCall` / `Transaction` / `None` — how aggressively a resolved value is reused within one pricing run.                                          |
|                                              | `Status__c`                                                                                     | Draft / Active / Inactive. Only Active variables resolve; a rule pointing at a non-Active variable is blocked by a lookup filter at save time.    |
| Variable Filter (`VariableFilter__c`)        | `Variable__c` / `FieldApiName__c` / `Operator__c` / `Value__c` / `ValueType__c` / `Priority__c` | The WHERE clause for an Aggregate Variable. Filters sharing a `Priority__c` are AND'd; different priorities are OR'd together.                    |

## Build it (tester)

Scenario: a $200 product gets a negotiated $160 contract price for one
account, and a Price Rule gives a 10% discount off whatever the account
pays — proving the rule stacks on the contract price, not the list price.

1. **Product.** Products tab → New → Name `Widget Pro`, Active ticked →
   Save. Related tab → Price Books → Add Standard Price → **200** → Active
   → Save.
2. **Account.** `Negotiated Co`, any Industry.
3. **Rate plan** so the line can commit later: Rate Plan Editor tab → open
   Widget Pro → add a plan → Recurring, Per unit, Monthly, price 200 →
   Active → Save (Stage 6 of the golden path, same steps).
4. **Contract price.** Rules Studio → left nav, **Pricing** group →
   **Contract Prices** → **New rule…** → For: pick `Widget Pro` → Account:
   `Negotiated Co` → Then: Override Price **160** → Status **Active** →
   Save changes.
5. **Price rule.** Rules Studio → **Pricing** group → **Price Rules** →
   **New rule…** (or **+ Add rule** inside an existing Active set) →
   For: `Widget Pro` → Then: **Sets List Price To** = "A % of list (or of
   contract price)" (`PctOfList`), **Value (Price or %)** = **90** →
   leave When empty (catch-all) → Save changes. New rule sets save as
   **Draft** — open the rule set and flip its status to **Active**, or the
   rule is ignored no matter how correct it looks.
6. **Quote.** Opportunity on `Negotiated Co` → New Quote → Configure
   Products → Add `Widget Pro`, quantity 1.
7. Read the line's price and click it to open the waterfall: **List $200
   → Contract Price $160 → Derived List Price $144** (90% of 160, not of
   200).

## Check it (Claude)

```bash
sf data query -o <alias> -q "SELECT DDCPQ__OverridePrice__c, DDCPQ__Status__c, DDCPQ__TargetAccount__r.Name, DDCPQ__TargetProduct__r.Name FROM DDCPQ__ContractPriceRule__c WHERE DDCPQ__TargetProduct__r.Name = 'Widget Pro'"
sf data query -o <alias> -q "SELECT DDCPQ__Matrix__r.Name, DDCPQ__Matrix__r.DDCPQ__Status__c, DDCPQ__PriceType__c, DDCPQ__PriceValue__c, DDCPQ__Priority__c FROM DDCPQ__PricingRule__c WHERE DDCPQ__TargetProduct__r.Name = 'Widget Pro'"
sf data query -o <alias> -q "SELECT Product2.Name, DDCPQ__DerivedListPrice__c, DDCPQ__NetPrice__c FROM QuoteLineItem WHERE Quote.Name = '<your quote name>'"
```

Expect the contract rule Active with `OverridePrice__c = 160`, the price
rule's matrix Active with `PriceType__c = PctOfList`, `PriceValue__c = 90`,
and the line's `DerivedListPrice__c = 144`.

Read-only price preview (no commit), useful when the cart disagrees with
your math — fill in real Ids:

```json
{
  "quoteId": "<0Q0…>",
  "selections": [
    { "localKey": "a", "productId": "<Widget Pro 01t…>", "quantity": 1 }
  ]
}
```

```bash
sf api request rest "services/apexrest/DDCPQ/dd/v1/price?includeWaterfall=true" -o <alias> --method POST --body @body.json
```

### Variables — build and test one

1. **App Launcher → "Variable"** tab (standard object tab; Variables are
   not authored inside Rules Studio). New → Variable Name `Negotiated
Rate`, API Name `Negotiated_Rate`, **Type** = "Constant (static
   literal)", **Output Type** = Number, **Scope** = Global, **Constant
   Value** = `175`, **Status** = Active → Save.
2. In a Price Rule's **Then**, set **Sets List Price To** = "The value of
   a Variable" → **Price Value Variable** picks `Negotiated Rate` (the
   lookup only offers Active variables — a filter blocks anything else
   with "Only an Active variable can be used here. Activate the variable
   first.").
3. Price a line for that product: Derived List Price = **175**, the
   Variable's resolved value, regardless of the pricebook list price.

Variables of type Aggregate, Formula and ApexMethod resolve live from SOQL,
other Variables, or a registered `VariableResolver` Apex class — the same
Constant mechanics apply once resolved; see `docs/EXTENDING.md` if a
feature needs custom Apex (that is a developer extension, not something
built from the Salesforce UI).

## Expected numbers

| Scenario                 | Inputs                                             | Expected          |
| ------------------------ | -------------------------------------------------- | ----------------- |
| Absolute                 | list 200, rule value 150                           | **150**           |
| PctOfList                | list 200, rule value 90                            | **180**           |
| Adjustment               | list 200, rule value −35                           | **165**           |
| Variable (Constant)      | Constant Value 175                                 | **175**           |
| Priority tiebreak        | Priority 10 → 155, Priority 20 → 999, same product | **155**           |
| Contract price only      | list 200, contract override 170, no price rule     | **170**           |
| Contract + PctOfList 90% | list 200, contract 160, rule 90%                   | **144**           |
| Multi-condition AND/OR   | `1 AND (2 OR 3)`, all three true/true/false        | matched → **120** |

## Troubleshooting

| Symptom                                                                  | Cause                                                                                                                  | Fix                                                                                                                                                             |
| ------------------------------------------------------------------------ | ---------------------------------------------------------------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Price stays at pricebook list                                            | every Price Rule Set is Draft, or the rule's Target doesn't cover this product                                         | Rules Studio → Price Rules → open the set → Status Active; recheck Target                                                                                       |
| Price rule ignored, contract price also ignored                          | rule's `Currency__c` is set to a currency the quote isn't in                                                           | clear `Currency__c` or match it to the quote's currency                                                                                                         |
| Wrong rule wins when two could apply                                     | the losing rule has a higher `Priority__c` number, or is less specific (every-product vs. named-product)               | lower the winner's Priority number, or make its Target name the product directly                                                                                |
| `PctOfList` gives a bigger discount than expected                        | a Contract Price Rule already lowered the seed — the percent is being taken off the contract price, not the list price | check the ContractPrice waterfall stage before judging the PctOfList stage                                                                                      |
| Price rule with `PriceType = Variable` has no effect                     | the Variable is not Active, or its resolver/formula/aggregate returned nothing                                         | waterfall note reads `VariableUnresolved`; check the Variable's Status and its Apex/SOQL/formula inputs                                                         |
| "Only an Active variable can be used here. Activate the variable first." | tried to pick a Draft/Inactive Variable in `Price Value Variable`                                                      | activate the Variable first, then pick it                                                                                                                       |
| Multi-condition rule never matches                                       | `Conditions__c` logic string references a condition number that doesn't exist, or the field path is wrong              | Rules Studio validates filter logic on save ("Filter logic does not use condition N" / "Filter logic … is invalid") — fix the numbers to match `conditions[].n` |
| Contract price never applies                                             | the line is a bundle parent                                                                                            | contract prices are skipped on bundle parents by design; check the child lines instead                                                                          |
| Variable cannot be deactivated/deleted                                   | it is still referenced by a live rule or a bundle option's quantity                                                    | the error names the holder; remove or repoint that reference first                                                                                              |

## Limits and gotchas

- A `PricingRule__c` with no `TargetProduct__c` and no `Target__c` matches
  **nothing** — unlike some other DD CPQ rule objects, a blank target is
  not "every product" here.
- `Conditions__c`, when filled, fully replaces the single flat condition
  fields — you cannot combine the old single-condition fields with the new
  multi-condition JSON on the same rule.
- Rule precedence is per-product, computed fresh on every reprice; it is
  not cached, so changing a Priority value takes effect on the next price
  call with no cache to clear.
- `docs/DD_CPQ_2D_MATRIX_TRAIL.html` is about Rate Plan Charge Profiles,
  not about `PricingMatrix__c` — see "2D matrix" above; do not expect this
  module's objects to show up there.
- Installed build 0.1.0-9 (commit 31444bb) still has the in-Rules-Studio
  rule tester ("Try it on a quote" → "Test on a quote," which prices a real
  quote with the draft rule and undoes it, nothing saved). **DDCPQ-68
  removes this panel** in a later build — if a tester's build no longer
  shows "Try it on a quote," that is expected, not a bug.
- Waterfall rule-attribution (naming which rule produced a stage, DDCPQ-69)
  and waterfall discount-percent precision (DDCPQ-112) are both later than
  0.1.0-9 — the installed build's waterfall shows the stage value and a
  `sourceRuleId`, not a human-readable rule name.

## Questions testers ask

**Q: I set a 90% PctOfList rule and expected 10% off list, but got less
than that.**
A: Check the ContractPrice waterfall stage first — if this account has a
negotiated price, the 90% is taken off that price, not off the pricebook
list.

**Q: I have two Active Price Rule Sets with a rule each for the same
product — which one wins?**
A: Neither "wins" by being the newer or more recently edited set. All
Active sets' rules pool together; the most specific Target wins first,
then the lowest Priority number, then the first whose conditions hold.

**Q: Why does my Price Rule with no conditions never get beaten by a more
specific one on the same product?**
A: It should — a rule naming the product directly outranks a catch-all
"every product" rule regardless of Priority. If a catch-all is winning
over a named rule, check that the named rule's Target actually resolved
(bad Id, inactive referenced product, or a JSON mode typo falls back to
`invalid`, which scores 0 and never wins).

**Q: Can I give a Price Rule an absolute negative price?**
A: Nothing in the engine clamps a negative Adjustment or Absolute value —
it will produce a negative Derived List Price, which then flows into
downstream discounts unmodified. Treat that as a data-entry mistake, not a
feature.

**Q: Do Variables work in contract price rules too, or only Price Rules?**
A: A Variable only drives the _value_ through `PriceValueVariable__c` on a
Price Rule. A Variable can appear inside any rule's **conditions**
(`Variable.<ApiName>` in the context map) regardless of rule type,
including Contract Price Rules' `Conditions__c` — but Contract Price
Rules have no "price from a Variable" mode; their price always comes from
the flat `OverridePrice__c`.

**Q: Where do I manage Variables — Rules Studio or somewhere else?**
A: The standard **Variable** tab (App Launcher). Rules Studio's left nav
does not list Variables; you pick an existing Active Variable from a
lookup when a rule needs one.

**Q: My Contract Price Rule has a Start Date in the future — why is it
applying today?**
A: It shouldn't. Re-check the date: the window is inclusive, and a quote
priced today only matches a rule whose Start Date is today or earlier (or
blank) and whose End Date is today or later (or blank).
