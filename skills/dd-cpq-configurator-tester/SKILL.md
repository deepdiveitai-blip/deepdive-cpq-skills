---
name: dd-cpq-configurator-tester
description: >-
  Adversarial configurator testing for DD CPQ on Claude Cowork. Covers
  bundles, option constraints, quantity rules, the Compatibility and
  Eligibility matrices, Variables and the cart display layer — the
  Configure half of CPQ, which pricing testing never touches. Works by
  comparing the authored rule, the engine's behaviour and what the
  screen shows, then proving all three agree.

  REPORT-ONLY: never writes a rule, edits a bundle or changes code. It
  may create throwaway quotes, since a configurator cannot be tested
  without configuring something.

  Trigger on: "test the configurator", "test bundles", "test option
  constraints", "test quantity rules", "check the compatibility
  matrix", "test the cart display", "find configurator bugs".

  Skip for pricing or Timeline regressions (dd-cpq-pricing-tester),
  rate plan authoring (dd-cpq-rate-plan-authoring), building a real
  quote (dd-cpq-quoting-agent), demos (dd-cpq-demo).
---

# DD CPQ Configurator Tester · operating guide

You are testing the **Configure** half of CPQ: what a rep may select,
what the rules do about it, what quantity results, and what the screen
then says. Pricing is somebody else's job — you touch prices only where
a configuration decision should have changed one.

## What makes you better than a human tester

A person opens the cart, clicks through a bundle, and checks whether the
result looks right. That finds broken happy paths. It does not find:

1. **A rule that exists but never fires.** The record is authored, the
   admin sees it in the list, and the engine never applies it. On screen
   nothing is wrong — there is simply no error where there should be one.
   You catch it by reading the rule and then provoking it.

2. **A layer disagreeing with the layer beneath it.** The engine prices
   a line as Package and the cart labels it PerUnit. Every number is
   correct; the word is wrong. This exact defect shipped and survived
   two full pricing runs because nothing compared the two.

3. **Combinations nobody clicks.** Four features with nine options is 47
   valid selections and a few hundred invalid ones. A person tests five.

4. **Silent defaults.** A quantity rule that stopped firing leaves the
   default quantity behind, which is a plausible number. Only the rule
   tells you it should have been something else.

Structure every run around those four. If a finding could have been
found by clicking once, it is not where your value is.

## Hard rules

1. **Never write a rule, constraint, feature, option or variable.** You
   read the configuration; you never author it. `dd_cpq_list_rules` is
   read-only by design and the admin write actions are deliberately not
   exposed. If you find yourself wanting to fix a seed, report it.

2. **Never modify code.** Route every defect to the human (Vijay).

3. **Throwaway quotes only.** You may `create_quote`, `configure_bundle`
   and `add_line_item` — a configurator cannot be tested without
   configuring. Name every quote `CFGTEST <scenario> <timestamp>` so it
   is obvious what to delete. Never touch a quote you did not create,
   except to READ it.

4. **Never leave a shared baseline quote modified.** `Rate Plan Coverage
Quote`, `Q-0042 Acme (golden $101.83)` and `Acme Cloud Platform Quote`
   belong to other suites. Read them; do not commit to them.

5. **`localKey` matters only when you write the payload yourself.**
   It is the primary key for per-line results across the engine: a null
   one makes every line take the last line's price and flags each lone
   line as its own bundle parent. The engine fills one in now, but a
   caller who supplies their own can match a result back to the line
   that caused it.

   **`configure_bundle` and `add_line_item` generate keys internally**
   and their schemas do not accept one — a selection there is
   `{optionId, quantity}`. Do not report the absence as a defect. The
   rule applies to raw engine payloads: `DdCpqApi`, `/dd/v1/price`,
   `/dd/v1/commit`, and any Apex you write.

6. **A Block that does not block is the most serious thing you can
   find.** It means a customer can sell an invalid configuration. Rank
   it above any wrong number.

## The five layers, and the tools that read each

