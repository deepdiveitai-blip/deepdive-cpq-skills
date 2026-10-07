# Eligibility: who may buy what

> Installed build 0.1.0-9 · Verified 2026-10-07 in cpq-pkg: a product targeted
> by one Eligibility Rule disappeared from the catalog for the account the
> rule matched (`totalAvailable` 47 vs 48 for an account it didn't match),
> priced the same ($100, qty 10, net $100, ARR $12,000) for both accounts
> while only previewing, then was refused at commit for the blocked account
> with `"EL Target Product is not available for this customer: requires an
approved account tier"` (`code: "CFG_ELIGIBILITY"`) and committed clean for
> the allowed account.

## What it is for

Eligibility decides **which products a given deal may even see**, before
anyone picks a price or a quantity. It is a gate, not a discount: a product
either is or is not on the menu for this Account/Opportunity/Quote. Typical
uses: "Compliance Add-on is Enterprise-tier only", "this SKU is US-only",
"this legacy plan is closed to new logos". Compare with Compatibility
(compatibility.md), which governs products _against each other_ once they
are both already eligible (requires/excludes/max quantity).

Step 6 of the CLAUDE.md engine pipeline — it runs once, early, against the
Quote/Account/Opportunity context, before Selection. Compatibility (step 8)
runs live after that, on every selection change.

## How it works

### Default allow

A fresh org with zero Eligibility Rules sells everything. Admins work by
**exception**: a product is eligible unless a rule says otherwise. There is
no "allow" rule type — every rule's job is to take a product _off_ the menu
for the deals its conditions match. The Rules Studio wizard tile says it
plainly: "Take products off the menu for the deals that match, e.g.
customers outside the US."

### Rule set → rule

- **`EligibilityMatrix__c`** ("Eligibility Rule Set") — a named, versioned
  group of rules switched on together via `Status__c`: `Draft` (default on
  save) → `Active` (evaluated) → `Inactive` (switched off, kept for history).
  Only **Active** sets are read at runtime.
- **`EligibilityRule__c`** ("Eligibility Rule") — one gate. Master-detail to
  its Matrix, so deleting the set deletes its rules.

### What a rule covers

Two ways to say which product(s) a rule is about, read by the shared
`RuleTarget` class:

- **`TargetProduct__c`** ("Applies To Product") — the simple case, one
  product lookup.
- **`Target__c`** ("Products Covered", JSON) — written by Rules Studio when
  you pick several products, a field-based condition on `Product.*`, or a
  Product Group. When present it wins over `TargetProduct__c`.

**Important asymmetry, confirmed by reading `RuleTarget.cls`:** for most rule
objects a blank target means "every product". `EligibilityRule__c` is one of
four objects in `RuleTarget.BLANK_IS_NONE`, where a **blank target means the
rule covers nothing at all** — the opposite of the field's own inline help
text ("Leave blank to apply it to every product"), which is simply wrong for
this object. In practice you cannot hit this through Rules Studio: its
eligibility wizard never offers "every product" as a target (see Limits and
gotchas) — it only shows up if a rule is built directly on the object with
the lookup left blank.

### Criteria — the shared 5-field schema

Each rule also carries the shared criteria fields (CLAUDE.md hard rule 5),
evaluated by the one `CriteriaEvaluator`:

| Field             | Label            | Notes                                                                                                                                                                      |
| ----------------- | ---------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `FieldApiName__c` | Condition Field  | Dotted path, e.g. `Account.Industry`                                                                                                                                       |
| `Operator__c`     | Condition        | `equals`, `not-equals`, `in`, `not-in`, `greater`, `less`, `greater-or-equal`, `less-or-equal`, `between`, `contains`, `starts-with`, `is-null`, `is-not-null`, `includes` |
| `Value__c`        | Condition Value  | Comma-separated for `in`/`between`/`not-in`/`includes`                                                                                                                     |
| `ValueType__c`    | Value Type       | `string` / `number` / `picklist` / `date` / `boolean` / `id`                                                                                                               |
| `Priority__c`     | Evaluation Order | Lower runs first; blank runs last                                                                                                                                          |

Empty Condition Field + Condition = catch-all, matches every deal that
reaches it. For more than one condition, Rules Studio writes `Conditions__c`
(JSON: `{"logic":"1 AND (2 OR 3)","conditions":[...]}`) instead, which
replaces the single-condition fields when present.

### What the condition can read

Confirmed from `CriteriaContextService`: any readable field (custom fields
included) on these roots, not just a short hard-coded list:

| Root            | Example                                                                                    |
| --------------- | ------------------------------------------------------------------------------------------ |
| `Account.*`     | `Account.Industry`, `Account.BillingCountry`, `Account.AnnualRevenue`                      |
| `Opportunity.*` | `Opportunity.Type`, `Opportunity.StageName`                                                |
| `Quote.*`       | `Quote.ExpirationDate`                                                                     |
| `User.*`        | `User.Department` (the person pricing)                                                     |
| `Product.*`     | `Product.Family`, `Product.ProductCode` — used in the _target_ criteria, not the condition |
| `Variable.*`    | a resolved Variable's value                                                                |

Every path is validated against describe and read in `USER_MODE` — an
unreadable or unknown path stays null and the condition does not match (it
never throws).

### Outcome: `Eligible__c`, and how allow/deny combine

`Eligible__c` is a checkbox, default `true` at the database level, but Rules
Studio's eligibility wizard **always saves it `false`** — an eligibility
rule's whole purpose is to hide something, so the UI does not offer "make
this eligible". `EligibleVariable__c` can override it: when set, the
rule's outcome is the resolved Boolean value of that Variable instead of the
static checkbox (e.g. "eligible only if the Variable `HasSignedMsa` is
true"), which lets an admin wire eligibility to something computed rather
than hard-coded.

**How several rules combine, confirmed by reading `EligibilityService`:**

1. Load every **Active** rule that could cover any candidate product
   (`TargetProduct__c` in the set, or blank — blank rows are fetched but may
   still resolve to "covers nothing", see above).
2. Sort the active rules once by `Priority__c` ascending (ties keep
   insertion order) — `RulePrecedence`.
3. **Per product**, keep only the rules that actually cover it
   (`RuleTarget.matches`), and within that set put the rule that **names the
   product directly** ahead of one that covers it by criteria or a group,
   ahead of one that covers "every product" (`RuleTarget.specificity`: 3 / 2
   / 1). The Priority ordering from step 2 is preserved within each
   specificity tier.
4. Walk that ordered list and evaluate each rule's criteria against the
   deal's context. **First rule whose criteria match wins** — its
   `Eligible__c` (or Variable override) decides the product, and evaluation
   for that product stops there. A rule later in the list never gets a
   chance to override an earlier match — there's no explicit "deny always
   wins" merge step; it's strictly first-match-by-specificity-then-priority.
5. If no rule covers the product, or none of the rules that do actually
   match the deal, the product stays eligible — default allow.

So in practice: give your most specific/important deny a low Priority
number (or let its direct product-name targeting naturally outrank a
criteria/group rule), and a catch-all "block everyone except approved
accounts" rule only ever fires when nothing more specific already decided
the product.

**Multiple Active matrices** are allowed at once — there's no "only one
Active set" constraint. All of their Active rules are pooled into the single
candidate list in step 1 above and compete by the same specificity/priority
order; they are not evaluated matrix-by-matrix.

### Where eligibility shows

| Surface                                                                     | Behaviour, confirmed by reading the code                                                                                                                                                                                                                                                                                                                                                                                                                                                                                          |
| --------------------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Catalog / product picker** (`CatalogBrowseService`, `GET /dd/v1/catalog`) | An ineligible product is **removed from the result entirely** — not shown greyed out, not shown with a reason. It simply isn't in `products[]`, and `totalAvailable` drops by one. Verified: 48 products for an allowed account's quote, 47 for a blocked one, same catalog, same quote shape.                                                                                                                                                                                                                                    |
| **Price preview** (`POST /dd/v1/price`)                                     | Does **not** block or warn. A line for an ineligible product still prices normally — `CpqPriceResource`'s response has no warnings field at all. The rule trace (`?includeWaterfall` response `rules.rules[]`) does show the eligibility rule as `evaluated/matched/applied` so you can see it fired, but the price itself is unaffected and there is no refusal.                                                                                                                                                                 |
| **Commit** (`POST /dd/v1/commit`)                                           | **Hard refused.** A new line (not a renewal/amendment line the customer already has) for a product an Active rule hides is collected and the whole commit throws `CommitValidatorService.EligibilityViolationException` before any DML, `code: "CFG_ELIGIBILITY"`. Message format, read from `CpqEngine.ineligibleLines`: `"<Product Name> is not available for this customer: <rule's Message__c>"`, or just `"<Product Name> is not available for this customer."` if `Message__c` is blank. Several refused lines join with `" | "`. |
| **Lines already on the quote**                                              | A component judged with its bundle; a line that arrived by renewal, amendment, or was saved before the rule went live, is **not re-judged** — eligibility only catches _new_ lines at commit. The comment in `CpqEngine` is explicit: "Eligibility hides a product in Add Products; a line that arrives any other way (REST, MCP, a quote saved before the rule went live) is flagged here and refused at commit."                                                                                                                |
| **LWC cart**                                                                | The configurator calls the catalog endpoint to populate Add Products, so an ineligible product never appears there to be added in the first place — the cart-level block is really the catalog-level removal, reached a different way.                                                                                                                                                                                                                                                                                            |

## Objects and fields

| Object label (API name, unprefixed)           | Field (label)                             | Meaning / values                                                                                                    |
| --------------------------------------------- | ----------------------------------------- | ------------------------------------------------------------------------------------------------------------------- |
| Eligibility Rule Set (`EligibilityMatrix__c`) | Rule Set Name (`Name`)                    | The set's name                                                                                                      |
|                                               | Status (`Status__c`)                      | `Draft` (default) / `Active` / `Inactive`; only `Active` is evaluated                                               |
| Eligibility Rule (`EligibilityRule__c`)       | Eligibility Matrix (`Matrix__c`)          | Master-detail to its rule set                                                                                       |
|                                               | Applies To Product (`TargetProduct__c`)   | Single-product target; blank means "covers nothing" for this object (see above)                                     |
|                                               | Products Covered (`Target__c`)            | JSON target — several products, `Product.*` criteria, or a Product Group; wins over `TargetProduct__c` when present |
|                                               | Eligible (`Eligible__c`)                  | Checkbox; outcome when criteria match. Rules Studio always saves `false`                                            |
|                                               | Eligible Variable (`EligibleVariable__c`) | Optional lookup to an Active `Variable__c`; its resolved Boolean overrides `Eligible__c`                            |
|                                               | Condition Field (`FieldApiName__c`)       | Dotted path, e.g. `Account.Industry`. Blank + blank Condition = catch-all                                           |
|                                               | Condition (`Operator__c`)                 | See operator table above                                                                                            |
|                                               | Condition Value (`Value__c`)              | Comma-separated for multi-value operators                                                                           |
|                                               | Value Type (`ValueType__c`)               | `string` / `number` / `picklist` / `date` / `boolean` / `id`                                                        |
|                                               | Conditions (Advanced) (`Conditions__c`)   | JSON, several conditions + AND/OR logic; replaces the single-condition fields when filled. Written by Rules Studio  |
|                                               | Evaluation Order (`Priority__c`)          | Lower runs first; blank runs last                                                                                   |
|                                               | Message (`Message__c`)                    | Shown to the rep after "`<Product>` is not available for this customer:"                                            |
|                                               | Description (`Description__c`)            | Admin's own note on why the rule exists; not shown to the rep                                                       |

## Build it (tester)

This builds on the golden path's org (Acme Corp / Healthcare, the Q-0042
quote already committed). Add one more account and reuse Compliance Add-on
as the target so you don't need new products.

1. **Account** → New → `Globex Inc`, Industry **Government** (anything other
   than Healthcare). Save.
2. **Opportunity** on Globex Inc → `Globex Renewal`, Stage **Proposal/Price
   Quote**, any close date. Save.
3. From that Opportunity's Quotes related list → **New Quote** →
   `Q-0050 Globex`. Save.
4. **App Launcher → DeepDive CPQ → Rules Studio tab.**
5. **+ New Rule Set** (or the equivalent button in your build) → pick the
   wizard tile **"Hide a product from some deals"** (type `eligibility`).
   Name the set `Compliance Gate`.
6. **+ New Rule** inside it:
   - **For**: choose a product → **Compliance Add-on**.
   - The statement reads "**Compliance Add-on** is not available on the
     quote."
   - **Why, for the rep**: type `requires Healthcare industry`.
   - Add a condition: **Condition Field** `Account.Industry`, **Condition**
     `equals`, **Condition Value** `Healthcare`, **Value Type** `picklist`.

     This rule hides Compliance Add-on **unless** the account is Healthcare
     — so to get a deny rule that _blocks the non-Healthcare account_, the
     condition has to describe the deals you want blocked. If Rules Studio's
     wizard phrases the condition as "allow when", flip it: use
     `Account.Industry` `not-equals` `Healthcare` instead, so the rule's
     criteria match exactly the deals that should lose the product.
7. Save → the set starts life **Draft**. Open it and switch **Status** to
   **Active**.
8. Open `Q-0050 Globex` → **Configure Products**. Try **Add products** and
   search for "Compliance" — it should not appear. Open `Q-0042 Acme`
   (Healthcare) instead and search — it should still be there.
9. If you added Compliance Add-on to Q-0050 _before_ activating the rule
   (or via a saved/renewed line), note that committing any _new_ selection
   of it afterward is what gets refused — a line already sitting on the
   quote is not retroactively kicked off.

## Check it (Claude)

```bash
sf data query -o cpq-pkg -q "SELECT Id, Name, DDCPQ__Status__c FROM DDCPQ__EligibilityMatrix__c"
sf data query -o cpq-pkg -q "SELECT DDCPQ__Matrix__r.Name, DDCPQ__Matrix__r.DDCPQ__Status__c, DDCPQ__TargetProduct__r.Name, DDCPQ__Eligible__c, DDCPQ__FieldApiName__c, DDCPQ__Operator__c, DDCPQ__Value__c, DDCPQ__ValueType__c, DDCPQ__Priority__c, DDCPQ__Message__c FROM DDCPQ__EligibilityRule__c"
```

Expect the set **Active**, the rule's `TargetProduct__r.Name` = Compliance
Add-on, `Eligible__c` = false, and the condition matching what you typed.

Read-only price preview (never commit on the tester's behalf — that's their
step): write the body to a file with the Globex quote and Compliance
Add-on's Id, then

```bash
sf api request rest "services/apexrest/DDCPQ/dd/v1/price?includeWaterfall=true" -o cpq-pkg --method POST --body @body.json
```

The response prices the line as if nothing were wrong (eligibility does not
block price) — the tell is in `rules.rules[]`: an entry with
`"ruleType": "eligibility"` and `"applied": 1` means the rule matched this
deal. The real proof is the catalog call or the commit refusal, both of
which the tester does by clicking — you read the result afterward with
`sf data query` on `EligibilityRule__c`/`EligibilityMatrix__c` and by asking
the tester what the Add Products search showed, or what message Commit gave.

## Expected numbers

All from a live run in cpq-pkg: one `EligibilityRule__c` (Active, in an
Active matrix) targeting `EL Target Product` by `TargetProduct__c`,
condition `Account.Name equals "EL Blocked Co"`, `Eligible__c = false`,
`Message__c = "requires an approved account tier"`. Two accounts, two
quotes, same product, same $100 Standard Price, same Active per-unit
monthly rate plan.

| Call                                              | Allowed account's quote                                                 | Blocked account's quote                                                                                                                                                                                               |
| ------------------------------------------------- | ----------------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `GET /dd/v1/catalog`                              | `totalAvailable: 48`, EL Target Product **present**                     | `totalAvailable: 47`, EL Target Product **absent** (EL Control Product, untargeted, present in both)                                                                                                                  |
| `POST /dd/v1/price?includeWaterfall=true`, qty 10 | `netPrice: 100`, `totalNet: 1000`                                       | **Same**: `netPrice: 100`, `totalNet: 1000` — price does not enforce eligibility                                                                                                                                      |
| `rules.rules[]` entry for the eligibility rule    | `matched: 0, applied: 0` (rule evaluated, didn't match this account)    | `matched: 1, applied: 1` (rule fired)                                                                                                                                                                                 |
| `POST /dd/v1/commit`, qty 10                      | Succeeds: `linesInserted: 1`, QLI `NetPrice__c = 100`, `ARR__c = 12000` | **Refused**: `code: "CFG_ELIGIBILITY"`, `errorType: "DDCPQ.CommitValidatorService.EligibilityViolationException"`, `error: "EL Target Product is not available for this customer: requires an approved account tier"` |

## Troubleshooting

| Symptom                                                                              | Cause                                                                                                                                                                                                                                                                            | Fix                                                                                                                                                               |
| ------------------------------------------------------------------------------------ | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Product still shows in Add Products for the account you meant to block               | Matrix is still **Draft** or **Inactive**                                                                                                                                                                                                                                        | Open the Eligibility Rule Set, switch **Status** to **Active**                                                                                                    |
| Product still shows for the blocked account, matrix is Active                        | The rule's condition doesn't actually match this deal — check the exact operator/value, and that `ValueType__c` matches the field (a picklist compared as `string` with the wrong casing silently fails)                                                                         | Re-check `FieldApiName__c` / `Operator__c` / `Value__c` against the account's real values with a `sf data query`                                                  |
| Product disappeared for **every** account, including ones you didn't mean to block   | `TargetProduct__c` and `Target__c` are both blank on the rule — for `EligibilityRule__c` that means "covers nothing" normally, but if the rule was created with an _explicit_ `Target__c` of `{"mode":"All"}` it covers everything; or the condition itself is blank (catch-all) | Check `Target__c`/`TargetProduct__c` names the right product(s); check `FieldApiName__c` isn't blank when it should carry a condition                             |
| Committing a quote throws `CFG_ELIGIBILITY` for a line you expected to be allowed    | A rule with **no `Message__c`** and a condition you didn't expect is matching — the error only tells you the product and the message, not which rule                                                                                                                             | Query `EligibilityRule__c` for Active rules targeting that product, and check each one's condition against the quote's Account/Opportunity by hand                |
| A line the customer already owns (renewal/amendment) gets blocked anyway             | Shouldn't happen — `CpqEngine.ineligibleLines` explicitly skips lines with `sourceAssetId` or `baselineLineId` set. If it does, the line may not actually be flagged as a renewal/amendment line (check those fields are populated)                                              | File it — this is the one case the design says should never be judged                                                                                             |
| Eligible product missing from the catalog, but no Eligibility Rule targets it at all | It's being filtered by something else first (a Catalog membership, Pricebook entry inactive, or the product itself `IsActive = false`) — `totalAvailable` already excludes it before eligibility even runs                                                                       | Check `PricebookEntry.IsActive` and `Product2.IsActive` before assuming it's an eligibility rule                                                                  |
| You left `TargetProduct__c` blank expecting "applies to every product"               | That's the documented field help text, but it's wrong for this object — a blank target on `EligibilityRule__c` matches **nothing** (`RuleTarget.BLANK_IS_NONE`)                                                                                                                  | Target a specific product/group/criteria explicitly; Rules Studio's wizard already forces this choice, so this only bites a rule built directly on the raw object |
| Rule seems to fire for the wrong deal when two rules both cover the same product     | Specificity order, not just Priority: a rule naming the product directly always outranks a criteria/group rule on the same product regardless of `Priority__c`                                                                                                                   | Give the more deliberate rule the narrower target (name the product), not just a lower Priority number                                                            |

## Limits and gotchas

- **No "allow" rule exists.** You cannot write a rule that makes a product
  eligible; you can only write rules that take it away. If a product
  should be available everywhere except one account, write the deny rule
  scoped to that one account — don't try to invert it with an "allow"
  rule, there isn't one.
- **Price preview never blocks or warns on eligibility.** Only Commit does.
  A rep (or an agent calling the REST API directly) can price an
  ineligible line all day; the gate only closes at Commit, and only for
  _new_ lines.
- **Renewal/amendment/pre-existing lines are never re-judged.** Making a
  product ineligible after a customer already has it on a quote does not
  retroactively remove it.
- **`TargetProduct__c`/`Target__c` blank = covers nothing**, the opposite of
  the field's own inline help text. Not reachable through Rules Studio's
  wizard, which always makes you pick a target — only a risk if a rule is
  created directly on the object.
- **No per-rule audit of which rule fired**, beyond the rule trace's
  aggregate counts (`matched`/`applied`) and the Message text in the
  commit refusal. If two Active rules could both cover a product, you have
  to work out which one actually won from Priority + specificity by hand;
  the response doesn't name the winning rule's Id back to you.
- **Multiple Active matrices pool together** rather than being evaluated
  independently matrix-by-matrix — there is no "only the highest-priority
  matrix counts" semantic.
- **Governor limits**: `EligibilityService` loads all qualifying Active
  rules and the context fields they reference in a bounded number of
  SOQL calls regardless of candidate product count (bulk-safe); nothing
  queries per product or per line.

## Questions testers ask

**Q: I made Compliance Add-on eligible=false with no condition at all — now it's gone for everyone, is that a bug?**
No. A rule with a blank Condition Field/Condition is a catch-all — it
matches every deal that reaches it. That's expected: scope the condition
if you only want it blocked for some accounts.

**Q: Why does the price preview still show a price for a product the account can't buy?**
Because eligibility is enforced at the catalog (hiding it from Add
Products) and at Commit (refusing the line) — not inside the pricing
waterfall. If a selection for that product is sent to `/dd/v1/price`
directly, it prices normally; nothing downstream of Selection re-checks
eligibility until Commit.

**Q: Can I make a rule that requires a signed contract (a lookup, not a
picklist) before a product is eligible?**
Yes, via `EligibleVariable__c` — point it at an Active Variable that
resolves to true/false for "has a signed MSA" or similar, and leave
`Eligible__c`'s static value unused; the Variable's resolved value decides.

**Q: Two Active rules both cover the same product — which one wins?**
Whichever one is more specific (names the product directly > criteria/group

> "every product"), and within the same specificity, whichever has the
> lower `Priority__c` (Evaluation Order). First match wins; evaluation stops
> there for that product.

**Q: Does an inactive rate plan on the product also block it, or is that a
separate thing from eligibility?**
Separate. No rate plan means the product can still be added and even
priced (flat monthly, legacy fallback — see the golden path's Stage 6), and
it is refused at **Commit** for a different reason
("...Each needs an Active rate plan..."), not `CFG_ELIGIBILITY`. The two
checks can both apply to the same line but throw different errors.

**Q: If I activate the matrix while a rep already has the ineligible
product open in their cart, does it disappear live?**
Not verified in this build — the catalog is read fresh on each Add
Products search, so a _new_ search will no longer show it, but a line
already added to the in-progress cart is not proactively removed; it would
only be caught if the rep tries to Commit.
