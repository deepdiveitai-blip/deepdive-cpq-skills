# System discounts, volume tiers, promotions, channel discounts

> Installed build 0.1.0-9 · Verified 2026-10-06 in cpq-pkg: AllStack 10%+5% on a
> $200 line → $171.00; TakeBest with Percent 20% vs Fixed $30 → $160.00 (Percent
> wins); volume ladder 1–10 @5%, gap 10–29, 30+ @15% on a $100 line → qty 5 =
> $95.00, qty 15 = $100.00 (no tier), qty 30 = $85.00; promo code `DSSPRING26`
> (10%) → $90.00; channel Gold stack 10%+5% → $85.50.

## What it is for

Four independent discount stages that sit between the price a rule gives a
line and the rep's own manual discount. Every one of them is _automatic_ —
the rep does not type a number, a matrix or a rule decides it:

- **System discounts** — standing policy discounts keyed off the account,
  opportunity or product ("Healthcare gets 5% off everything", "Renewals get
  3% off"). Authored in Rules Studio as a **Discount Rule Set**.
- **Volume discounts** — a quantity ladder on one product ("100+ seats gets
  15% off"). Authored as a **Volume Discount Table**.
- **Promotions** — a code the rep (or a system) puts on the quote header
  ("SPRING26 takes 10% off"). Time-boxed, usage-limited.
- **Channel discounts** — a discount keyed off the reseller's partner tier
  ("Gold partners get 25% off"). No UI surface in this build — see
  Limits and gotchas.

All four reduce a _per-unit rate_, not the line total, and all four run
**before** the rep's manual discount (margin-and-manual-discounts.md) and
**before** CommitDiscount (prepaid-and-commitments.md).

## How it works

### Pipeline order (CpqEngine, confirmed by reading the waterfall stages)

```
DerivedListPrice  (pricing rule, prepaid-and-commitments / price-rules module)
  → RampAdjustedPrice → TermAdjustedPrice → UsageTierPrice
  → SystemDiscount
  → PromotionDiscount
  → ChannelDiscount
  → VolumeDiscount
  → ManualDiscount        (margin-and-manual-discounts.md)
  → BlockDiscount         (Deal Blocks, cart.md)
  → CommitDiscount        (prepaid-and-commitments.md)
  → Net
```

This is **not** the same order CLAUDE.md's short pipeline summary implies
("System + Volume Discounts (multiplicative stack)" as one step) — Promotion
and Channel sit _between_ System and Volume. Verified directly: pricing one
line with both a $10 Absolute promo code and a 10% Gold channel rule gives
$100 → (promo) $90 → (channel) $81.00, not $89.10 (which is what you'd get if
channel ran first). Every stage is multiplicative against the price the
stage before it left — never against the original list price.

### System discounts (`SystemDiscountMatrix__c` → `SystemDiscountRule__c`)

A **Discount Rule Set** (the matrix) has one setting, **When Several Rules
Match** (`StackingStrategy__c`):

| Value                | Label               | Meaning                                                                                                    |
| -------------------- | ------------------- | ---------------------------------------------------------------------------------------------------------- |
| `AllStack` (default) | Apply all           | Every matching rule in the set fires, multiplicatively, in **Apply Order** (`StackingOrder__c`, ascending) |
| `TakeBest`           | Apply only the best | Only the single most generous matching rule in the set fires                                               |

Several Active rule sets can all cover the same product at once (DDCPQ-32):
each set applies its **own** strategy internally first, then the survivors of
every set stack together, multiplicatively, in Apply Order. With one rule set
this is indistinguishable from the old single-matrix behaviour.

`Discount Type` (`DiscountType__c`) is **Percent** or **Fixed** — "Fixed"
takes a flat amount off each unit, not off the line total. TakeBest has to
compare a Percent rule against a Fixed one on equal terms, so it converts
Fixed to an _effective percent_ against the pre-System price
(`value / basePrice × 100`) before picking the winner. This matters:

> Verified: on a $200 line, Percent 5% (eff. 5%) vs Fixed $30 (eff. 15%) — the
> **Fixed rule wins** even though its own field shows "30" and the other
> shows "5", because 15% beats 5%. On the same $200 line, Percent 20% (eff.
> 20%) vs Fixed $30 (eff. 15%) — the **Percent rule wins**. TakeBest never
> compares raw numbers, always effective percent.

`Applies To Product` (`TargetProduct__c`) blank = every product on the quote.
A rule can also carry criteria (the shared 5-field schema — see
concepts.md): blank criteria = catch-all.

### Volume discounts (`VolumeDiscountMatrix__c` → `VolumeDiscountTier__c`)

One **Volume Discount Table** (matrix) is a quantity ladder for **one**
product — `TargetProduct__c` on the matrix header is required for an Active
matrix (validation rule `TargetProduct_Required`). Each **Tier** row is a
band: `MinQuantity__c` is inclusive, `MaxQuantity__c` is **exclusive** — a
tier of Min 1 / Max 10 covers quantities 1–9, not 1–10. The highest tier
whose `[Min, Max)` the line's quantity falls into wins; there is no
"best wins" or stacking between tiers — exactly one tier (or none) applies
to a line.

**Gaps are real and silent.** If your ladder has Min 1/Max 10 and Min 30 (no
Max), quantities 10–29 fall in neither tier and the line prices at **zero
volume discount** — not the lower tier, not the higher one. The engine does
not warn you; the waterfall's VolumeDiscount stage simply shows `applicability:
passthrough` with no note.

> Verified: ladder Min 1/Max 10 @ 5%, Min 30 (no Max) @ 15%, on a $100 line.
> Qty 5 → **$95.00** (5% tier). Qty 15 → **$100.00**, `volumeDiscountPct: 0`
> (the gap). Qty 30 → **$85.00** (15% tier).

`DiscountType__c` is Percent or Fixed, same semantics as System discounts —
the Fixed amount comes off **every** unit in the line, not once per line.
`MinQuantity__c`/`MaxQuantity__c` are evaluated against the **line's own
quantity**, not a running total across lines or across the whole quote —
there is no quote-level aggregate tier in this build.

**Legacy field, now ignored in favor of the header:** `VolumeDiscountTier__c.
TargetProduct__c` still exists but the matrix's own `TargetProduct__c` wins
when both are set; it is a fallback only for matrices saved before the
header field existed.

**Overlapping matrices are not resolved by "best wins."** Two Active
matrices can both target the same product (nothing stops you creating two).
When they do, and both have a tier that matches the line's quantity, the
engine does not pick the bigger discount — it keeps whichever tier it
examined first, which in practice means whichever matrix/tier was **created
earlier** (lower record Id), not the more specific or more generous one.

> Verified: two Active matrices both targeting DS Zeta, both with a Min-1
> (no Max) tier — one at 10%, one at 25%, the 10% one created first. Pricing
> the line gave **$90.00** (`volumeDiscountPct: 10`), the 10% tier, not the
> more generous 25%. This is a real footgun: never let two Active matrices
> cover the same product.

### Promotions (`PromotionRule__c` → `PromotionRedemption__c`)

A promotion is a **code** a rep puts on the quote, not something they pick
from a list. The engine reads `Quote.PromotionCode__c` on every reprice
unless the caller explicitly overrides it for that one call — so a code
survives Edit Lines and later reprices without the rep retyping it. There is
no cart UI for entering it in 0.1.0-9: the rep edits the quote's **Promotion
Code** field directly (standard field edit on the Quote record, or the Quote
edit page), the same way they'd set any other quote field.

Matching, per line: the code (case-insensitive) must equal
`PromotionRule__c.PromotionCode__c`; the rule's **Applies To Product**
(blank = every product) must cover the line; `Start Date`/`End Date` (blank
= open-ended) must bracket today. A product-specific rule beats a catch-all
rule for the same code. Only **one** promotion rule applies per line, ever —
promotions do not stack with each other the way System discounts do.

`Discount Type` on `PromotionRule__c` is **Percent** or **Absolute** — note
the different label from System/Channel/Volume, which call the flat-amount
option "Fixed." Same math either way: Absolute comes off the per-unit rate,
clamped at zero.

A code that matches nothing — wrong spelling, expired, right code but wrong
product — is not silently ignored: the engine adds a quote-level notice
("promo code did not match any active rule" or similar) so the rep knows
their code did nothing, rather than thinking it worked.

**Usage limits.** `MaxRedemptions__c` caps total uses across every customer;
`PerCustomerLimit__c` caps uses by one Account. A "use" is counted only when
a quote carrying the code is **accepted** — a draft quote, or a hundred reps
previewing the same code, never consumes it. On every reprice the engine
quietly drops (not errors) any promo rule whose limit is already used up —
the line just prices as if the code were never supplied, with no row-level
message in the price preview itself (the limit-exhaustion message is
surfaced when the rep tries to **accept** the quote, where it blocks the
accept outright rather than letting it through silently).

> Verified: `DSABS10` ($10 Absolute, Max Uses = 1) with one simulated prior
> redemption recorded on a different quote. Repricing the same line again —
> same code, same product — silently fell back to no promotion: the
> `$10` never applied; only the Channel discount on that line showed.

**Code length — a real build mismatch.** `PromotionRule__c.PromotionCode__c`
is 64 characters. `Quote.PromotionCode__c` — the field the rep actually
types into — is only **40** characters in this installed build. DDCPQ-104
(fixing the quote field to 64 to match) is **not installed**: any code
longer than 40 characters can be authored on the rule but can never be
entered on the quote. Keep codes to 40 characters or fewer until a newer
build ships.

### Channel discounts (`ChannelDiscountRule__c`)

Keyed off a **Partner Tier** picklist (`Gold`, `Silver`, `Bronze`,
`Distributor`) on the rule. Multiple matching Active rules for the same tier
stack **multiplicatively**, in `Stacking Order` ascending — same mechanics
as System discounts' AllStack, but channel rules have no AllStack/TakeBest
switch; they always all apply.

> Verified: two Active Gold rules on one product, 10% then 5%, Stacking
> Order 1 then 2. $100 line → $100 × 0.90 × 0.95 = **$85.50**. A single Gold
> rule at 10% on another product → **$90.00**.

**Where the partner tier comes from: nowhere in this build's UI.** There is
no field on Account, Quote, or anywhere else that holds "this deal's partner
tier". The fix (a Partner Tier the rep sets on the quote) is DDCPQ-116 and
arrives in a later build. The **only** way to apply a channel discount is to
pass `partnerTier` explicitly on the pricing request — something only a
caller of the REST/engine API can do, not a rep clicking in the cart. See
Limits and gotchas.

### Rounding

Every intermediate stage keeps 6 decimal places (`setScale(6,
HALF_UP)`). Only the final value written to `QuoteLineItem.NetPrice__c`
rounds to cents (field scale 2). The stored discount-percent fields
(`SystemDiscount__c`, `VolumeDiscount__c`, `ChannelDiscount__c`,
`PromotionDiscount__c`) keep 4 decimal places, so two stacked 5% System
discounts (0.95 × 0.95 = 0.9025) store as 9.75%, not a rounded 10%. In this
installed build (DDCPQ-112 is **not** in 0.1.0-9) the price waterfall's
_header_ percentage in the cart UI still rounds to the nearest whole number,
so it can show "10%" next to a line whose stored `SystemDiscount__c` is
9.7500 — a cosmetic mismatch, not a pricing bug. Trust the QLI field and the
per-rule waterfall rows over the header's rounded percent.

## Objects and fields

| Object label (API name)                           | Field                                                                                  | Meaning / values                                              |
| ------------------------------------------------- | -------------------------------------------------------------------------------------- | ------------------------------------------------------------- |
| Discount Rule Set (`SystemDiscountMatrix__c`)     | `Status__c`                                                                            | Draft / Active / Archived — only Active enters pricing        |
|                                                   | `StackingStrategy__c`                                                                  | `AllStack` (apply all) / `TakeBest` (apply only the best)     |
| System Discount Rule (`SystemDiscountRule__c`)    | `Matrix__c`                                                                            | Lookup to its rule set                                        |
|                                                   | `TargetProduct__c`                                                                     | Blank = every product                                         |
|                                                   | `DiscountType__c`                                                                      | `Percent` / `Fixed`                                           |
|                                                   | `DiscountValue__c`                                                                     | The number — % or flat amount per unit                        |
|                                                   | `StackingOrder__c`                                                                     | Apply Order, ascending, within AllStack                       |
|                                                   | `FieldApiName__c`/`Operator__c`/`Value__c`/`ValueType__c`/`Priority__c`                | The shared 5-field criteria schema (concepts.md)              |
| Volume Discount Table (`VolumeDiscountMatrix__c`) | `TargetProduct__c`                                                                     | Required for Active — one matrix per product                  |
|                                                   | `Status__c`                                                                            | Draft / Active / Archived                                     |
| Volume Discount Tier (`VolumeDiscountTier__c`)    | `Matrix__c`                                                                            | Lookup to its table                                           |
|                                                   | `MinQuantity__c`                                                                       | Inclusive lower bound                                         |
|                                                   | `MaxQuantity__c`                                                                       | Exclusive upper bound; blank = unbounded                      |
|                                                   | `DiscountType__c`                                                                      | `Percent` / `Fixed`                                           |
|                                                   | `DiscountValue__c`                                                                     | The number                                                    |
|                                                   | `TargetProduct__c`                                                                     | Legacy; matrix header wins when both set                      |
| Promotion Rule (`PromotionRule__c`)               | `PromotionCode__c`                                                                     | Up to 64 chars; case-insensitive match                        |
|                                                   | `TargetProduct__c`                                                                     | Blank = every product                                         |
|                                                   | `DiscountType__c`                                                                      | `Percent` / `Absolute`                                        |
|                                                   | `DiscountValue__c`                                                                     | The number                                                    |
|                                                   | `StartDate__c`/`EndDate__c`                                                            | Blank = open-ended                                            |
|                                                   | `MaxRedemptions__c`                                                                    | Total accepted-quote uses; blank = unlimited                  |
|                                                   | `PerCustomerLimit__c`                                                                  | Uses by one Account; blank = unlimited                        |
|                                                   | `Status__c`                                                                            | Draft / Active                                                |
| Promotion Redemption (`PromotionRedemption__c`)   | `PromotionRule__c`, `Quote__c`, `Account__c`                                           | One row per rule per accepted quote                           |
|                                                   | `RedemptionKey__c`                                                                     | `ruleId:quoteId` — stops double-counting a re-accept          |
| Quote (standard, extended)                        | `PromotionCode__c`                                                                     | **40 chars** in this build (not 64 — DDCPQ-104 not installed) |
| Channel Discount Rule (`ChannelDiscountRule__c`)  | `PartnerTier__c`                                                                       | `Gold` / `Silver` / `Bronze` / `Distributor`, required        |
|                                                   | `TargetProduct__c`                                                                     | Blank = every product                                         |
|                                                   | `DiscountType__c`                                                                      | `Percent` / `Absolute`                                        |
|                                                   | `DiscountValue__c`                                                                     | The number                                                    |
|                                                   | `StackingOrder__c`                                                                     | Ascending; all matching rules apply (no TakeBest)             |
|                                                   | `Status__c`                                                                            | Draft / Active                                                |
| QuoteLineItem (extended)                          | `SystemDiscount__c`, `VolumeDiscount__c`, `PromotionDiscount__c`, `ChannelDiscount__c` | Effective % from each stage, 4 decimal places                 |
|                                                   | `NetPrice__c`                                                                          | Final per-unit price after every stage, rounded to cents      |

## Build it (tester)

A worked scenario for **System discounts (AllStack)** and **Volume**
together, reusing the golden-path quote so you can see every stage move.
Numbers below are what a correctly-built org gives — see "Expected numbers."

1. **Products** — in Bundle Builder / Products tab, create **DS Widget**,
   Active, Standard Price **$100**. Add an Active rate plan: Recurring, Per
   Unit, Monthly, $100 (Rate Plan Editor tab — the Rate Plan Gate blocks any
   line with no Active plan, same as Stage 6 of the golden path).
2. **Account** — `DS Test Co`, any Industry.
3. **Discount Rule Set** — Rules Studio tab → **New Discount Rule Set** →
   name `DS System Discounts`, **When Several Rules Match** = **Apply all**.
   Add one rule: target **DS Widget**, Discount Type **Percent**, value
   **10**, Apply Order **1**, no condition (catch-all). Save, then flip the
   set to **Active** — Rules Studio always saves new sets as Draft.
4. **Volume Discount Table** — Rules Studio tab → **New Volume Discount
   Table** → Applies To Product **DS Widget**. Add tiers: Min **1** / Max
   **10** / Percent **5**; Min **30** (leave Max blank) / Percent **15**.
   Save, switch to **Active**.
5. **Quote** — add **DS Widget**, quantity **5**. Net price should read
   **$85.50** ($100 × 0.90 System × 0.95 Volume). Change quantity to **15**
   — net price jumps to **$90.00** (System only — the 10–29 gap has no
   volume tier). Change quantity to **30** — net price drops to **$76.50**
   ($100 × 0.90 × 0.85).
6. Click the price to open the waterfall and confirm the System and Volume
   rows show the rule(s) that fired, and (at qty 15) that Volume shows no
   rule and no discount.

For **Promotions**: add a **Promotion Rule** (App Launcher → Promotion
Rules) with Promotion Code `DSPROMO1` (≤ 40 characters — see Limits), Status
Active, Discount Type Percent, value 10, Applies To Product blank. Open the
quote → edit the **Promotion Code** field (quote detail page, not the cart)
→ type `dspromo1` (case does not matter) → Save. Reprice the cart (reopen
Configure Products, or change any quantity) — every line's price should
drop another 10% from wherever System/Volume left it.

Channel discounts cannot be built or seen through the UI at all in this
build — there is nothing to click. If you want to prove `ChannelDiscountRule__c`
works, create one (App Launcher → Channel Discount Rules → New: Partner
Tier Gold, Applies To Product your product, Discount Type Percent, value
25, Status Active) and ask Claude to verify it with the read-only price
preview below — never ask the tester to run a REST call themselves.

## Check it (Claude)

```bash
sf data query -o dd-e2e -q "SELECT Name, DDCPQ__Status__c, DDCPQ__StackingStrategy__c, (SELECT DDCPQ__TargetProduct__r.Name, DDCPQ__DiscountType__c, DDCPQ__DiscountValue__c, DDCPQ__StackingOrder__c FROM DDCPQ__SystemDiscountRules__r) FROM DDCPQ__SystemDiscountMatrix__c WHERE Name LIKE '%System Discounts%'"

sf data query -o dd-e2e -q "SELECT Name, DDCPQ__Status__c, DDCPQ__TargetProduct__r.Name, (SELECT DDCPQ__MinQuantity__c, DDCPQ__MaxQuantity__c, DDCPQ__DiscountType__c, DDCPQ__DiscountValue__c FROM DDCPQ__VolumeDiscountTiers__r) FROM DDCPQ__VolumeDiscountMatrix__c WHERE Name LIKE '%Volume%'"

sf data query -o dd-e2e -q "SELECT DDCPQ__PromotionCode__c, DDCPQ__Status__c, DDCPQ__TargetProduct__r.Name, DDCPQ__DiscountType__c, DDCPQ__DiscountValue__c, DDCPQ__StartDate__c, DDCPQ__EndDate__c, DDCPQ__MaxRedemptions__c, DDCPQ__PerCustomerLimit__c FROM DDCPQ__PromotionRule__c"

sf data query -o dd-e2e -q "SELECT DDCPQ__PartnerTier__c, DDCPQ__TargetProduct__r.Name, DDCPQ__DiscountType__c, DDCPQ__DiscountValue__c, DDCPQ__StackingOrder__c, DDCPQ__Status__c FROM DDCPQ__ChannelDiscountRule__c"

sf data query -o dd-e2e -q "SELECT Id, PromotionCode__c FROM Quote WHERE Id = '<quote Id>'"

sf data query -o dd-e2e -q "SELECT Product2.Name, Quantity, DDCPQ__NetPrice__c, DDCPQ__SystemDiscount__c, DDCPQ__VolumeDiscount__c, DDCPQ__PromotionDiscount__c, DDCPQ__ChannelDiscount__c FROM QuoteLineItem WHERE QuoteId = '<quote Id>'"
```

If the subquery relationship names are rejected, query the child objects
directly with `DDCPQ__Matrix__r.Name` instead — same pattern as the golden
path's note for pricing rules.

**Read-only price preview** (never commit — never call `dd/v1/commit`):

```bash
sf api request rest "services/apexrest/DDCPQ/dd/v1/price?includeWaterfall=true" -o dd-e2e --method POST --body @body.json
```

```json
{
  "quoteId": "<0Q0…>",
  "promoCode": "DSPROMO1",
  "partnerTier": "Gold",
  "selections": [{ "localKey": "x", "productId": "<01t…>", "quantity": 30 }]
}
```

`promoCode`/`partnerTier` are optional — omit either to see the line without
that stage. If blank, `promoCode` falls back to the quote's own
`PromotionCode__c` field automatically; `partnerTier` has no fallback at
all (there is nowhere on the org for it to come from), so leaving it out is
the only way most testers will ever see it, since the cart never sends it.

## Expected numbers

All runs below against a $100 or $200 standard-price line, quantity as
noted, verified in cpq-pkg 2026-10-06.

| Scenario                        | Setup                                                   | Net price                                                 | Effective % stored                             |
| ------------------------------- | ------------------------------------------------------- | --------------------------------------------------------- | ---------------------------------------------- |
| System, AllStack                | $200 line, 10% then 5%, Apply Order 1/2                 | **$171.00**                                               | SystemDiscount 14.5%                           |
| System, TakeBest (Percent wins) | $200 line, Percent 20% vs Fixed $30                     | **$160.00**                                               | SystemDiscount 20%                             |
| System, TakeBest (Fixed wins)   | $200 line, Percent 5% vs Fixed $30                      | **$170.00**                                               | SystemDiscount 15%                             |
| Volume, in first tier           | $100 line, qty 5, ladder 1–10 @5% / 30+ @15%            | **$95.00**                                                | VolumeDiscount 5%                              |
| Volume, in the gap              | same ladder, qty 15                                     | **$100.00**                                               | VolumeDiscount 0%                              |
| Volume, in second tier          | same ladder, qty 30                                     | **$85.00**                                                | VolumeDiscount 15%                             |
| Volume, Fixed type              | $100 line, qty 5, Fixed $10 off, Min 1                  | **$90.00**                                                | VolumeDiscount 10%                             |
| Volume, overlapping matrices    | $100 line, two Active matrices both Min 1 (10% and 25%) | **$90.00** (10% won, not 25%)                             | VolumeDiscount 10%                             |
| Promotion, Percent              | $100 line, code 10% Percent, product-specific           | **$90.00**                                                | PromotionDiscount 10%                          |
| Channel, single rule            | $100 line, one Active Gold rule, 10%                    | **$90.00**                                                | ChannelDiscount 10%                            |
| Channel, stacked                | $100 line, two Active Gold rules, 10% then 5%           | **$85.50**                                                | ChannelDiscount 14.5%*                         |
| Promotion + Channel together    | $100 line, Absolute $10 promo then Gold 10% channel     | **$81.00**                                                | Promo 10%, Channel 10% (on the post-promo $90) |
| Promotion, limit reached        | Absolute $10 promo, Max Uses 1, already used once       | **$90.00** (promo silently dropped; only Channel applied) | PromotionDiscount 0%                           |

\* 1 − (0.90 × 0.95) = 14.5% combined, even though each rule's own value reads 10 and 5.

## Troubleshooting

| Symptom                                                                     | Cause                                                                                                                                                                                                                                 | Fix                                                                                                                |
| --------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------ |
| Promotion code does nothing                                                 | Code on `Quote.PromotionCode__c` doesn't match `PromotionRule__c.PromotionCode__c` (check spelling — comparison is case-insensitive but not fuzzy), or the rule's product doesn't cover this line, or today is outside Start/End Date | Re-check the rule's code, product and dates; the engine adds a quote notice when a supplied code matched nothing   |
| Promotion code was accepted on the rule but won't save on the Quote         | Code is longer than 40 characters — `Quote.PromotionCode__c` is only 40 chars in this build even though the rule field allows 64                                                                                                      | Shorten the code to 40 characters or fewer                                                                         |
| Two System discounts you expected to add up only gave one                   | The rule set's **When Several Rules Match** is **Apply only the best** (TakeBest), not **Apply all**                                                                                                                                  | Check `StackingStrategy__c` on the Discount Rule Set                                                               |
| A Fixed discount beat a bigger-looking Percent discount under TakeBest      | TakeBest compares _effective percent_, not the raw numbers — a $30 Fixed off a $200 line is 15%, which beats a 5% Percent rule                                                                                                        | Expected behaviour — convert the Fixed value to a percent of the base price yourself to sanity-check it            |
| Volume discount missing at a quantity between two tiers                     | A gap in the ladder — `MaxQuantity__c` is **exclusive**, so Min 1/Max 10 does not cover quantity 10                                                                                                                                   | Add a tier (or extend Max) to close the gap, or confirm the gap is intentional                                     |
| Volume discount is lower than expected even though a bigger discount exists | Two Active matrices target the same product; the engine keeps whichever tier it reaches first, not the larger one                                                                                                                     | Never let two Active `VolumeDiscountMatrix__c` cover the same product — archive one                                |
| Channel discount never shows up in the cart                                 | There is no UI to set a partner tier in this build — the cart never sends `partnerTier`                                                                                                                                               | Expected in 0.1.0-9. Verify the rule itself only via the read-only price preview, passing `partnerTier` explicitly |
| "Each needs an Active rate plan" when adding the discounted product         | Nothing to do with discounts — the Rate Plan Gate (pricing-models.md) is unrelated and runs first                                                                                                                                     | Add an Active rate plan to the product, then retry                                                                 |
| Promotion worked once, then stopped working on the same quote               | `MaxRedemptions__c` or `PerCustomerLimit__c` reached — a prior **accepted** quote used it up                                                                                                                                          | Check `PromotionRedemption__c` rows for this rule; raise the limit or use a different code                         |
| Waterfall header % doesn't match the stored discount field                  | DDCPQ-112 (precise header rounding) is not in this build — the header rounds to a whole number                                                                                                                                        | Cosmetic only; trust `SystemDiscount__c`/`VolumeDiscount__c` on the QLI                                            |

## Limits and gotchas

- **No cart UI for Channel discounts.** `partnerTier` only exists as a field
  on the engine's pricing/commit request. There is no field on Account,
  Quote, or anywhere else holding a partner tier, and no LWC control sends
  one. A tester clicking through the cart will never see a channel discount
  apply, no matter how the `ChannelDiscountRule__c` is set up. This is a known
  gap, tracked as DDCPQ-116: there is nowhere for "this deal's partner
  tier" to live yet.
- **Quote.PromotionCode__c is 40 characters, not 64.** `PromotionRule__c.
PromotionCode__c` allows up to 64; a rule authored with a longer code can
  never actually be entered on a quote in this build. DDCPQ-104 fixes this
  in trunk; not installed in 0.1.0-9.
- **Rules Studio's rule tester is removed in this build too** (DDCPQ-68, not
  installed) — there is no in-app way to test a discount/promotion/channel
  rule before saving it Active. Verify with a real quote or the read-only
  price preview instead.
- **No per-rule attribution on the waterfall header** in this build
  (DDCPQ-69, not installed) for Promotion/Channel/Volume the way System
  discount rows list every stacked rule — the response JSON still carries
  `sourceRuleId` per stage, so `sf api request` reveals which rule fired
  even when the cart UI doesn't spell it out as plainly.
- **Overlapping Volume matrices are not validated against each other.**
  The `TargetProduct_Required` validation rule stops an Active matrix from
  having a blank product, but nothing stops two Active matrices sharing one.
  Treat "one Active matrix per product" as a hard rule you enforce by hand.
- **A promo or channel discount never fires on a bundle parent** (roll-up
  pricing) or on a hand-typed Sales Price Override — both skip every
  discount stage by design, same as System and Volume.
- **Governor limits:** all four services batch-load their rules once per
  price/commit call (no SOQL per line), so a 200-line cart is one query per
  object type, not 200. Nothing here is a bulkification risk a tester would
  hit by hand.

## Questions testers ask

**Why did my 5% + 5% System discount show as 10% in one place and 9.75% in
another?**
Two 5% discounts compound (0.95 × 0.95 = 0.9025, i.e. 9.75% off), they don't
add. The stored field is the correct 9.75% (rounds to 9.8% in the cart
grid); the waterfall header rounding it to "10%" is a known display bug in
this build (DDCPQ-112, not installed).

**I set Discount Type to Fixed and put in 10 — why did it take 10% off, not
$10?**
It didn't — check the field again. Fixed always means a flat currency
amount per unit; if the line moved by a percentage, a _different_ rule (or
the Volume tier) is the one that fired.

**Can I stack two promotion codes?**
No. Only one `PromotionRule__c` ever applies per line, even if the quote
somehow had two codes that both matched. There's no concept of multiple
promo codes on one quote in this build.

**Why does my channel discount rule do nothing when I price a real quote?**
Because nothing in the cart UI sends a partner tier. Ask Claude to verify
the rule with the read-only REST preview instead — that is the only path to
exercising `ChannelDiscountRule__c` in 0.1.0-9.

**My volume tiers are 1–50 and 50–100 — why did quantity 50 get the wrong
discount?**
`MaxQuantity__c` is exclusive. A tier of Min 1/Max 50 stops at 49; quantity
50 needs a tier whose Min is 50 (or lower) — check your ladder's boundaries
match what you intend, inclusive-low/exclusive-high.

**I lowered a promotion's Max Uses after it was already used more times
than that — what happens?**
The next reprice drops the rule immediately (it's over the limit now), and
an attempt to accept a quote using it is refused. Nothing retroactively
undoes quotes already accepted under the old limit.

**Does a System discount apply before or after the account's volume
tier?**
Before. The order is System → Promotion → Channel → Volume → Manual. Volume
is the last automatic discount before the rep's own manual discount.

**Why did my Absolute promotion of $10 and my 10% Channel discount not add
up to 11% off a $100 line?**
They don't add — each stage multiplies against what the stage before it
left. $100 → ($10 off) $90 → (10% off $90) $81.00, which is 19% off overall,
not 11%. Each stage's own "effective %" field is measured against its own
starting price, not the original list price.
