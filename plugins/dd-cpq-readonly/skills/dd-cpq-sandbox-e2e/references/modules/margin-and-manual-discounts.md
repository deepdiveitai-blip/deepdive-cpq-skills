# Manual discounts, margin floors and approvals

> Installed build 0.1.0-9 · Verified 2026-10-06 in cpq-pkg: % off 20% on a
> $100/unit line → $80; $ off $15/unit → $85; Set price $42 → $42; a
> Clamp-action floor at 20% markup on cost $90 raised a 50%-off $100 line to
> $108; a Block-action floor refused the commit with `FLOOR_BLOCKED`; an
> Approval-action floor let the line commit at $50 with
> `FloorApprovalRequired__c = true`.

## What it is for

Two separate concerns that share one pipeline position (the last two stages
before Net):

- **Manual discounts** are what a rep types on a line by hand — a percent
  off, a dollar amount off each unit, or a flat price per unit. They are the
  rep's own concession, layered on top of (or instead of) every system,
  promotion, channel and volume discount the engine already computed.
- **Margin floors** are the house's guard rail against that concession —
  and against every other discount stage — going too far below what the
  line actually costs. They run after everything else, including the
  manual discount, and they are the only check in this build that can stop
  a price from committing.

Approvals are the join between the two: a margin floor can be configured to
flag a line instead of silently fixing or blocking it, and that flag is
meant to route to a human. This build stamps the flag; it does not ship the
routing.

## How it works

### Manual discount modes

The cart offers exactly three, one at a time, aria-labelled **"How to apply
the discount"** in `cpqLedger`:

| Button label  | Field written             | What it means                                                                                                                                                                                                                                                                   |
| ------------- | ------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **% off**     | `ManualDiscount__c`       | A percent taken off whatever the line is after every system, promotion, channel and volume discount. Moves with the line as quantity/tier change.                                                                                                                               |
| **$ off**     | `ManualDiscountAmount__c` | A fixed amount off **each unit**, taken at the same point as % off, after it. Stays that amount when quantity or tier changes — a percent would not.                                                                                                                            |
| **Set price** | `SalesPriceOverride__c`   | The final per-unit price. Every stage below it — system, promotion, channel, volume, manual %, manual $, commitment, payment terms — records itself as **skipped**. A standing entitlement (e.g. 5% for Healthcare) does **not** apply to a line whose price was typed by hand. |

The cart lets a rep choose only one mode per line — picking a new mode
clears the others (`handleDiscountMode` resets `discountDraft`, and
`handleDiscountApply` writes only the active field, zeroing/nulling the
other two). An API caller (REST/MCP) is not bound by that: the engine will
happily apply `manualDiscountPct`, `manualDiscountAmount` and
`salesPriceOverride` together if all three are sent, because the override
makes the other two meaningless (they are skipped, not summed).

**Clamps** (`CpqEngine.clampManualPct` / `clampManualAmount`):

- `manualDiscountPct`: negative or blank → 0; anything above 100 → 100.
  Never negative, never over 100.
- `manualDiscountAmount`: negative or blank → 0; anything above the line's
  own per-unit price → clamped to that price. A line can be discounted to
  exactly $0 by amount, never below it.
- `salesPriceOverride` has no clamp — a rep (or API caller) can type $0 or
  any positive number; it is the literal per-unit price.

**Pipeline position** — `ManualDiscount` is the second-to-last waterfall
stage, right before `CommitDiscount` and `MarginFloor`:

```
… VolumeDiscount → ManualDiscount → CommitDiscount → MarginFloor → Net
```

`SalesPriceOverride`, when present, is its **own** stage that runs where
`ManualDiscount` would have — not a note on another stage. This exists
because an earlier bug let an override's price drop show up against
whichever stage printed next (a $20 line set to $5 once read "System
discount $5.00" on a quote with no system discount rule at all — see
`project_ddcpq_cart_discount_modes` memory). The `SalesPriceOverride` stage
is always shown, never collapsed, even when every other stage is
not-applicable.

