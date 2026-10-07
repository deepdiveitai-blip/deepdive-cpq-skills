# Multi-currency

> Installed build 0.1.0-9 · Verified 2026-10-07 in cpq-pkg: with EUR added as
> a second active currency (0.92 conversion rate), an MC Seat line priced on
> a USD quote at **100/unit**; the same product on a EUR quote picked up its
> EUR-native Rate Plan price of **92**, then a EUR-scoped price rule
> overrode it to **85** (totalNet 850, totalArr 10,200 at qty 10) — while
> the USD quote never saw that rule at all.

## What it is for

DD CPQ can quote in more than one currency, but it does **not do foreign-
exchange conversion anywhere**. Every currency a product sells in needs its
own hand-authored price: its own `PricebookEntry`, and — if you want
anything past a flat list price — its own `RatePlan__c` row with its own
native `UnitPrice__c`. The engine's job is narrower than "convert $100 to
€92": it resolves the quote's currency once, then **filters** which
Pricebook row, which Rate Plan, and which of the six currency-scoped rule
types are even allowed to fire on that quote. A single-currency org (the
default) never touches any of this — every currency-aware path short-
circuits to its pre-multi-currency behavior.

This module covers: enabling multi-currency and adding a new active
currency, how the engine resolves `Quote.CurrencyIsoCode`, why a Rate Plan's
own price wins over the Pricebook list price, the `Currency__c` field shared
by six rule objects (and the one rule type that does **not** have it), the
USD fallback when a product has no plan in the quote's currency, the
`clone-to-currency` write action, and rounding.

**Not this module:** the pricing models themselves
([pricing-models.md](pricing-models.md)), price rules and contract prices
([price-rules-and-contract-prices.md](price-rules-and-contract-prices.md)),
volume/system discounts ([discounts.md](discounts.md)).

## How it works

### Turning it on — self-service, irreversible

Multi-currency is a company-wide setting, not a DD CPQ setting:

1. Setup → Quick Find **"Company Information"** → **Edit**.
2. Tick **Activate Multiple Currencies** → **Save**.

This no longer needs a Salesforce Support case — it is a checkbox any admin
can tick in a Developer Edition org or sandbox. **It cannot be turned off
once saved.** Your org's current default currency becomes the **corporate
currency** and every existing amount field is assumed to already be in that
currency; nothing gets converted retroactively.

Adding a second currency is separate and additive, and is what this module
actually tests:

3. Setup → Quick Find **"Manage Currencies"** → **Active Currencies** → **New**.
4. Pick the ISO code (e.g. **EUR**), set a **Conversion Rate** relative to
   the corporate currency (e.g. `0.92`), tick **Active** → **Save**.

Nothing here is DD CPQ metadata — it is the same screen used for every
Salesforce standard currency field (Opportunity.Amount, PricebookEntry,
etc). cpq-pkg already has multi-currency on with only USD active; this
module's own verification added EUR as a new active currency and left USD
as the untouched corporate currency.

### Resolving the quote's currency

`CpqEngine.resolveQuoteCurrency(quoteId)` reads `Quote.CurrencyIsoCode` once
per pipeline run (cached in `EngineCache`), but only after checking
`ChargeProfileService.isMultiCurrencyOrg()` — a cached
`Schema.getGlobalDescribe().get('CurrencyType') != null` describe check.
On a single-currency org this returns `null` immediately and every
currency-aware filter downstream treats `null` as "match everything" — so
nothing in this module changes behavior there.

**A Quote's currency cannot differ from its Opportunity's currency.**
Verified: updating a Quote's `CurrencyIsoCode` to `EUR` while its
Opportunity is still `USD` is refused by the platform itself with:

> _"Your quote currency must be the same as your opportunity currency. We
> updated your quote currency to match.: Currency ISO Code"_

To quote in a second currency you need an Opportunity already in that
currency — set `Opportunity.CurrencyIsoCode` to EUR when you create it, not
after.

### Pricebook list price vs. Rate Plan price — which one wins

