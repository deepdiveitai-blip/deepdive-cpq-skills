# Compatibility and Option Constraints

> Installed build 0.1.0-9 · Verified 2026-10-07 in cpq-pkg (CP-prefixed data):
> Exclude Hard refuses commit — `"Cross-line exclusion violation: CP Trigger
Hard and CP Excl Hard Target cannot be quoted on the same quote."` · Exclude
> Warning commits (2 lines inserted) and the warning rides in the response ·
> Require Hard refuses until the companion is added, then commits · MaxQty 3
> refuses at quantity 5, commits at quantity 3 · a bundle's Option Constraint
> (Require/Exclude, Block/Warning) enforces the same way, one bundle down.

## What it is for

Two rule objects keep products that do not belong together apart, or force
them together, or cap how much of one a trigger allows:

- **Compatibility Rule** (`CompactabilityRule__c`, grouped in a
  **Compatibility Rule Set** = `CompactabilityMatrix__c`) — scoped to the
  **whole quote**: standalone lines, bundle components, and (for Exclude and
  MaxQty) what the account already owns. Authored in **Rules Studio**.
- **Option Constraint** (`OptionConstraint__c`) — scoped to **options inside
  one bundle**: "if the rep picks Edition X, they cannot also pick Add-on Y."
  Authored in **Bundle Builder**, not Rules Studio.

**The "Compactability" spelling is deliberate.** The object and field API
names are `CompactabilityMatrix__c` / `CompactabilityRule__c` — missing the
second "i" — per Data Model v1.2 §8.1. The object's own **label** reads
correctly ("Compatibility Rule Set" / "Compatibility Rule"); only the
underlying API name carries the typo. Do not "fix" it — renaming a
managed-package API name after registration breaks every installed org.

Use Compatibility Rules for "these two products, anywhere on the deal, must
never / must always / can only sell N of." Use Option Constraints for
"inside this one bundle, picking A blocks or demands B."

## How it works

**Pipeline.** Compatibility is step 8: it runs live on every selection change
in the configurator, and again at commit as a server-side backstop (so a
REST or MCP caller cannot bypass what the UI enforced). Option Constraints
run whenever a bundle is configured, and again at commit for the same
reason.

**Actions.**

| Object             | Actions                        | Notes                   |
| ------------------ | ------------------------------ | ----------------------- |
| Compatibility Rule | Exclude, Require, Max Quantity | All three.              |
| Option Constraint  | Require, Exclude               | No Max Quantity action. |

**Severity — two different picklists, same idea.**

| Object                           | Picklist values         | Blank means                                             |
| -------------------------------- | ----------------------- | ------------------------------------------------------- |
| Compatibility Rule `Severity__c` | **Hard** / **Warning**  | Hard                                                    |
| Option Constraint `Severity__c`  | **Block** / **Warning** | the UI always sets one (defaults to Block on a new row) |

Hard / Block refuses the commit outright. Warning lets the commit proceed
and surfaces the message. A rule authored with no severity behaves as the
stricter option, so a typo never silently under-enforces.

**Collision resolution.** When two rules fire on the same affected
product/option, the stricter wins — never a random pick:

- Compatibility Rule: **Exclude > Require > MaxQty**.
- Option Constraint: **Exclude+Block > Exclude+Warning > Require+Block >
  Require+Warning**, tiebroken deterministically by the trigger option Id.

**Cross-line enforcement — the whole quote, not just the bundle.** Every
Compatibility Rule is checked at commit against every product on the quote:
standalone lines, every bundle's components, and (for Exclude and MaxQty)
Assets the account already owns from a prior sale. There is no scope flag to
turn this off — it is unconditional. Two components that sit inside the
_same_ bundle line are left to that bundle's own Option Constraints instead
(an Exclude or an unmet Require between true siblings is skipped here, so
the two objects never double-report the same pair).