### Margin floor

`MarginFloorRule__c` is evaluated by `MarginFloorService.enforce`, the last
step before `Net` — after the manual discount, not before it. One rule can
cover one product (`TargetProduct__c`) or every product (`TargetProduct__c`
blank, matched by the shared 5-field criteria schema). A rule naming this
specific product beats an all-products rule; among equals, lowest
`Priority__c` wins (`RulePrecedence.pickWinner`).

**Unit cost.** The floor needs `QuoteLineItem.Cost__c`. It is populated at
engine time, in this order:

1. **`RatePlan__c.UnitCost__c`** — read in system mode by
   `ChargeProfileService`, so it works even for a rep with no FLS on the
   field. If the plan has a cost, it **always wins**.
2. Only when the plan has none does a cost the API caller sent
   (`selections[].cost`) get used. The cart has no field for this — a rep
   cannot type a cost. It exists for REST/MCP callers (and this module's
   own verification) to exercise the floor without stamping a plan.

A line with **no cost at all** (blank on the plan and nothing sent) is
**not checked** — it is not silently skipped: the engine emits a
`FLOOR_NOT_CHECKED` warning ("Margin floor not checked: the rate plan for
this line has no unit cost.") and a `MarginFloor` waterfall entry noting
"Not checked: no unit cost on the rate plan." Product 360 surfaces the same
gap as finding `FLOOR_NO_COST`.

**The floor itself** — `FloorBasis__c` picks the formula (blank reads as
Markup):

| Floor Basis                  | Formula              | Example (cost 90, Floor % 20) |
| ---------------------------- | -------------------- | ----------------------------- |
| **Markup on cost** (default) | `cost × (1 + %/100)` | `90 × 1.20 = 108`             |
| **Gross margin**             | `cost / (1 − %/100)` | `90 / 0.80 = 112.50`          |

`FloorBasis__c = Margin` with `FloorPercent__c ≥ 100` has no valid floor
(division by zero or negative) — the engine returns `null` and skips the
line rather than erroring.

**Action, when Net is below the floor:**

| Action              | Behaviour, verified                                                                                                                                                                                | Exact message                                                                                                                                                                                                                                                                                                                           |
| ------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Clamp** (default) | `draft.netPrice` is silently raised to the floor. The line commits at the floor price.                                                                                                             | Waterfall note: `"Raised to floor."` Waterfall UI badge: **"Raised to the margin floor"**.                                                                                                                                                                                                                                              |
| **Block**           | Nothing is raised. `QuoteLineItem.BlockedByFloor__c` is set true in-memory and **commit is refused outright** — no QuoteLineItems are inserted for the whole request, not just the offending line. | REST `commit` returns `"errorType": "ConfigurationRejected"`, `"code": "FLOOR_BLOCKED"`, and verified text: `"Commit refused: 1 line(s) violate an Active MarginFloorRule with Action=Block. Blocked line keys: b1. Adjust price above the floor, request an approval override, or change the rule to Action=Clamp before committing."` |
| **Approval**        | Nothing is raised or blocked. `QuoteLineItem.FloorApprovalRequired__c` is set true and the line **commits at the price the rep typed**, below the floor.                                           | `price` preview returns an `EngineWarning` with code `FLOOR_APPROVAL_REQUIRED`: `"This line is below its margin floor and needs approval. The quote can still be saved."` Waterfall UI badge: **"Below margin floor · approval required"**.                                                                                             |

A `Block` failure is **all-or-nothing per commit request** — in testing, a
quote with one clean line and one blocked line in the same `commit` call
inserts **zero** lines, not one. Price it again with the offending line
removed, fixed, or the rule's `Action` changed, then commit again.

`FloorApprovalRequired__c` clears itself on the next reprice once the price
is back above the floor — it is not a one-way flag the tester has to clear
by hand.

Bundle **parents** are exempt from margin floor by design — they carry only
the license line; the children that carry real variable cost are what gets
checked (`CommitService.isRateable`).

### The cart highlight — `ManualDiscountOver15`

A custom metadata rule (`DD_CPQ_Cart_Rule__mdt`, record
`ManualDiscountOver15`) tells the cart to stripe a row **warn**-coloured
when `ManualDiscount__c > 15`. This is **presentation only**:

- It does **not** cap the discount, does **not** block anything, and does
  **not** require approval. A rep can still type 40% and commit it — the
  row just turns amber.
- It fires on **`ManualDiscount__c` (the % mode) only.** A line discounted
  by the same economics via `ManualDiscountAmount__c` ($ off) or
  `SalesPriceOverride__c` (Set price) does **not** trigger this rule, even
  if the effective discount is far larger than 15%, because the rule's
  `Field__c` names only `ManualDiscount__c`.
- It stacks with a second rule, `BelowMarginFloor` (`BlockedByFloor__c =
true`, tone **crit**). `crit` always beats `warn`: a line that is both
  over-15%-discounted and blocked by the floor shows as crit, not warn.
- Both rules apply in **every** cart view (`DealDesk`, `Finance`, `Rep`,
  `Customer`, `Amendment`) — neither metadata record sets `Views__c`, which
  the engine reads as "no filter, show everywhere."
- The stripe is counted in the **"Needs attention · N"** KPI tile at the
  top of the cart (`cpqLedger.js` `attentionCount`), alongside a quote the
  engine could not price and a Block that is keeping the quote from
  committing.
- This is unrelated to the `MarginFloor` **waterfall badge** (below the Net
  summary tile, not a cart row stripe) and to the `FloorApprovalRequired__c`
  / `BlockedByFloor__c` **line badges** ("Needs approval", "Floor") shown on
  the line itself — three different signals for three different audiences,
  all sourced from the same underlying flags.

### Approvals — what is and is not packaged

**There is no Salesforce Approval Process shipped with this package.**
Searching the metadata for `ApprovalProcess` returns nothing, and the
package cannot ship one — approval processes are org-specific automation,
not package-installable metadata DD CPQ controls.

What exists instead:

- The engine's own gate (`MarginFloorRule__c` with `Action = Approval`,
  above) — a flag on the QuoteLineItem, nothing more.