The pipeline seeds every line's list price from the **currency-matching**
`PricebookEntry.UnitPrice` first (`CpqEngine.loadPricebookEntries` adds
`AND CurrencyIsoCode = :currencyIsoCode` once it knows the quote's
currency — without that filter, multi-currency orgs have one
`PricebookEntry` row per (Pricebook, Product, Currency) and an unfiltered
query picks a non-deterministic winner, which is exactly the "price book
entry currency code is different than the one assigned to the Quote" DML
error at commit).

But if the product has an **Active Rate Plan authored in that same
currency**, that plan's own `UnitPrice__c` **overrides** the Pricebook seed
before the `PricebookList` waterfall stage is even recorded — the same
override pricing-models.md documents for single-currency orgs, just scoped
to the matching currency. Verified: with the EUR `PricebookEntry` at 92 and
the EUR `RatePlan__c` at 92, `PricebookList` showed 92; changing only the
Rate Plan's `UnitPrice__c` to 77 (Pricebook unchanged) changed `PricebookList`
to 77. **The Rate Plan price is what the customer is actually charged; the
Pricebook entry mainly exists because the platform requires one to add the
product to the quote at all.**

### The USD fallback — silent, no conversion

`ChargeProfileService.loadActiveRatePlansByProductWithFallback(productIds,
primaryCurrency, fallbackCurrency, fallbackAppliedOut)` tries the quote's
own currency first; for any product with **no** Active plan in that
currency, it retries with the fallback currency (`'USD'`, hardcoded in
`CpqEngine`, not configurable) and uses that plan's price **as-is** — no
multiplication by a conversion rate. Verified: a product with only a USD
Rate Plan at `100`, priced on a EUR quote, returned `netPrice` **100**, not
92 and not any converted figure — a German customer would see a plain "100"
with no currency conversion, label, or warning anywhere in the response.

The fallback is tracked internally
(`CommitService.LineDraft.currencyFallbackApplied` /
`currencyFallbackFrom` / `currencyFallbackTo`) but **this build never
surfaces it** — not in the `price` REST response, not on the committed
`QuoteLineItem` (no such field exists there), not in any LWC. If a line
priced in the "wrong" currency, the only way to catch it today is to notice
the number looks like the wrong currency's price, or to check which
currency each `RatePlan__c` carries directly.

### The six currency-scoped rule objects — and the one that isn't

A shared `Currency__c` Text(3) field (added 2026-09-11, waterfall gap review
Theme 01) lives on:

`PricingRule__c` · `ContractPriceRule__c` · `PromotionRule__c` ·
`ChannelDiscountRule__c` · `RampSchedule__c` · `UsageTierMatrix__c` · plus
`VolumeDiscountMatrix__c` (the matrix header, not the tier row — a Volume
Discount's currency is set once for the whole ladder).

Rule: **blank `Currency__c` = catch-all, fires on every currency. A
specific ISO code fires only when it equals the quote's currency** (case-
insensitive). `CurrencyFilter.byCurrency(rules, quoteCurrency)` is the one
shared utility six of these loaders call; `PricingService.derive` filters
`PricingRule__c` itself with the identical blank-or-match logic inline.
Verified: a `PricingRule__c` with `Currency__c = 'EUR'` targeting MC Seat
fired on the EUR quote (`DerivedListPrice` 92 → **85**) and did **not** fire
on the USD quote for the same product (stayed at 100) — confirmed by the
`rules.rules[].ruleId` in the price response naming it only on the EUR run.

**`SystemDiscountRule__c` has no `Currency__c` field.** System discounts
(the percentage stack from eligibility like Account.Industry or
Opportunity.Type — see discounts.md) are **not** currency-scoped in this
build; a system discount fires on every quote regardless of currency,
because `CpqEngine.loadActiveSystemDiscountRules()` has no currency
parameter at all. If you need a discount that applies only in one market,
author it as a `PromotionRule__c` or `ChannelDiscountRule__c` instead —
both of those do carry `Currency__c`.

### Cloning a plan to a new currency