**Option Constraint's own scope flag — `EvaluationScope__c`.** Unlike
Compatibility Rule, Option Constraint has a scope picklist: **Bundle Only**
(default), **Quote Only**, **Cross-Line**. But **Bundle Builder's UI never
exposes this field** — every constraint a tester authors by clicking is
Bundle Only. In practice that still enforces whenever the two options are
actually selected together inside that bundle (verified below); Cross-Line
additionally catches the trigger product added as a _plain standalone line_
with no bundle wrapper, or already on the account as an Asset. Reaching that
wider scope today means editing `EvaluationScope__c` outside Bundle Builder.

**Live preview vs. commit.** `/dd/v1/price` (and the cart's live pricing)
never refuses on Exclude, Require or MaxQty — pricing proceeds regardless.
Only **commit** (`/dd/v1/commit`) blocks. The configurator's own live
compatibility check, `/dd/v1/compatibility`, answers "what should I grey out
or auto-add right now" — its response carries `action`, `affectedProductId`
and `message`, but **not severity**; the cart decides Hard-vs-Warning styling
some other way, and the authoritative Hard/Warning call is only made at
commit.

**Message precedence.** A rule's own `Message__c` wins when set. Blank falls
back to a generated sentence (exact wording below, verified).

**Criteria gate.** Both rule objects carry the optional shared 5-field
criteria schema (`FieldApiName__c` / `Operator__c` / `Value__c` /
`ValueType__c`, plus `Priority__c`) — blank criteria is a catch-all that
always fires; criteria that cannot be judged (no quote/account context)
**fail closed** — the rule does not fire, rather than firing on everything
the way a pre-2026-08-14 bug once did.

**Option Constraint's tier-floor mode.** A constraint with `FieldApiName__c`
set (instead of a `TargetOption__c`) becomes a rank-floor check — "this
option needs at least Professional edition selected in the bundle" — instead
of naming one exact option. **No UI authors this** — Bundle Builder's Rules
section only ever writes `TriggerOption__c` / `Action__c` / `TargetOption__c`
/ `Severity__c` / `Message__c`. It exists and the engine evaluates it, but
reaching it means editing the record directly.

## Objects and fields

| Object (API name, unprefixed)                      | Field                                                                           | Meaning / values                                                                                                                                                                                                               |
| -------------------------------------------------- | ------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| Compatibility Rule Set (`CompactabilityMatrix__c`) | `Status__c`                                                                     | Draft / **Active** / Inactive. Only Active is read when quotes price.                                                                                                                                                          |
| Compatibility Rule (`CompactabilityRule__c`)       | `Matrix__c`                                                                     | Lookup to its Rule Set.                                                                                                                                                                                                        |
|                                                    | `TriggerProduct__c`                                                             | The product that, when on the quote, fires this rule. Blank = any product (a catch-all trigger).                                                                                                                               |
|                                                    | `AffectedProduct__c`                                                            | The single product this rule excludes/requires/limits.                                                                                                                                                                         |
|                                                    | `Target__c` / `AffectedTarget__c`                                               | Written by Rules Studio's product picker when the trigger or affected side is "every product," a list, a criteria match, or a Product Group, instead of one named product. JSON; blank means the single-product field decides. |
|                                                    | `Action__c`                                                                     | **Exclude** / **Require** / **MaxQty**.                                                                                                                                                                                        |
|                                                    | `Severity__c`                                                                   | **Hard** (blocks commit) / **Warning** (surfaces, commit proceeds). Blank = Hard.                                                                                                                                              |
|                                                    | `MaxQuantity__c`                                                                | MaxQty only — the combined quantity ceiling on the affected side.                                                                                                                                                              |
|                                                    | `MaxQtyVariable__c`                                                             | MaxQty only — a Variable that supplies the limit instead of a static number; wins over `MaxQuantity__c` when it resolves.                                                                                                      |
|                                                    | `Message__c`                                                                    | Shown to the rep. Blank falls back to a generated sentence.                                                                                                                                                                    |
|                                                    | `FieldApiName__c` / `Operator__c` / `Value__c` / `ValueType__c` / `Priority__c` | The shared 5-field criteria gate. Blank = catch-all.                                                                                                                                                                           |
| Option Constraint (`OptionConstraint__c`)          | `TriggerOption__c`                                                              | Lookup to the `ProductOption__c` that fires this rule — scoped to one bundle by that option's parent product.                                                                                                                  |
|                                                    | `TargetOption__c`                                                               | The `ProductOption__c` this rule requires or excludes.                                                                                                                                                                         |
|                                                    | `Action__c`                                                                     | **Require** / **Exclude** (no MaxQty).                                                                                                                                                                                         |
|                                                    | `Severity__c`                                                                   | **Block** (stops the rep) / **Warning** (shown, rep continues).                                                                                                                                                                |
|                                                    | `EvaluationScope__c`                                                            | **Bundle Only** (default; Bundle Builder's UI always writes this) / Quote Only / Cross-Line.                                                                                                                                   |
|                                                    | `Message__c`                                                                    | Shown to the rep. Blank falls back to a generated sentence.                                                                                                                                                                    |
|                                                    | `FieldApiName__c` / `Operator__c` / `Value__c` / `ValueType__c`                 | Tier-floor mode (not UI-authored) — see above.                                                                                                                                                                                 |
|                                                    | `Priority__c`                                                                   | Evaluation order when more than one constraint could fire.                                                                                                                                                                     |

## Build it (tester)

This builds on the golden path's CRM Suite Pro bundle (Stage 5) and its six
products (Stage 4). It adds four new standalone products for cross-line
Compatibility, and two Option Constraints inside the existing bundle.

**1. New standalone products** (Products tab → New → Save, then Related →
Price Books → Add Standard Price → **100** → Active → Save, for each):

| Product           | Role                      |
| ----------------- | ------------------------- |
| Legacy Firewall   | Exclude trigger (Hard)    |
| Cloud Firewall    | Exclude target (Hard)     |
| Trial Seats       | Exclude trigger (Warning) |
| Enterprise Seats  | Exclude target (Warning)  |
| API Gateway       | Require trigger           |
| Rate Limiter Pack | Require target            |
| Burst Capacity    | MaxQty trigger            |
| Overage Credits   | MaxQty target             |

**2. A rate plan for each** (Rate Plan Editor tab, same as golden path Stage
6): **Recurring**, **Per unit**, **Monthly**, price **100**, **Active**. A
priced product with no Active plan cannot commit — the error below shows
exactly that message if you skip one.

**3. Rules Studio tab → Compatibility → New rule set** → name it
`Connector Compatibility` → it saves as **Draft**. Add four rules, each
**For** the trigger, **When** nothing (no criteria — catch-all), **Then**:

| Rule | Trigger         | Affected          | Action  | Severity | Extra              |
| ---- | --------------- | ----------------- | ------- | -------- | ------------------ |
| 1    | Legacy Firewall | Cloud Firewall    | Exclude | Hard     | —                  |
| 2    | Trial Seats     | Enterprise Seats  | Exclude | Warning  | —                  |
| 3    | API Gateway     | Rate Limiter Pack | Require | Hard     | —                  |
| 4    | Burst Capacity  | Overage Credits   | MaxQty  | Hard     | Max Quantity **3** |

Switch the rule set to **Active** — Rules Studio saves new sets as Draft and
the engine ignores a Draft set entirely.

**4. Bundle Builder tab → open CRM Suite Pro → Rules section → + Add rule**,
twice:

| When this option is selected | Rule     | This option     | Severity      | Message                                                   |
| ---------------------------- | -------- | --------------- | ------------- | --------------------------------------------------------- |
| Storage +100GB               | requires | Premium Support | Block the rep | "Extra storage needs Premium Support on the quote."       |
| Service Cloud                | excludes | Storage +100GB  | Warn only     | "Service Cloud with extra storage is an unusual pairing." |

Save the bundle (the setup rail should stay green — Warning-severity rules
never block Bundle Builder's own save).

**5. On quote Q-0042 Acme (or a fresh quote), try each pairing** in
Configure Products:

- Add Legacy Firewall + Cloud Firewall, qty 1 each → commit → refused.
- Remove Cloud Firewall, add Trial Seats + Enterprise Seats → commit →
  succeeds, with a warning shown.
- Add API Gateway alone → commit → refused; add Rate Limiter Pack too →
  commit → succeeds.
- Add Burst Capacity qty 1 + Overage Credits qty 5 → commit → refused; drop
  Overage Credits to qty 3 → commit → succeeds.
- In the bundle, pick Storage +100GB without Premium Support → blocked in
  Bundle Builder's own checklist; pick both → clear. Pick Service Cloud +
  Storage +100GB together → a warning, not a block.

## Check it (Claude)

```bash
sf data query -o dd-e2e -q "SELECT Name, DDCPQ__TriggerProduct__r.Name, DDCPQ__AffectedProduct__r.Name, DDCPQ__Action__c, DDCPQ__Severity__c, DDCPQ__MaxQuantity__c, DDCPQ__Matrix__r.DDCPQ__Status__c FROM DDCPQ__CompactabilityRule__c WHERE DDCPQ__Matrix__r.Name = 'Connector Compatibility'"
```

Expect 4 rows: Exclude/Hard, Exclude/Warning, Require/Hard, MaxQty/Hard
(MaxQuantity 3), all under a Rule Set reading **Active**.

```bash
sf data query -o dd-e2e -q "SELECT Name, DDCPQ__TriggerOption__r.DDCPQ__ChildProduct__r.Name, DDCPQ__TargetOption__r.DDCPQ__ChildProduct__r.Name, DDCPQ__Action__c, DDCPQ__Severity__c, DDCPQ__EvaluationScope__c FROM DDCPQ__OptionConstraint__c WHERE DDCPQ__TriggerOption__r.DDCPQ__ParentProduct__r.Name = 'CRM Suite Pro'"
```

Expect 2 rows (Storage→Premium Support Require/Block, Service Cloud→Storage
Exclude/Warning), both reading **Bundle Only** for Evaluation Scope — that
confirms Bundle Builder's UI never set anything else, even though the field
supports more.

After a refused commit, check nothing was written:

```bash
sf data query -o dd-e2e -q "SELECT Product2.Name FROM QuoteLineItem WHERE Quote.Name = 'Q-0042 Acme' AND Product2.Name IN ('Legacy Firewall','Cloud Firewall')"
```

A refusal must return **0 rows** — the engine checks before any line is
inserted, so a Hard violation never leaves a half-committed quote.

**Read-only live check** — the same pre-commit answer the configurator
polls, with no severity in the response (verified: the response carries only
`action` / `affectedProductId` / `message`):

```bash
sf api request rest "services/apexrest/DDCPQ/dd/v1/compatibility" -o dd-e2e --method POST --body @compat-check.json
```

```json
{ "selectedProductIds": ["<Legacy Firewall 01t…>"] }
```

Returns `{"actions":[{"affectedProductId":"<Cloud Firewall 01t…>","action":"Exclude","message":null,"maxQty":null}]}`
when `Message__c` is blank — Claude only reads this; never call `dd/v1/commit`.

## Expected numbers

All verified in cpq-pkg with equivalent CP-prefixed products (same structure,
different names — the engine's message templates are deterministic, so the
sentence shape is identical for the tester's own products).

| Scenario                           | Call                                                      | Result                                                                                                                                                                                                                              |
| ---------------------------------- | --------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Exclude, Hard                      | commit both products                                      | **Refused.** `errorType: "ConfigurationRejected"`, `code: "CFG_EXCLUSION"`, `error: "Cross-line exclusion violation: CP Trigger Hard and CP Excl Hard Target cannot be quoted on the same quote."`                                  |
| Exclude, Warning                   | commit both products                                      | **Committed**, `linesInserted: 2`. Response `warnings[0].message`: `"CP Trigger Warn is unusual with CP Excl Warn Target — proceeding anyway."` (the rule's own `Message__c`), `code: "PRODUCTS_EXCLUDED"`, `data.blocking: false`. |
| Require, unmet                     | commit trigger alone                                      | **Refused.** `errorType: "DDCPQ.CommitValidatorService.RequirementViolationException"`, `code: "CFG_REQUIREMENT"`, `error: "Missing required product: CP Req Trigger requires CP Req Target on the same quote."`                    |
| Require, met                       | commit trigger + companion                                | **Committed**, `linesInserted: 2`.                                                                                                                                                                                                  |
| MaxQty, over limit                 | trigger qty 1 + target qty 5, limit 3                     | **Refused.** `errorType: "DDCPQ.CommitValidatorService.MaxQuantityViolationException"`, `code: "CFG_MAX_QUANTITY"`, `error: "CP MaxQty Trigger allows at most 3 of CP MaxQty Target on the quote; there are 5."`                    |
| MaxQty, at limit                   | trigger qty 1 + target qty 3, limit 3                     | **Committed**, `linesInserted: 2`.                                                                                                                                                                                                  |
| Option Constraint, Require unmet   | bundle: trigger option alone                              | **Refused.** `errorType: "ConfigurationRejected"`, `code: "CFG_OPTION_CONSTRAINT"`, `error: "Option constraint violation: CP Opt Require Trigger needs CP Opt Require Target in this bundle."`                                      |
| Option Constraint, Exclude Block   | bundle: trigger + excluded option, Require side satisfied | **Refused.** Same shape, `error: "Option constraint violation: CP Opt Require Trigger and CP Opt Exclude Block Target cannot be picked together."`                                                                                  |
| Option Constraint, Exclude Warning | bundle: trigger + warned option, Require side satisfied   | **Committed**, `linesInserted: 4`. Top-level `warnings: []` — **empty**; the only trace is the Rules summary (`ruleType: "OptionConstraint__c"`, `evaluated: 1`).                                                                   |

## Troubleshooting

| Symptom                                                                       | Cause                                                                                                                                                                                    | Fix                                                                                                                                                                                                  |
| ----------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Commit refused: `"Cross-line exclusion violation: …"`                         | A Hard (or blank-severity) Exclude rule fires — the two products, or one plus an owned Asset, are both present.                                                                          | Remove one of the two named products, or change the rule's Severity to Warning if it should only advise.                                                                                             |
| Commit refused: `"Missing required product: … requires … on the same quote."` | A Hard Require rule's trigger is present but its companion is not.                                                                                                                       | Add the named companion product, or change the rule's Action if Require was a mistake.                                                                                                               |
| Commit refused: `"… allows at most N of … on the quote; there are M."`        | A MaxQty rule's combined affected quantity exceeds the limit.                                                                                                                            | Lower the quantity, raise `MaxQuantity__c`, or point `MaxQtyVariable__c` at a Variable that resolves higher.                                                                                         |
| Commit refused: `"Option constraint violation: …"` inside a bundle            | A Block-severity Require or Exclude on `OptionConstraint__c` is unmet, even though the configurator let the pick through client-side.                                                    | Check Bundle Builder's Rules section for the named pair; the server re-checks every Block constraint at commit regardless of what the UI showed.                                                     |
| A Warning rule still blocks the commit                                        | `Severity__c` is blank, not actually set to Warning. Blank behaves as Hard/Block on both objects.                                                                                        | Open the rule and explicitly set Severity to Warning (Compatibility Rule) or Warn only (Option Constraint).                                                                                          |
| An Option Constraint never fires outside the bundle configurator              | `EvaluationScope__c` is Bundle Only (the default, and the only value Bundle Builder's UI writes) — it only fires when the two options are actually selected together inside that bundle. | If the constraint must also catch the trigger product added as a standalone line or already owned, the field needs to be set to Quote Only or Cross-Line directly on the record; no UI exposes this. |
| Rule set shows the right rules but nothing fires                              | `CompactabilityMatrix__c.Status__c` is still **Draft**. Rules Studio always saves a new set as Draft.                                                                                    | Switch the rule set to Active.                                                                                                                                                                       |
| Two rules on the same pair disagree and you're not sure which one "won"       | Collision resolution is automatic and silent: Exclude beats Require beats MaxQty (Compatibility), Exclude+Block beats everything (Option Constraint).                                    | Check the Rules trace in the commit response (`rules.lines[]`), or query both rules and apply the precedence table above by hand.                                                                    |

## Limits and gotchas

- **Exclude/Require/MaxQty never block `/dd/v1/price`** — only `/dd/v1/commit`
  does. A quote can be priced and shown to a rep in a state that will be
  refused at commit; don't read a clean price preview as "this will commit."
- **The error shape is inconsistent across the four refusal paths.** Exclude
  and Option Constraint violations both report `errorType: "ConfigurationRejected"`;
  Require and MaxQty leak the raw Apex exception class name
  (`DDCPQ.CommitValidatorService.RequirementViolationException` /
  `...MaxQuantityViolationException`). All four still carry a distinct `code`
  (`CFG_EXCLUSION` / `CFG_REQUIREMENT` / `CFG_MAX_QUANTITY` /
  `CFG_OPTION_CONSTRAINT`), so branch on `code`, not `errorType`.
- **A Warning-severity Option Constraint produces no entry in the commit
  response's `warnings[]` array** — verified empty. Contrast with a
  Warning-severity Compatibility Rule, which does populate `warnings[]` with
  the rule's message. The only trace of an Option Constraint Warning firing
  is the Rules summary (`rules.rules[].ruleType == "OptionConstraint__c"`).
  A rep relying on the cart to show them something will see nothing for this
  case today.
- **Bundle Builder's Rules section cannot author `EvaluationScope__c` or the
  tier-floor fields** (`FieldApiName__c` / `Operator__c` / `Value__c` /
  `ValueType__c` on Option Constraint). Both exist and the engine honors
  them; neither is reachable by clicking.
- **Max Quantity has no Option Constraint equivalent.** A per-bundle quantity
  cap across two options needs a Compatibility Rule (which can reach into a
  bundle's components) rather than anything in Bundle Builder's Rules
  section.
- Governor limits: both enforcement passes are bulk-safe (one SOQL for the
  rule rows, one for names), but a quote with hundreds of lines and dozens of
  Active rules still means dozens of rule rows evaluated per commit — normal
  CPQ scale, not a concern at golden-path size.

## Questions testers ask

**Why are there two different objects for basically the same idea?**
Scope. Compatibility Rules see the whole quote and owned Assets; Option
Constraints see inside one bundle. A bundle's own Require/Exclude belongs in
Bundle Builder next to the options it governs; a rule about two products
that might never even be in the same bundle belongs in Rules Studio.

**I set Severity to Warning but the quote still won't save — why?**
Check which object. A blank `Severity__c` on either object behaves as the
stricter value (Hard / Block), not as "no severity." Open the rule and
confirm the picklist actually reads Warning / Warn only.

**Does the rep see a Warning before they commit, or only after?**
The live `/dd/v1/compatibility` check (polled on every selection) tells the
UI to grey out or auto-add, but it does not carry severity. The authoritative
Hard-vs-Warning verdict, and the message text, is decided at commit.

**Can one rule's Trigger be "any product in this family" instead of one
specific product?**
Yes, on Compatibility Rule, via `Target__c` / `AffectedTarget__c` — a Product
Group, a list, or a criteria match instead of one product. Option
Constraint's trigger/target are always one exact option.

**If I exclude A and B, and the customer already owns B from a prior order,
does adding A to a new quote get blocked?**
Yes, for Compatibility Rule's Exclude and MaxQty — owned Assets count as
present. Require ignores what's owned unless the companion is also on the
new quote or already owned (same check, same Scope).

**Why did my Option Constraint's Warning never show up anywhere?**
See Limits and gotchas — Option Constraint Warnings are silent in the
commit response today; only the Rules trace records them.

**Do I need an Active rate plan on the excluded product to test this?**
Yes — commit refuses first on whichever check fails first in the pipeline.
A product with no Active plan fails before Compatibility ever gets a
chance to fire, and the error is a different one (missing rate plan, not
an exclusion).