- `dd_cpq_submit_for_approval` (REST `process/approvals`, MCP tool
  `dd_cpq_submit_for_approval`) — a thin wrapper around Salesforce's
  **standard** Process Automation REST API. It submits whatever Quote Id
  it is given to whatever Approval Process the org has configured. If the
  org has none, Salesforce itself returns `NO_APPLICABLE_PROCESS` and the
  tool surfaces that verbatim. DD CPQ ships **zero** approval processes,
  so calling this tool in a fresh org fails until an admin builds one.

**For a tester (or admin) who wants real approval routing**, the engine
side is already done; only the native Salesforce side is missing:

1. Setup → Approval Processes → **Manage** for object **Quote**.
2. **Entry Criteria**: `QuoteLineItem.FloorApprovalRequired__c` lives on
   the line, not the Quote header, so a Quote-level process cannot test it
   directly. Two practical options: (a) key entry criteria on a Quote-level
   field the rep's own process already rolls up, or (b) have the admin
   add a roll-up/formula field on Quote that reflects whether any child
   line has `FloorApprovalRequired__c = true` (CLAUDE.md forbids the
   package itself from adding more Quote fields — this would be the
   subscriber org's own customization, not a DD CPQ field).
3. Approval steps, approvers, and actions (e.g. lock the record) same as
   any native approval process.
4. Only then does `dd_cpq_submit_for_approval` have something to submit to.

Until that is built, "needs approval" in DD CPQ means exactly what the
flag says — a fact stamped on the line — and nothing routes anywhere on
its own.

## Objects and fields

| Object (API, unprefixed)                   | Field                                                                                                                                   | Meaning / values                                                                                                                                                                                                                                                                                           |
| ------------------------------------------ | --------------------------------------------------------------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| QuoteLineItem                              | `ManualDiscount__c` ("Manual Discount")                                                                                                 | Rep percent, 0–100. Drives the `ManualDiscountOver15` cart highlight.                                                                                                                                                                                                                                      |
| QuoteLineItem                              | `ManualDiscountAmount__c` ("Manual Discount Amount")                                                                                    | Money off each unit, taken last, after the percent. Clamped to the line's own price.                                                                                                                                                                                                                       |
| QuoteLineItem                              | `SalesPriceOverride__c` ("Sales Price Override")                                                                                        | Final per-unit price. Every stage below it is skipped.                                                                                                                                                                                                                                                     |
| QuoteLineItem                              | `Cost__c` ("Cost")                                                                                                                      | Unit cost the margin floor checks against. Reps do not read it (removed from the USER_MODE SELECT in `CommitService`); admin-visible only.                                                                                                                                                                 |
| QuoteLineItem                              | `BlockedByFloor__c`                                                                                                                     | True when an Active `Action=Block` floor fired. Commit of the whole request is refused while any line has this true.                                                                                                                                                                                       |
| QuoteLineItem                              | `FloorApprovalRequired__c` ("Needs Margin Floor Approval")                                                                              | True when an Active `Action=Approval` floor fired. Line still commits. Clears itself on the next reprice once the price clears the floor.                                                                                                                                                                  |
| QuoteLineItem                              | `IsFloorAdjustment__c`                                                                                                                  | Marks a line the engine itself generated as a floor correction (not exercised directly in this module's scenarios).                                                                                                                                                                                        |
| Margin Floor Rule (`MarginFloorRule__c`)   | `TargetProduct__c`                                                                                                                      | One product this rule covers; blank = every product (lower precedence than a named-product rule).                                                                                                                                                                                                          |
| Margin Floor Rule                          | `Target__c` ("Products Covered")                                                                                                        | JSON `{mode, products, criteria, groupId, exclude}` written by Rules Studio when the rule covers more than one named product. Blank defers to `TargetProduct__c`.                                                                                                                                          |
| Margin Floor Rule                          | `FloorPercent__c` ("Floor %")                                                                                                           | Required. Read per `FloorBasis__c`.                                                                                                                                                                                                                                                                        |
| Margin Floor Rule                          | `FloorBasis__c` ("Floor Basis")                                                                                                         | `Markup` (default) or `Margin`.                                                                                                                                                                                                                                                                            |
| Margin Floor Rule                          | `Action__c` ("Action")                                                                                                                  | `Clamp` (default) / `Block` / `Approval`.                                                                                                                                                                                                                                                                  |
| Margin Floor Rule                          | `Status__c`                                                                                                                             | Must be `Active` — Draft rules are ignored.                                                                                                                                                                                                                                                                |
| Margin Floor Rule                          | `FieldApiName__c` / `Operator__c` / `Value__c` / `ValueType__c` / `Priority__c`                                                         | The shared 5-field criteria schema (CLAUDE.md rule 5) — same operators as every other rule object: `equals`, `not-equals`, `in`, `greater`, `less`, `between`, `is-null`, plus this object also lists `greater-or-equal`, `less-or-equal`, `not-in`, `contains`, `starts-with`, `is-not-null`, `includes`. |
| DD_CPQ_Cart_Rule (`DD_CPQ_Cart_Rule__mdt`) | `Field__c` / `Operator__c` (`gt`\|`lt`\|`eq`\|`ne`) / `Value__c` / `Tone__c` (`warn`\|`crit`) / `SortOrder__c` / `Pack__c` / `Views__c` | Cart-only row-stripe rules. `ManualDiscountOver15` and `BelowMarginFloor` are the two shipped records.                                                                                                                                                                                                     |

## Build it (tester)

This module's own verification used REST (products, rate plans and rules
created by `sf data create record` to isolate the scenario), not the UI —
the clicks below follow the same object model and the golden path's Stage
7 pattern (Rules Studio), but were not click-tested in this pass. Treat
them as "from source," not "verified," and tell Claude if a label does not
match what you see.

**Scenario: a 50%-off line against a 20%-markup floor on cost $90, three
ways.**

1. Reuse the Stage 4–6 setup (a product with an Active rate plan). Open
   the rate plan in **Rate Plan Editor** and set **Unit Cost** to `90`
   (admin-only field — if you cannot see it, your permission set lacks
   `UnitCost__c` read/edit, expected for the Rep permission set).
2. **Rules Studio** → the rule-type tab labelled **Margin Floors** → **New**.
   Title it, e.g., `MD Floor Clamp`. Target your one product. **Floor %**
   `20`, **Floor Basis** `Markup on cost`, **Action** `Clamp`. Save, then
   set **Status** to **Active** (new rule sets save as Draft).
3. Open **Configure Products** on a quote for an account with Industry/Type
   that does not otherwise discount the line (to isolate the floor from
   system discounts), add the product, quantity 1.
4. Click the line's discount icon → **% off** → type `50` → Apply. The net
   price should jump to **$108** (not $50) — the floor overrode the manual
   discount. Open the price to see the **MarginFloor** badge read
   **"Raised to the margin floor."**
5. Repeat with a second rule on a second product, **Action** `Block`.
   Discount it 50% the same way, then **Commit** the quote. Expect the
   commit to be refused and an error naming `FLOOR_BLOCKED` /
   "violate an Active MarginFloorRule with Action=Block" — look for it in
   the **Error Console** (`cpqErrorConsole`) if the inline message is cut
   off.
6. Repeat with a third rule, **Action** `Approval`. Discount it 50% and
   commit — this one succeeds. Reopen the line: it carries a **"Needs
   approval"** badge.
7. Hover a line discounted more than 15% by **% off** (not $ off, not Set
   price) anywhere in the cart and watch the row pick up a warn stripe;
   check the **"Needs attention · N"** tile at the top count it.

## Check it (Claude)

```bash
sf data query -o dd-e2e -q "SELECT Product2.Name, Quantity, UnitPrice, DDCPQ__NetPrice__c, DDCPQ__Cost__c, DDCPQ__ManualDiscount__c, DDCPQ__ManualDiscountAmount__c, DDCPQ__SalesPriceOverride__c, DDCPQ__BlockedByFloor__c, DDCPQ__FloorApprovalRequired__c FROM QuoteLineItem WHERE Quote.Name = '<quote name>'"
```

```bash
sf data query -o dd-e2e -q "SELECT Id, Name, DDCPQ__TargetProduct__r.Name, DDCPQ__FloorPercent__c, DDCPQ__FloorBasis__c, DDCPQ__Action__c, DDCPQ__Status__c FROM DDCPQ__MarginFloorRule__c"
```

```bash
sf data query -o dd-e2e -q "SELECT DDCPQ__TargetProduct__c, DDCPQ__UnitCost__c, DDCPQ__Status__c FROM DDCPQ__RatePlan__c WHERE DDCPQ__TargetProduct__r.Name = '<product name>'"
```

A read-only price preview (never `commit`) to see what the engine says
independent of the cart, once you have the quote, account and product Ids:

```json
{
  "quoteId": "<0Q0…>",
  "selections": [
    {
      "localKey": "c1",
      "productId": "<product 01t…>",
      "quantity": 1,
      "sourceBundleId": "<product 01t…>",
      "manualDiscountPct": 50
    }
  ]
}
```

```bash
sf api request rest "services/apexrest/DDCPQ/dd/v1/price?includeWaterfall=true" -o dd-e2e --method POST --body @body.json
```

Look at `lines[0].waterfall` for the stages named `ManualDiscount` and
`MarginFloor`, and at `rules.lines[0].outcome` (`applied` / `blocked` /
`warned`) for the margin floor's verdict on that line.

## Expected numbers

All runs below against a $100/unit line, quantity 1 unless noted,
verified 2026-10-06 in cpq-pkg via REST `price`/`commit` (no UI).

| Scenario        | Input                                    | Net price                        | Notes                                   |
| --------------- | ---------------------------------------- | -------------------------------- | --------------------------------------- |
| % off           | `manualDiscountPct: 20`, qty 10          | $80/unit, $800 total, ARR $9,600 | plain percent                           |
| $ off           | `manualDiscountAmount: 15`, qty 10       | $85/unit                         | money off each unit                     |
| Set price       | `salesPriceOverride: 42`, qty 10         | $42/unit                         | final; nothing below it ran             |
| % off clamp     | `manualDiscountPct: 150`                 | $0/unit                          | clamped to 100%, not rejected           |
| $ off clamp     | `manualDiscountAmount: 500` (price $100) | $0/unit                          | clamped to the unit price, not negative |
| Floor, Clamp    | 50% off $100, cost 90, Floor % 20 Markup | **$108**/unit, ARR $1,296        | floor (90×1.20) beats the 50%-off $50   |
| Floor, Block    | 50% off $100, cost 90, Floor % 20 Markup | commit refused                   | `FLOOR_BLOCKED`, 0 lines inserted       |
| Floor, Approval | 50% off $100, cost 90, Floor % 20 Markup | **$50**/unit (committed)         | `FloorApprovalRequired__c = true`       |

## Troubleshooting

| Symptom                                                                                      | Cause                                                                                                                                                | Fix                                                                                                                                                              |
| -------------------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Discount icon greyed out or missing on a line                                                | The line is a bundle parent whose options carry the price (`discountable(r)` is false for bundle parents in this build)                              | Discount the child option lines instead; the parent has nothing of its own to discount.                                                                          |
| Net price is higher than the discount you typed                                              | A Margin Floor rule is Active for this product and raised the price                                                                                  | Query `DDCPQ__MarginFloorRule__c` for this product; check `FloorPercent__c`/`FloorBasis__c` against `Cost__c`. Expected, not a bug, when `Action = Clamp`.       |
| Commit fails with `FLOOR_BLOCKED` / "violate an Active MarginFloorRule with Action=Block"    | One or more lines are below an Active `Action=Block` floor. The whole commit is refused, not just that line.                                         | Raise the price above the floor, switch the rule to `Clamp` or `Approval`, or remove the line, then commit again.                                                |
| A floor you just activated seems to do nothing                                               | `Cost__c` is null for the line — the rate plan has no `UnitCost__c` and no caller sent `cost`                                                        | Set Unit Cost on the rate plan (Rate Plan Editor, admin-only field) and reprice. Look for the `FLOOR_NOT_CHECKED` warning / Product 360 finding `FLOOR_NO_COST`. |
| A line discounted 40% by $ off (or Set price) never shows the amber "needs attention" stripe | `ManualDiscountOver15` only reads `ManualDiscount__c` (the percent field) — it does not look at `ManualDiscountAmount__c` or `SalesPriceOverride__c` | Expected behaviour of this build, not a bug. Use the waterfall or line badges to see the real discount on those modes.                                           |
| `FloorApprovalRequired__c` stays true forever                                                | It does not — it clears on the next reprice once the price is back above the floor                                                                   | Reprice the line (change quantity, remove the manual discount, etc.) and re-check.                                                                               |
| `dd_cpq_submit_for_approval` fails with `NO_APPLICABLE_PROCESS`                              | The org has no native Approval Process on Quote — DD CPQ ships none                                                                                  | Build one in Setup → Approval Processes (see "Approvals" above). Not a DD CPQ defect.                                                                            |
| Account standing 5%/3% entitlement vanished after you typed a price                          | Expected — `SalesPriceOverride__c` makes every stage below it, including system discounts, skip itself                                               | Use % off or $ off instead of Set price when the account's own rules should still run.                                                                           |
| A rep can see and edit `Cost__c` on a quote line                                             | FLS/permission set misconfiguration — reps should not read Cost; it was deliberately removed from `CommitService`'s USER_MODE read                   | Check the assigned permission set's field-level security for `Cost__c` on QuoteLineItem; it should not be granted to the User permission set.                    |

## Limits and gotchas

- No approval routing ships with the package. `FloorApprovalRequired__c`
  is a flag; `dd_cpq_submit_for_approval` calls Salesforce's native
  approval REST API against whatever process the subscriber org built.
  A fresh org has none, so the tool fails until an admin configures one.
- Block is all-or-nothing per `commit` call. One blocked line among
  several clean ones zeroes out the whole insert. There is no partial
  commit of "the lines that passed."
- The cart highlight is cosmetic. `ManualDiscountOver15` changes a row
  stripe and a KPI count; it enforces nothing and does not gate commit.
  Do not confuse it with the margin floor, which can.
- The highlight only watches `ManualDiscount__c`. A much larger effective
  discount via `ManualDiscountAmount__c` or `SalesPriceOverride__c`
  triggers no stripe at all.
- `crit` always beats `warn`. A line that is both over-15%-discounted and
  below its margin floor shows only the crit (floor) stripe.
- Governor cost: `MarginFloorService` adds one SOQL to load Active rules
  for the products on the request (zero extra per line); the cost lookup
  itself is free since `ChargeProfileService` already queries the rate
  plan for other fields.
- One mode at a time is a cart-UI rule, not an engine rule. An API caller
  that sends `manualDiscountPct`, `manualDiscountAmount` and
  `salesPriceOverride` together gets the override's behaviour (the other
  two are recorded but functionally skipped) — the cart simply never
  constructs a request shaped like that.