`CpqRatePlanWriteResource` exposes a `clone-to-currency` write action:
`{"action":"clone-to-currency","planId":"a0c…","targetCurrencyIsoCode":"EUR","newUnitPrice":18}`
duplicates one `RatePlan__c` row, stamps the new `CurrencyIsoCode`, applies
the new native price (or keeps the source's number if `newUnitPrice` is
omitted), appends `_EUR` to its `DisplayLabel__c`, and bumps `SortOrder__c`
by 1 so the clone sorts after its source. **This build's Rate Plan Editor
has no button for it** (its LWC has no `clone` or `currency` handler at
all) — it exists today only as a REST/Apex entry point for an admin or an
agentic client, not as a click a tester can make. Build a second-currency
plan by hand instead (see "Build it").

### Rounding — not currency-aware

The main waterfall rounds `NetPrice__c` to 2 decimal places with
`HALF_UP`, same for every currency
(`CpqEngine.normalizeNetPriceScale` / the `totalNet` aggregation) — there is
no branch anywhere on `CurrencyIsoCode` or on `CurrencyType.DecimalPlaces`.
A separate helper, `NormalizedRateService.round`, used for usage/MRR math,
rounds to 4 decimal places with a plan's own `RoundingMode__c`
(`HalfUp` / `HalfEven` / `Floor` / `Ceiling`) — also with no currency
branch. For every currency this build was tested against (USD, EUR) that is
invisible, because both use 2 decimal places natively. **It has never been
exercised against a 0-decimal currency like JPY** — `RatePlanMultiCurrencyTest`
only fixtures EUR/GBP, and its own tests skip entirely
(`DdCpqTestAccess.hasCurrencies` guard) unless the org already has those
currencies active. Treat a JPY quote as unverified territory; expect
`150.00`-style output rather than a whole-yen `150`.

## Objects and fields

| Object label (API name)                            | Field             | Meaning / values                                                                                                                                                                                      |
| -------------------------------------------------- | ----------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Rate Plan (`RatePlan__c`)                          | `CurrencyIsoCode` | Standard system field, appears automatically once the org is multi-currency — not a DD CPQ field. The currency this plan's `UnitPrice__c` is natively denominated in.                                 |
|                                                    | `UnitPrice__c`    | The native price in that plan's own currency. Never converted; authored per currency.                                                                                                                 |
| Pricing Rule (`PricingRule__c`)                    | `Currency__c`     | Text(3) ISO code. Blank = catch-all. Filtered inline in `PricingService.derive`.                                                                                                                      |
| Contract Price Rule (`ContractPriceRule__c`)       | `Currency__c`     | Same semantics, filtered via `CurrencyFilter`.                                                                                                                                                        |
| Promotion Rule (`PromotionRule__c`)                | `Currency__c`     | Same semantics.                                                                                                                                                                                       |
| Channel Discount Rule (`ChannelDiscountRule__c`)   | `Currency__c`     | Same semantics.                                                                                                                                                                                       |
| Ramp Schedule (`RampSchedule__c`)                  | `Currency__c`     | Same semantics.                                                                                                                                                                                       |
| Usage Tier Table (`UsageTierMatrix__c`)            | `Currency__c`     | Same semantics.                                                                                                                                                                                       |
| Volume Discount Matrix (`VolumeDiscountMatrix__c`) | `Currency__c`     | Lives on the **matrix header**, applies to every tier row underneath it.                                                                                                                              |
| System Discount Rule (`SystemDiscountRule__c`)     | —                 | **No `Currency__c` field exists.** Always fires, every currency.                                                                                                                                      |
| Pricebook Entry (standard)                         | `CurrencyIsoCode` | One row per (Pricebook, Product, Currency). Required for the product to be addable to the quote at all; its `UnitPrice` is only what actually charges the customer when no matching Rate Plan exists. |
| Quote (standard)                                   | `CurrencyIsoCode` | Must equal its Opportunity's `CurrencyIsoCode`; cannot be changed independently once set.                                                                                                             |

## Build it (tester)

Do this after the golden path, on your own products — do not touch CRM
Suite Pro, Sales Cloud or the golden-quote data, and do not touch
`DD_CPQ_Rate_Plan_Settings__mdt` or any other org-wide setting.