| Layer            | Question                      | Tool                                                                            |
| ---------------- | ----------------------------- | ------------------------------------------------------------------------------- |
| Catalogue        | what may this quote see?      | `dd_cpq_list_products({quoteId})`                                               |
| Structure        | what does the bundle offer?   | `dd_cpq_get_bundle_structure`                                                   |
| Authored rules   | what did the admin configure? | `dd_cpq_list_rules`                                                             |
| Engine behaviour | what does it actually do?     | `dd_cpq_configure_bundle`, `dd_cpq_check_compatibility`, `dd_cpq_explain_price` |
| Display          | what does the rep see?        | `dd_cpq_get_cart`                                                               |

**The method is always the same: read two adjacent layers and prove they
agree.** A single layer read in isolation can only tell you it is
internally consistent, which it almost always is.

## Quickstart · a full run

1. `dd_cpq_current_user` once, to know who you are running as.
2. `dd_cpq_find_products` → pick the bundles under test.
3. For each bundle: `dd_cpq_get_bundle_structure`, then
   `dd_cpq_list_rules({ruleType:'optionConstraints', bundleProductId})`.
   **Cross-check them** — see [Constraint parity](#a-constraint-parity).
4. Run the five test families below in order. Families A and B are the
   ones that find real defects; do them first even on a short run.
5. Write the HTML report to
   `~/Downloads/dd-cpq-configurator-YYYYMMDD-HHMM.html` and print the
   absolute path. Never overwrite a previous report.
6. Delete every `CFGTEST` quote you created, or list them in the report
   footer if deletion is not available to you.

---

## Family A · Option constraints

The Acme Cloud Platform bundle in `cpq-dev` is seeded to fail on
purpose. Four constraints, three Block and one Warning:

| Trigger            | Requires                     | Severity |
| ------------------ | ---------------------------- | -------- |
| Advanced Analytics | Edition — Enterprise (exact) | Block    |
| API Gateway        | **tier rank ≥ 20** (floor)   | Block    |
| Premium Support    | excludes Edition — Starter   | Warning  |
| Archive Storage    | Extra Storage 1TB (exact)    | Block    |

### Constraints come in TWO styles — do not report the second as malformed

**Exact match** names one `TargetOption__c`. That is every constraint
you have seen before.

**Tier floor** (2026-09-15) has **no target at all**. It carries criteria
instead, and is satisfied by rank rather than by one specific option:

```
FieldApiName__c = Tier.SelectedRank
Operator__c     = greater-or-equal
Value__c        = 20
targetOption    = null          ← correct, not missing
```

`Tier.SelectedRank` is the **highest** `ProductTier__c.Rank__c` among
the products selected. Ranks in `cpq-dev`: Starter 10, Professional 20,
Enterprise 30.

A floor constraint exists because "at least Professional" is
unrepresentable as an exact match — it named Professional, so Enterprise
was blocked, and Analytics and API Gateway became unsellable together.

**A constraint with a null target and non-null `FieldApiName__c` is
correct.** A constraint with neither is inert, and the
`Target_Or_Criteria_Required` validation rule now refuses to save one.

### A. Constraint parity

Compare `get_bundle_structure` (each option's `requires` / `excludes`)
against `list_rules({ruleType:'optionConstraints'})`. Every authored
constraint must appear on the structure, with the same severity and
message.

**Match on the right thing for each style**: an exact-match constraint by
its target option; a floor by its criteria. Expecting a target on a floor
produces a false finding.

A constraint present in `list_rules` but absent from the structure is a
**rule that can never fire** — finding type 1, and invisible on screen.

### B. Provoke every Block

For each Require constraint, configure the trigger option **without**
its target and commit. Expected: the commit is refused and the message
names the reason.

```
Advanced Analytics + Edition — Starter   -> BLOCKED
  "Advanced Analytics needs the Enterprise edition."
```

Then configure it **with** the target and confirm it succeeds. A
constraint that blocks everything is as broken as one that blocks
nothing.

**A tier floor needs three points, not two**, because the interesting
failure is at the boundary and above it:

| Selection                  | Rank | Expected                             |
| -------------------------- | ---- | ------------------------------------ |
| Starter + API Gateway      | 10   | BLOCKED                              |
| Professional + API Gateway | 20   | ALLOWED — a floor of 20 is met BY 20 |
| Enterprise + API Gateway   | 30   | ALLOWED — higher clears it           |

The middle row is the one people forget, and an off-by-one in the
operator passes the other two.

**Then prove the pair is sellable.** `Enterprise + Advanced Analytics +
API Gateway` must commit — roughly $23,520 across 5 lines. That
combination was impossible in every configuration before the floor
existed, and it is the business outcome the whole change is for.

### C. Warning must NOT block

`Premium Support` + `Edition — Starter` is severity Warning. It must
**commit successfully** and surface the message. A Warning that blocks
is a false negative that will cost a customer a sale; report it as
High.

### D. The inverse direction

Constraints are directional. `Archive Storage` requires `Extra Storage
1TB`, but `Extra Storage 1TB` alone must be perfectly legal. Test the
reverse of every constraint and confirm it is allowed. An accidentally
bidirectional constraint is a classic authoring bug that no forward test
detects.

---

## Family B · Quantity rules

`get_bundle_structure` returns eight quantity fields per option. They
were invisible to MCP until 2026-09-14, so this family found real
defects on its first two runs; both are fixed and the expected
behaviour below has changed accordingly.

| Field                                             | What to prove                                                                                                                                                                                                                                                                                    |
| ------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| `quantityRule`                                    | `Match with Parent`: set the bundle to N, commit, assert the child's committed quantity is exactly N. Test N = 1, 7, 50 — a rule that stopped firing leaves a plausible default behind.                                                                                                          |
| `quantityEditable`                                | `false` means the override is ignored and the rule's quantity used — and the response now says so. Expect a `QuantityOverrideIgnored` entry in `warnings[]` naming the option, the ignored value and the quantity actually applied, carrying the line's `localKey`. Silence here is the finding. |
| `minQty` / `maxQty`                               | Commit at min, at max, at min−1 and max+1. The two outer cases are now **REFUSED** — HTTP 400, `errorType: ConfigurationRejected`, naming the option and the bound. Not clamped: a ceiling is a business rule, and silently selling 10 when the rep typed 11 is its own wrong answer.            |
| `quantityMultiplier` + `quantityDrivenByOptionId` | Change the driving option's quantity and assert the driven option moves by the multiplier.                                                                                                                                                                                                       |
| `quantityVariableId`                              | Resolve the variable with `dd_cpq_test_variable`, then confirm the committed quantity equals the resolved value.                                                                                                                                                                                 |

**Read the committed quantity from `dd_cpq_get_cart`, not from your own
request.** The request says what you asked for; the cart says what the
rep will see. Where they differ, the cart is the finding.

In the Acme bundle: `Named User Pack` is `Match with Parent` and NOT
editable, and `Extra Storage 1TB` is editable with bounds 1–10 and a
default of 2. In CRM Suite Pro, `Storage +100GB` is editable with a
ceiling of 20.

**Bounds are enforced server-side, not in the MCP tool's pre-check.** A
rejection therefore costs a round trip, while a Block constraint is
caught before any network call. That is a deliberate difference, not a
regression — do not report the round trip.

### E. Cross-line requirements — the fourth constraint layer

`QuoteLineRequirement__c` is separate from all three above, and easy to
miss because it fires at a different moment. It says **"if this product
is on the quote, at least one of these companions must be too"** — the
must-be-sold-with pattern — and it is enforced at step 11.5 of the
pipeline by `CommitValidatorService`, AFTER pricing and immediately
BEFORE lines are persisted.

Its companions are `QuoteLineRequirementCompanion__c` child rows with
any-of semantics: one present companion satisfies the rule.

Read the rules with
`dd_cpq_list_rules({ruleType:'quoteLineRequirements'})`. The companion
rows are separate records — if you need them, query
`QuoteLineRequirementCompanion__c` with `dd_cpq_soql` and join by
parent id.

For each rule:

- Commit the trigger product with **no** companion. Hard severity must
  block; Warning must commit and return the message.
- Commit it **with** one companion and confirm it succeeds — any-of
  means one is enough, so a rule demanding all of them is a defect.
- Commit a companion **alone**. Requirements are directional; the
  companion by itself must be legal.

Because this fires after pricing, a Hard failure means the rep watched
a price appear and then lost it at commit. Note in the report whether
the message named the missing companion — if it did not, the rep cannot
act on it.

---

## Family C · Compatibility and eligibility

These are the cross-line rules — they govern what may sit on a quote
together, regardless of which bundle each line came from.

1. `dd_cpq_list_rules({ruleType:'compatibilityRules'})` — read what is
   authored. **If this returns an empty list, say so plainly and stop
   this family.** Reporting "compatibility passed" when no rule exists
   is a false green, and it is the easiest mistake to make here.
2. For each rule, send the product combination to
   `dd_cpq_check_compatibility` and assert the expected action comes
   back with the authored message.
3. Send a combination the rule does NOT cover and assert `actions` is
   empty. Over-firing is as much a defect as under-firing.
4. Commit the blocked combination through `configure_bundle` /
   `add_line_item` and confirm the engine refuses it. The matrix
   advising the UI and the engine enforcing it are different code paths;
   test both. (Cross-line exclusion enforcement is DDCPQ-016.)
5. Eligibility: `dd_cpq_list_products({quoteId})` is already filtered by
   the Eligibility Matrix. Compare against
   `list_rules({ruleType:'eligibilityRules'})` and against the full
   catalogue — a product that should be hidden and is not, or vice
   versa, is the finding.

**Empty criteria means catch-all, not inert.** A rule with no criteria
matches every line. That is the single most common authoring mistake, so
read the criteria before concluding a rule is scoped.

### Seeded 2026-09-15 — this family now has data

It reported SKIPPED on the first two runs because every rule type stood
at zero. One record of each now exists, chosen so both halves are
testable:

| Rule                   | Seeded                                                       | The negative case                        |
| ---------------------- | ------------------------------------------------------------ | ---------------------------------------- |
| compatibility          | Premium Support excludes Archive Storage, Hard               | either alone is fine                     |
| eligibility            | Workspace Pro visible only to Technology accounts            | a non-Technology account must NOT see it |
| quote-line requirement | Advanced Analytics needs Named User Pack **or** Collab Seats | either one alone satisfies it            |

The eligibility rule is criteria-gated rather than catch-all
deliberately, so the _hidden_ half can be proven. A catch-all would look
like it passed while testing nothing.

### Evaluation scope changes WHERE a constraint applies

`OptionConstraint__c.EvaluationScope__c` (2026-09-15):

| Scope         | Enforced                                                                                             |
| ------------- | ---------------------------------------------------------------------------------------------------- |
| `Bundle_Only` | inside `configure_bundle` only — the default, and what every constraint did before the field existed |
| `Cross_Line`  | bundle **and** quote level, against the Virtual State                                                |
| `Quote_Only`  | quote level only                                                                                     |

**A `Bundle_Only` constraint being bypassable line-by-line is expected,
not a defect.** That is what the scope means. Report it only for a rule
marked `Cross_Line` — check `list_rules` before concluding.

The API Gateway floor is `Cross_Line`, so it is the one to test both
ways: through `configure_bundle`, and as standalone `add_line_item`
calls that never touch a bundle.

### The Virtual State includes purchase history

A `Cross_Line` rule evaluates against the cart **plus the account's
installed Assets**:

```
[ cart line products ] + [ Assets ] → Virtual State → rank evaluation
```

So a customer who bought Enterprise last year can buy an add-on whose
floor is Enterprise, with **no edition on today's quote at all**. If you
see a floor satisfied by something that is not in the cart, check
`Asset` on the account before reporting it — that is the feature.

Counted: `Installed`, `Purchased`, `Shipped`, `Registered`. Excluded:
`Obsolete`, and anything flagged `IsCompetitorProduct`.

---

## Family C2 · Waterfall stage → rule traceability

Every stage the waterfall shows was produced by a record somewhere. As
of 2026-09-14 all of them are readable, so every stage can be traced:

| Waterfall stage   | `ruleType`                        |
| ----------------- | --------------------------------- |
| DerivedListPrice  | `pricingRules`                    |
| ContractPrice     | `contractPriceRules`              |
| TermAdjustedPrice | `termDiscountCurves`              |
| UsageTierPrice    | `usageTierMatrices`, `usageTiers` |
| SystemDiscount    | `systemDiscountRules`             |
| VolumeDiscount    | `volumeTiers`                     |
| PromotionDiscount | `promotionRules`                  |
| ChannelDiscount   | `channelDiscountRules`            |
| CommitDiscount    | `commitmentDiscountRules`         |
| MarginFloor       | `marginFloorRules`                |

Run it both directions — each catches a different defect:

**Stage → rule.** For every stage marked `applied` on a line, find the
record that produced it. A stage that fired with no matching Active rule
is the engine inventing a price adjustment.

**Rule → stage.** For every Active rule whose criteria the line
satisfies, confirm the matching stage actually fired. An Active rule that
never fires is invisible on screen — there is simply no discount where
there should be one, and the number looks perfectly ordinary. This is the
single highest-value check in the whole skill, because it is the one a
person cannot perform at all.

Read the criteria before deciding a rule should have matched. **Empty
criteria means catch-all, not inert.** Twelve rule objects share the same
5-field schema (FieldApiName, Operator, Value, ValueType, Priority), all
evaluated by one `CriteriaEvaluator`, and a rule with no criteria matches
every line.

`renewalUpliftRules` applies only on renewal quotes — do not report it
missing on a net-new one.

**Seeded 2026-09-15**, so this family can finally run in both
directions:

| Rule                   | Seeded in `cpq-dev`                  | Stage it should produce |
| ---------------------- | ------------------------------------ | ----------------------- |
| `promotionRules`       | CFG10, 10% off Collab Seats          | PromotionDiscount       |
| `channelDiscountRules` | Gold tier, 5% off Workspace Pro      | ChannelDiscount         |
| `marginFloorRules`     | 20% floor on Support Retainer, Block | MarginFloor             |

Everything else still stands at zero. Say which ones are empty rather
than reporting the family as passing — a stage with no authored rule is
untested, not clean.

---

## Family D · The display layer

This is where the bugs nobody else can find live. `dd_cpq_get_cart`
returns exactly what the LWC renders from; every other tool returns what
the engine computed or what the QLI stored.

For every line, prove the cart agrees with the record beneath it:

| Cart field                       | Must match                                         |
| -------------------------------- | -------------------------------------------------- |
| `pricingModel`                   | the bound `RatePlan__c.PricingModel__c`            |
| `chargeType`                     | the plan's `RevenueNature__c`                      |
| `ratePlanId`                     | the QLI's `RatePlan__c`                            |
| `netPrice`                       | `explain_price`'s Net stage for that line          |
| `quantity`                       | the option's quantity rule, not its default        |
| `isBundleParent` / `parentQliId` | the structure's parentage                          |
| `totalArr`                       | `MRR × 12` for Recurring; `0`/projection for Usage |

**This family already has a scalp.** On the day the endpoint shipped it
showed every saved line labelled `PerUnit` regardless of plan, because
the cart resolved pricing model from `ProductChargeProfile__c` — an
object deprecated a release earlier, holding zero records. Prices were
right, the label was wrong, and two full pricing runs had passed over
it. Fixed in DDCPQ-CFG-03.

Note the shape of that bug, because others will look like it: the
preview path was correct and the **reload** path was wrong. Adding a
line showed the right value; reopening the quote showed the default. So
**always reload the cart after committing** rather than trusting the
commit response.

---

## Family D3 · Add products — the catalog picker

Since DDCPQ-CATALOG-01 the drawer is fed by the engine's own catalog
builder, so it has a tool-readable twin: `dd_cpq_list_products` returns
the same rows, and passing its `catalogId` narrows them the same way.
Every claim on the screen can be checked against that call.

| On the screen                                   | Must match                                                                                                        |
| ----------------------------------------------- | ----------------------------------------------------------------------------------------------------------------- |
| A product is listed at all                      | an active PricebookEntry on the **quote's** price book (the Opportunity's wins), and the Eligibility Matrix       |
| `Standalone SKU`                                | no ProductFeature__c children                                                                                     |
| `Configurable bundle`                           | at least one feature with `MinSelections__c` < `MaxSelections__c` (or Max blank / 0)                              |
| `Static bundle`                                 | features exist and every one has Min = Max — **or** an Active BundleDefinition__c saying `BundleType__c` = Static |
| `N features · N options`                        | ProductFeature__c and ProductOption__c counts for that parent                                                     |
| Pricing Mode / Header Treatment on a bundle row | the Active BundleDefinition__c picklist **labels**, not API values                                                |
| `no bundle definition`                          | a bundle with no Active BundleDefinition__c — it is on the legacy roll-up path                                    |
| The Catalog dropdown exists                     | at least one Active Catalog__c. No catalogs, no dropdown — that is correct, not a defect                          |
| A catalog option's count                        | that catalog's products that ALSO survived price book + eligibility, never its raw CatalogProduct__c count        |
| `featured` tag                                  | `CatalogProduct__c.Featured__c`                                                                                   |

**Group by is the answer to "why is this grouped like that".** Family
groups on `Product2.Family`; Kind groups on the three types above. The
chips carry counts and **All** is always first. Switching the grouping
must clear the selected chip — a chip left selected from the other
grouping would filter to a bucket that no longer exists, and that IS a
defect.

**The classic false positive:** a product the rep expects but cannot see.
Check the quote's price book before writing it up. The drawer deliberately
does not offer what the engine could not price, and the count line says
which scope it is reporting.

## Family D2 · The Bundle Studio — the configure screen

Since 2026-09-18 the ⚙ on a bundle row in the ledger opens
`cpqBundleStudio`: feature rail on the left, options in the middle, live
summary on the right. Options are indented one level under their feature
behind a tree line — a flat list where the feature and its options share a
left edge is a defect (DDCPQ-STUDIO-05). It renders the same structure
`dd_cpq_get_bundle_structure` returns and prices through the same preview
the ledger uses, so everything on it has a tool-readable twin. A human
tester reads the screen; you read the twin; the defect is the gap.

| On the screen                                                      | Must match                                                                                                                             |
| ------------------------------------------------------------------ | -------------------------------------------------------------------------------------------------------------------------------------- |
| Feature pill `Required · pick 1–2` / `Optional · up to 3`          | `minSelections` / `maxSelections`; a Radio feature reads `pick one`                                                                    |
| `recommended` tag on an option                                     | `defaultSelected`                                                                                                                      |
| Quantity chip `= bundle qty`, `bundle qty ×0.1`, `= <var>`, `1–20` | `quantityRule`, `quantityMultiplier`, `quantityOperator`, `minQty` / `maxQty`. A stepper appears only for Static or `quantityEditable` |
| A picked option's amount                                           | `dd_cpq_get_cart` `netPrice × quantity` for that child after Save + Commit                                                             |
| `+$X` on an unpicked option                                        | the unit price the engine gives that child on this quote (one all-options preview on open) × the rule's estimated quantity             |
| `Included` instead of a price                                      | `pricingRole` Included, or the engine returned net 0                                                                                   |
| Greyed option saying `P excludes this`                             | an Exclude constraint, severity Block, with P as trigger                                                                               |
| `Needs T` on a picked option and Save reading `fix N above`        | a Require constraint, severity Block. A Warning shows the same words but Save stays enabled                                            |
| Picking one option silently adds another                           | the Require cascade — the same targets `dd_cpq_configure_bundle` insists on                                                            |
| Amber rail step and Save reading `Fill required slots`             | a feature below `minSelections`                                                                                                        |
| Feature header subtotal                                            | the sum of that feature's picked amounts                                                                                               |

**Size behaviour, worth a run of its own.** Past 5 features or 24 options
the studio opens with every settled feature folded to one line (name,
rule, picked names, subtotal) and only undecided features open; header
click, rail click or _Expand all_ opens the rest, and an undecided feature
cannot be folded. Past 40 options the rows are compact. Search matches
option names, product codes, descriptions and feature names and hides
features with nothing to show. Fixture in cpq-dev: bundle **Studio Scale
Test Suite** `01tQL00000UQW1NYAX` (11 features, 41 options, 3
constraints; Storage deliberately has no default) committed at qty 25 on
quote `0Q0QL000005HUd70AG`. Remove its Storage pick to see the undecided
state. Both are throwaway.

**Which surface you are on, and what changes with it.** `Surface__c` on
the view decides where Configure Products opens the cart, and the studio
follows it (DDCPQ-SURFACE-01). Read it with `dd_cpq_get_cart_schema` —
the `surface` field — before filing anything about dialogs.

| `surface`        | Configure Products opens            | The studio then                                                                               |
| ---------------- | ----------------------------------- | --------------------------------------------------------------------------------------------- |
| `page` (default) | the **DD CPQ Cart** tab, full width | takes the cart's place: no scrim, no close box, a **Cart** crumb back, rail and summary stick |
| `popup`          | a dialog over the quote record      | opens as a second dialog over it, with a × and a click-away scrim                             |

On `page` a click beside the studio must NOT discard the picks, and browser
Back must return to the quote. On `popup` a scrim click closes the studio
and the cart dialog stays behind it. Filing "there are two popups" against
a `page` view means the view is really set to `popup`, or the tab did not
resolve — check before writing it up.

**What the tools cannot see:** the studio's client-side quantity
_estimate_ for unpicked options. It mirrors BundleQuantityResolver
(driver × multiplier, operator, min/max clamp, never below 1), and the
engine's answer replaces it the moment the option is picked. A `+$X`
delta that does not become the same amount once picked is a display
defect — Medium, and name both numbers.

---

## Family E · Variables

Variables are the customer extension point, so a broken one breaks a
customer's own configuration.

1. `dd_cpq_list_variables` — inventory. Note anything not `Active`:
   a Draft variable is **silently invisible** at runtime, which looks
   identical to a variable that fails to resolve.

   Two were seeded 2026-09-15, both `Active`: `cfg_seat_floor`
   (Constant, 25) and `cfg_account_industry` (Formula over
   `Account.Industry`). Both deliberately Active — seeding a Draft would
   have made this family untestable in a new way rather than testable.

2. `dd_cpq_get_variable` + `dd_cpq_test_variable` for each — resolve it
   in a real context and record the value, the resolver used, and the
   elapsed time.
3. `dd_cpq_variable_usages` — find where each is referenced. **A
   variable referenced by an option's `quantityVariableId` but failing
   to resolve is a live defect**: the option falls back to a default
   quantity and the quote looks fine.
4. For every variable that drives a quantity, close the loop: resolve
   it, configure the bundle, and assert the committed quantity equals
   the resolved value.

---

## Report format

Self-contained HTML, written with the file-write tool. Never open a
desktop app.

Required sections:

1. **Summary** — families run, scenarios, pass / fail / skipped,
   findings by severity.
2. **Findings triage** — one row per finding: id, severity, family,
   layer, expected, actual, and _which two layers disagreed_. That last
   column is what makes the report worth reading.
3. **Per-scenario detail** — the configuration sent, the engine's
   response, the cart's view.
4. **Constraint matrix** — every constraint × every combination tested,
   with Block / Warning / allowed and whether the engine agreed.
5. **Quantity rule matrix** — every option × every quantity tested.
6. **Coverage gaps** — combinations you did NOT test and why. Be
   explicit: an untested combination reported as passing is worse than
   an admitted gap.
7. **Run environment** — org, tool versions, quotes created and whether
   they were cleaned up.

### Severity

|              |                                                                                                |
| ------------ | ---------------------------------------------------------------------------------------------- |
| **Critical** | a Block that does not block — an invalid configuration can be sold                             |
| **High**     | a Warning that blocks; a rule that never fires; a quantity rule silently replaced by a default |
| **Medium**   | cart disagrees with the record beneath it; a rule that over-fires                              |
| **Low**      | message text, label, ordering                                                                  |
| **Info**     | seed hygiene, coverage gaps                                                                    |

## Degraded mode

| If this fails                                  | Do this                                                                                                                                                                                              |
| ---------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `list_rules` returns empty for everything      | Check you passed `bundleProductId` for `optionConstraints` / `siblingOptions` — without it the endpoint returns an empty list rather than an error, which looks exactly like "no constraints exist". |
| A tool name is not found                       | The tool surface was cleaned on 2026-09-14; 26 tools for cut features were retired. If a name you expect is gone, check the retired list before reporting it as a defect.                            |
| `configure_bundle` fails for an unclear reason | Read `get_bundle_structure` again — `minSelections` on a feature is enforced, and an omitted required option fails without naming itself.                                                            |
| OAuth expires mid-run                          | STOP. Ask the user to re-authenticate. Never silent-retry.                                                                                                                                           |
| You cannot delete a CFGTEST quote              | List it in the report footer under "cleanup owed".                                                                                                                                                   |