- Bundle parent lines are exempt from margin floor by design; only the
  priced child option lines are checked.

## Questions testers ask

**Q: Why did my 50%-off line not actually drop to half price?**
A margin floor rule for that product is Active and raised (or is about to
refuse) the line. Check `DDCPQ__MarginFloorRule__c` for the product and
compare `FloorPercent__c` and `FloorBasis__c` against `Cost__c`.

**Q: I set a Block floor and now I cannot commit anything, even unrelated
lines on the same quote. Is that a bug?**
No, that is verified behaviour. A `Block` violation anywhere in the
request refuses the entire `commit`, not just the offending line.

**Q: Where do I set a product's cost?**
Rate Plan Editor, the plan's Unit Cost field. It is admin-only by design;
reps never see or edit it.

**Q: Can I combine an account discount with a margin floor?**
Yes, unless you used Set price. A percent or dollar manual discount still
lets every system, promotion and volume rule run; a Set-price override
makes all of them, including the account's standing entitlements, stand
aside. The margin floor still runs either way.

**Q: Why does the cart show a different discount number than what I typed
as "$ off"?**
It should not. $ off is stored and read back as-is
(`ManualDiscountAmount__c`), never converted to or from a percent. If the
numbers disagree, check whether a margin floor also fired on the line.

**Q: Does "Needs approval" on a line mean someone has to approve it before
I can send the quote?**
Not in this build. It is a flag (`FloorApprovalRequired__c`) with no
routing behind it yet. The quote commits and can be sent regardless.

**Q: I built an Approval Process in Setup. Why does DD CPQ not route to it
automatically?**
Nothing watches `FloorApprovalRequired__c` and calls
`dd_cpq_submit_for_approval` for you. Submission is a deliberate, separate
step (REST, MCP, or your own automation), not an engine side-effect of
pricing.

**Q: Is there a limit on how big a $ off or % off can be?**
Yes. % off clamps to 0–100; $ off clamps to 0 and to the line's own
per-unit price, never negative. Set price has no clamp at all; it is a
literal price you name.