1. **Setup → "Company Information" → Edit.** If **Activate Multiple
   Currencies** is unticked, tick it and Save — read the warning on screen
   first; this cannot be undone. (If your org already shows an Active
   Currencies related list, multi-currency is already on — skip this step.)
2. **Setup → "Manage Currencies" → Active Currencies → New.** ISO code
   **EUR**, any Conversion Rate (e.g. `0.92`), **Active** ticked → Save. Do
   **not** touch the Corporate currency or any existing currency's rate.
3. **Products tab → New** → Name it `MC Seat`, **Active** ticked → Save.
4. Related tab → **Price Books** → **Add Standard Price**: List Price
   `100`, Currency **USD**, Active → Save. Click **Add Standard Price**
   again: List Price `92`, Currency **EUR**, Active → Save. You now have
   two Pricebook Entries on the same product, one per currency.
5. **Rate Plan Editor tab** → open `MC Seat` → **Start this product's
   pricing** → Shape: Charge type **Recurring**, Pricing model **Per
   Unit**, Billing schedule **Monthly**, Unit Price `100` → **Status
   Active** → Save.
6. **App Launcher → "Rate Plans"** (the plain object tab, not the Rate Plan
   Editor) → find the plan you just saved → open its record page → edit
   the standard **Currency** field to **EUR** and `Unit Price` to `92` →
   Save. This becomes your EUR-native plan on the same product; the USD
   plan from step 5 still exists as a separate row.
7. **Rules Studio tab** → new Price rule set `MC Pricing` → one rule
   targeting **MC Seat**, **Sets list price to** a fixed price **85**, no
   condition → **Activate the rule set**. Open the rule's currency field
   (if Rules Studio exposes it) or edit the `PricingRule__c` record
   directly and set **Currency** to **EUR**. Leaving it blank would make it
   fire on every currency, which defeats this test.
8. Build **Account** `MC Test Co`. Build **two Opportunities** on it: `MC
USD Opp` (currency USD, default) and `MC EUR Opp` — when creating it,
   set its **Currency** field to **EUR** at creation time (you cannot
   change it after a Quote exists on it).
9. **New Quote** on each Opportunity: `MC USD Quote` and `MC EUR Quote`.
   Each quote inherits its Opportunity's currency and cannot be changed to
   a different one.
10. **Configure Products** on `MC USD Quote` → add `MC Seat`, quantity
    `10`. Expect net price **100** (the EUR-only price rule never fires
    here).
11. **Configure Products** on `MC EUR Quote` → add `MC Seat`, quantity
    `10`. Expect list price **92** (your EUR Rate Plan), then **85** after
    the EUR price rule — read the waterfall to see both.

## Check it (Claude)

```bash
sf data query -o cpq-pkg -q "SELECT IsoCode, ConversionRate, IsActive, IsCorporate FROM CurrencyType"
sf data query -o cpq-pkg -q "SELECT DDCPQ__TargetProduct__r.Name, CurrencyIsoCode, DDCPQ__Status__c, DDCPQ__UnitPrice__c FROM DDCPQ__RatePlan__c WHERE DDCPQ__TargetProduct__r.Name = 'MC Seat'"
sf data query -o cpq-pkg -q "SELECT Product2.Name, CurrencyIsoCode, UnitPrice, IsActive FROM PricebookEntry WHERE Product2.Name = 'MC Seat'"
sf data query -o cpq-pkg -q "SELECT DDCPQ__TargetProduct__r.Name, DDCPQ__Currency__c, DDCPQ__PriceType__c, DDCPQ__PriceValue__c FROM DDCPQ__PricingRule__c WHERE DDCPQ__TargetProduct__r.Name = 'MC Seat'"
sf data query -o cpq-pkg -q "SELECT Name, CurrencyIsoCode FROM Opportunity WHERE Name LIKE 'MC %Opp'"
sf data query -o cpq-pkg -q "SELECT Name, CurrencyIsoCode FROM Quote WHERE Name LIKE 'MC %Quote%'"
```

Expect two `CurrencyType` rows (USD corporate, EUR active); two
`RatePlan__c` rows for MC Seat, one per currency; two `PricebookEntry` rows;
one `PricingRule__c` with `DDCPQ__Currency__c = 'EUR'`.

Read-only price preview (no commit), against the Ids you queried for your
own quote and product:

```json
{
  "quoteId": "<0Q0… your EUR quote>",
  "selections": [
    { "localKey": "s1", "productId": "<MC Seat 01t…>", "quantity": 10 }
  ]
}
```

```bash
sf api request rest "services/apexrest/DDCPQ/dd/v1/price?includeWaterfall=true" -o cpq-pkg --method POST --body @body.json
```

Look at `lines[0].waterfall` — the `PricebookList` stage should read your
EUR Rate Plan's price (92), and `DerivedListPrice` should show 85 with a
`sourceRuleId` pointing at your price rule. Run the same body against the
USD quote/Id and confirm `DerivedListPrice` stays 100 with no
`sourceRuleId`.

## Expected numbers

All verified 2026-10-07 in cpq-pkg via the read-only price preview, with
EUR newly active (0.92 conversion rate, USD still corporate) and MC Seat
carrying a USD Rate Plan (100), a EUR Rate Plan (92), and a EUR-scoped
price rule overriding to 85.

| Quote currency | Product                                                             | Qty | PricebookList | DerivedListPrice               | NetPrice   | Line total | ARR    |
| -------------- | ------------------------------------------------------------------- | --- | ------------- | ------------------------------ | ---------- | ---------- | ------ |
| USD            | MC Seat                                                             | 10  | 100           | 100                            | **100.00** | 1,000      | 12,000 |
| EUR            | MC Seat                                                             | 10  | 92            | **85** (rule `a0QE2…cvZUbMAM`) | **85.00**  | 850        | 10,200 |
| EUR            | MC Fallback Seat (USD-only plan, no EUR plan or EUR PricebookEntry) | 5   | 60            | 60                             | **60.00**  | 300        | 3,600  |

The third row is the USD fallback: a product authored only in USD, priced
on a EUR quote, returns the raw USD number with no conversion and no
warning anywhere in the response — see "How it works."

## Troubleshooting

| Symptom                                                                                                       | Cause                                                                                                          | Fix                                                                                                                                        |
| ------------------------------------------------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------ |
| Updating a Quote's Currency field fails: "Your quote currency must be the same as your opportunity currency…" | Standard Salesforce behavior, nothing to do with DD CPQ — a Quote always inherits its Opportunity's currency   | Set the Opportunity's currency correctly before creating the Quote; build a second Opportunity if you need a second currency               |
| Adding a product to a quote fails with a price book entry error at commit                                     | No active `PricebookEntry` exists for that product in the quote's currency                                     | Product2 → Price Books → Add Standard Price in that specific currency                                                                      |
| EUR quote shows the USD price, unconverted                                                                    | The product has no Active Rate Plan in EUR, so the engine silently fell back to its USD plan                   | Author a Rate Plan in the quote's currency (edit the Currency field on the Rate Plans tab record), or accept the fallback is expected here |
| A price/discount/promo rule you scoped to EUR also fires on USD quotes                                        | `Currency__c` was left blank (catch-all) instead of set to `EUR`                                               | Open the rule record directly and set Currency; Rules Studio may not expose this field in every rule type                                  |
| A "market-only" system discount fires on every currency anyway                                                | `SystemDiscountRule__c` has no `Currency__c` field — system discounts are never currency-scoped in this build  | Rebuild it as a Promotion or Channel Discount rule instead, both of which carry `Currency__c`                                              |
| Rate Plan Editor has no obvious way to make a second-currency version of a plan                               | No clone-to-currency button in this build's LWC                                                                | Create the new plan by hand: Rate Plan Editor for the shape, then the plain Rate Plans tab to set its Currency field                       |
| Two Rate Plans on the same product, one per currency, both show as "alternatives" / chip-picked in the cart   | `PlanGroupKey__c` matched on both rows (see pricing-models.md) — currency does not itself separate plan groups | Give each currency's plan a distinct `PlanGroupKey__c`, or rely on only one plan per product ever matching the quote's currency            |
| Enabling multi-currency is greyed out / missing in Company Information                                        | Some very old or restricted Developer Edition shapes do not show the checkbox                                  | Confirm the org edition supports it; this is a platform limitation, not DD CPQ                                                             |

## Limits and gotchas

- **No FX conversion exists anywhere in the engine.** Every number in this
  module came from a hand-authored native price. If the business needs
  actual rate-based conversion, that is unbuilt — track it as a gap, not a
  configuration task.
- The USD fallback currency is hardcoded (`'USD'`) in `CpqEngine` — not a
  Setting, not configurable per org.
- The fallback flag (`currencyFallbackApplied`) is computed but never
  surfaced: not in the `price`/`includeWaterfall` REST response, not on the
  committed `QuoteLineItem`, not in any LWC. A line quietly priced in the
  wrong currency looks identical to one priced correctly.
- `SystemDiscountRule__c` has no `Currency__c` — system discounts always
  fire regardless of quote currency. The other six rule/matrix types do
  respect it.
- Rounding is fixed at 2 decimal places, `HALF_UP`, for every currency —
  never tested against a 0-decimal currency like JPY in this build.
  `RatePlanMultiCurrencyTest`'s own suite skips its assertions entirely on
  any org that does not already have EUR/GBP active, which includes most
  fresh package installs — so this path has effectively zero executed
  coverage until a tester does exactly what this module does.
- `clone-to-currency` (`CpqRatePlanWriteResource`) exists only as a
  REST/Apex write action in this build — no Rate Plan Editor button calls
  it yet.
- DDCPQ-67's 2,000-line cart limit, DDCPQ-112's waterfall discount percent
  precision fix, and DDCPQ-69's rule-behind-each-step waterfall labels are
  not in this build; they land after commit 31444bb and do not change
  anything in this module.
- Activating multiple currencies is irreversible for the whole org. Never
  do this in a shared org without confirming with whoever else uses it.

## Questions testers ask

Q: If I set a EUR conversion rate, does DD CPQ use it to convert my USD
prices?
A: No. The conversion rate on `CurrencyType` is a standard Salesforce
field used by Salesforce reporting roll-ups (e.g. converting Opportunity
Amounts into the corporate currency for dashboards) — DD CPQ's own pricing
engine never reads it. Every DD CPQ price in a given currency has to be
authored natively in that currency.

Q: My product has a Rate Plan in USD and a Pricebook Entry in EUR with a
different number. Which one wins on a EUR quote?
A: Neither directly — the engine looks for an Active Rate Plan **in EUR**
first. Finding none, it falls back to the USD Rate Plan's price (not the
EUR Pricebook Entry's price) and charges that number unconverted on the EUR
quote. The EUR Pricebook Entry only matters if the product has no Rate
Plan in EUR at all and also none to fall back to (then the EUR Pricebook
Entry's own number prices the line).

Q: Why did my price rule fire on both my USD and my EUR test quotes?
A: Its `Currency__c` field is blank, which is read as "any currency" (catch-
all), not "unauthored." Set it explicitly if you want the rule scoped to
one market.

Q: Can I change a quote's currency after I've already added products?
A: You can try, but the platform refuses it the same way it refuses
mismatching it against the Opportunity — it is not a DD CPQ restriction.
Start a new Quote on an Opportunity already in the right currency instead.

Q: I don't see a Currency field anywhere when creating a Rate Plan in the
Rate Plan Editor. Did I miss a tab?
A: No — this build's Rate Plan Editor does not expose currency. Save the
plan first, then open it from the plain **Rate Plans** tab (not the
Editor) and set its standard Currency field there.

Q: Is there a way to see which currency each Rate Plan belongs to without
SOQL?
A: Product 360's Rate Plan list and the Rate Plan Editor's own price
previews render amounts using the plan's `currencyIsoCode` (its money
formatting adapts automatically), but neither currently displays the ISO
code itself as a visible label on the plan card in this build — if in
doubt, check the record's Currency field directly.
