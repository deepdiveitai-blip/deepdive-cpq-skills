# DD CPQ Settings, Engine Health, Product 360, catalogs, audit

> Installed build 0.1.0-9 · Verified 2026-10-07 in cpq-pkg: `DD CPQ Settings`
> POST with an empty body returns all 4 setting groups with live values (no
> `Setting__c` rows exist — every value is still the shipped default);
> `telemetry.last` on the golden quote returns a real `RUN-0000002` with 29
> pipeline stages, `worstRatio` 0.28 of the SOQL budget; a product with no
> standard price and no rate plan comes back from Product 360 with health
> codes `NO_ACTIVE_PLAN`, `NO_STANDARD_PBE` and `NOT_IN_ANY_CATALOG`.

## What it is for

Four admin surfaces that sit next to the selling tabs:

- **DD CPQ Settings** — the one place that turns package behaviour on/off:
  quote expiry, the cart header, how contracts are created, and what the
  Engine Health tab keeps.
- **Engine Health** — what every pricing/commit run cost against Salesforce's
  own limits, which rules fired, and every error a user hit, each with a
  short reference code.
- **Product 360** — one screen with everything DD CPQ knows about a single
  product: is it sellable, what's wrong with it, what sells it, who prices
  it, and a one-click fix for the easy problems.
- **Catalogs** — a curated, orderable slice of the product list, independent
  of pricing and independent of who may buy what (Eligibility Matrix).

None of these touch the cart's pricing math — they are read surfaces (plus
the Settings writes) that watch and configure the engine, not pipeline
stages themselves.

## How it works

### Settings — `CpqSettings` is the one source of truth

`DD CPQ Settings` is a Lightning tab, backed by one Apex class
(`CpqSettings`) that is the single registry of every setting that exists —
its key, group, label, help text, type, allowed values, default and range.
A `Setting__c` row only exists once an admin actually changes something; a
brand-new org has **zero** `Setting__c` rows and every setting reads its
shipped default. This is why the golden path's org, never touched, still
behaves correctly — nothing needs to be seeded.

Four groups, in this fixed order:

| Group                     | What it governs                                                                                                                                                                                    |
| ------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Quote Expiration Date** | Whether new quotes get an expiry date, who can edit it, the default validity window, what "expired" does to Commit, and the nightly Draft→Expired sweep                                            |
| **Cart Header**           | How many header-field lines show before a Collapse button appears, and whether reps can add their own fields from a pool                                                                           |
| **Contracts**             | How accepted quotes become Contracts, whether DD CPQ activates them itself, and whether a nightly job drafts renewal quotes ahead of a contract's end                                              |
| **Engine Health**         | Whether the cart's Run meter / Rules meter show (and to whom), whether run history and error logs are kept at all, the sample rate for healthy runs, and how long each kind of history is retained |

Every setting is one of four **types**: `Boolean` (true/false), `Number`
(has a `min`/`max`/`unit`), `Picklist` (has a fixed `options` list), or
`Info` (a read-only note with a link — e.g. "Fields users can add" points at
Setup's field set editor; it carries no value of its own).

A value is read fresh once per transaction — there is no per-user override,
so this is **package-wide configuration**, not a personal preference. Saving
requires the `Administer Pricing` permission (part of
`DDCPQ__DD_CPQ_Engine_Admin`); every save is validated key-by-key (unknown
keys and `Info` keys are refused, numbers must be in range, booleans must be
`true`/`false`, picklist values must be one of the listed options) and
written **all-or-nothing** — one bad value in the batch rejects the whole
save with every problem listed, not just the first.

**Saving some settings starts something immediately.** Turning
`Renewal_Draft_Days` above 0, `Expire_Drafts_Nightly` on, or either
`Engine_Telemetry_Persist`/`Error_Log_Enabled` on schedules a nightly
Apex job the first time (see Housekeeping below) — it is a one-time
`System.schedule`, found-or-created, never duplicated.

Every save is itself audited: `AuditService.log('SettingsChanged', …)`
records the full before/after map of every key that changed, visible
through `AuditLog__c` (see Audit below).

### Settings reference — every key in this build

| Key                               | Group                 | Type     | Default           | Options / range                                                       | What it changes                                                                                                                  |
| --------------------------------- | --------------------- | -------- | ----------------- | --------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------- |
| `Quote_Expiration_Date`           | Quote Expiration Date | Boolean  | `true`            | —                                                                     | New quotes get an expiry date automatically                                                                                      |
| `Quote_Expiry_Rep_Editable`       | Quote Expiration Date | Boolean  | `true`            | —                                                                     | A rep's typed date is never overwritten by automation                                                                            |
| `Quote_Validity_Days`             | Quote Expiration Date | Number   | `30`              | 1–365 days                                                            | How many days out the default expiry is set                                                                                      |
| `Quote_Expiry_Basis`              | Quote Expiration Date | Picklist | `CreatedDate`     | `CreatedDate`, `LastCommit`                                           | Whether the date is fixed at creation or moves forward on every commit                                                           |
| `Expired_Quote_Enforcement`       | Quote Expiration Date | Picklist | `Warn`            | `Off`, `Warn`, `Block`                                                | Off = nothing; Warn = banner offering to reprice; Block = Commit refused until repriced                                          |
| `Expire_Drafts_Nightly`           | Quote Expiration Date | Boolean  | `false`           | —                                                                     | Nightly job moves expired Draft quotes to Expired (repricing brings them back)                                                   |
| `Quote_Expiry_Label`              | Quote Expiration Date | Picklist | `Expiration Date` | `Expiration Date`, `Quote valid until`                                | What the date is called in the header/banner                                                                                     |
| `Header_Max_Lines`                | Cart Header           | Number   | `2`               | 1–20 lines                                                            | Lines of header fields shown before a Collapse button appears                                                                    |
| `Allow_Header_Field_Selection`    | Cart Header           | Boolean  | `true`            | —                                                                     | Shows "Choose fields" so each user can add from the pool                                                                         |
| `Header_Available_Pool`           | Cart Header           | Info     | —                 | —                                                                     | Note only — the pool is field set `DD CPQ Header Available`, the default header is `DD CPQ Header Default`, both edited in Setup |
| `Contracting_Method`              | Contracts             | Picklist | `PerQuote`        | `PerQuote`, `ByEndDate`, `SingleContract`, `ContractPerWindowedBlock` | How accepted quotes turn into Contracts (see contracts-and-acceptance.md)                                                        |
| `Contract_Set_Status`             | Contracts             | Boolean  | `false`           | —                                                                     | Whether DD CPQ sets a created Contract to Activated itself                                                                       |
| `Renewal_Draft_Days`              | Contracts             | Number   | `0`               | 0–365 days                                                            | Nightly job drafts a renewal quote this many days before a contract ends; 0 = off                                                |
| `Engine_Telemetry_Panel`          | Engine Health         | Picklist | `Off`             | `Off`, `Admins`, `Everyone`                                           | Who sees the cart's Run meter                                                                                                    |
| `Rules_Meter_Panel`               | Engine Health         | Picklist | `Off`             | `Off`, `Admins`, `Everyone`                                           | Who sees the cart's Rules meter                                                                                                  |
| `Engine_Telemetry_Persist`        | Engine Health         | Boolean  | `false`           | —                                                                     | Keeps `EngineRun__c` history at all (failed/near-limit always kept regardless once this is on; see sample rate)                  |
| `Engine_Telemetry_Sample_Pct`     | Engine Health         | Number   | `10`              | 0–100 %                                                               | Share of _healthy_ runs kept as a baseline; problem runs are always kept                                                         |
| `Engine_Telemetry_Retention_Days` | Engine Health         | Number   | `30`              | 1–365 days                                                            | How long `EngineRun__c` rows are kept before nightly purge                                                                       |
| `Error_Log_Enabled`               | Engine Health         | Boolean  | `false`           | —                                                                     | Keeps the full detail (quote, line, stage, rule, limits, stack trace) behind every error reference                               |
| `Error_Log_Retention_Days`        | Engine Health         | Number   | `90`              | 1–730 days                                                            | How long `ErrorLog__c` rows are kept                                                                                             |
| `Rule_Stats_Retention_Days`       | Engine Health         | Number   | `400`             | 7–1000 days                                                           | How long daily per-rule counts (`RuleStat__c`) are kept; needs "Keep a history of runs" on to be populated                       |

A setting outside its range, or a bad picklist value, silently **reads back
as its default** (`CpqSettings.getInteger`/`getPicklist`) rather than
erroring — the save-time validation is the only gate; a record hand-edited
in Setup to an invalid value is simply ignored thereafter.

### Engine Health — what a pricing run cost, and what went wrong

Every price/commit/load/reprice/rule-test run is measured against the
platform's own limits (SOQL queries, query rows, DML statements, DML rows,
CPU ms, heap bytes, callouts) and against **DD CPQ's own, tighter, budgets**
(`DdCpqLimits`: a cart-preview SOQL budget, a preview DML budget, and
`maxLinesPerRequest` — the same number as `EngineLimits__mdt`). A run is
only **kept** as a history row (`EngineRun__c`) when `Engine_Telemetry_Persist`
is on, and then only if it failed, came within the warn/critical thresholds
of a platform limit, went over one of DD CPQ's own budgets, or was picked
up by the random healthy-run sample (`Engine_Telemetry_Sample_Pct`). A
failed run is always kept even though its own database changes rolled
back — that is the point: you can see exactly what blew up.

Two fixed thresholds colour every meter, everywhere a ratio to a limit is
shown: **60%** of a limit is "warn" (amber), **85%** is "critical" (red).
These are code constants (`EngineHealthService.WARN_AT`/`CRITICAL_AT`), not
settings — they do not move.

**The cart's Run meter** (a collapsible panel at the foot of the cart, if
`Engine_Telemetry_Panel` is `Admins` or `Everyone`) shows the _same_ data for
the quote's last run: per-limit progress bars with the 60/85 colouring, a
budget marker where DD CPQ's own budget sits inside the platform limit, a
"Failed"/"Over budget" flag, a per-pipeline-stage breakdown (queries, rows,
writes, CPU, heap), and the run's trace id. **The Rules meter** (its own
panel, `Rules_Meter_Panel`) lists, per line, every rule the run **applied**,
**blocked**, **warned** about, or evaluated with **no match** — with a link
to open the rule and (where testable) "Test on this quote".

**The Engine Health tab** is the admin's wider view of the same data, with
a **Classic**/**Lab report** toggle (saved per user) and, in Classic, four
tabs: **Runs**, **Errors**, **Rules**, **Quote**.

- **Runs** — every kept run in a period (Last day / Last 7 days / Last 30
  days), filterable by run kind (**All runs**, **Price previews**,
  **Commits**, **Cart loads**, **Reprices**, **Rule tests**). Columns: When,
  Run, Quote, Lines, Queries, CPU ms, Worst (the highest ratio to any
  limit), Heaviest stage, Kept because. Clicking a row opens the same Run
  meter the cart shows.
- **Errors** — every kept `ErrorLog__c` row, filterable by code, severity,
  stage, operation, quote, user, and a day window; grouped by root-cause
  fingerprint so one bug that fired 50 times reads as one group with a
  count, not 50 rows.
- **Rules** — which rules matter (`RuleInsightService`): rules that never
  fire, rules that block or warn a lot, rules that are slow.
- **Quote** — one quote's own runs, its chosen run's full pipeline flow, and
  its errors (`DiagnosticsService.quote`) — the same packet
  `diagnostics.packet`/`dd_cpq_issue_packet` builds for filing a Jira issue.

**Every error reference is reproducible from its trace id.** A run and its
errors share one `TraceId__c` (a UUID). The six-character code after `DD-`
(e.g. `DD-7F3K2Q`) is not random or sequential — it is the first 32 bits of
that UUID, shifted right 2 bits (30 bits) and base-32 encoded in a
Crockford-style alphabet with no `I`/`L`/`O`/`U` (so it can be read aloud or
typed without the classic 1/I, 0/O mixups). The same trace id always
produces the same reference, which is why "sameRequest" and "sameRootCause"
lookups on one reference work. `ErrorLog__c.Fingerprint__c` is a different
hash — of the error's code, stage and message with every id and number
stripped out — so the _same bug_ recurring on different quotes/lines groups
together even though each occurrence gets its own reference.

A code or trace id that is not found reads: _"No error is logged under
{key}. It may be older than the retention period, or the error log may be
off (Settings: Engine Health)."_ — the first thing to check is whether
`Error_Log_Enabled` was even on when the error happened.

### Catalogs — a curated slice, independent of price and eligibility

A **Catalog** (`Catalog__c`) is a named, orderable subset of the product
list — "Microsoft Worldwide", "EMEA Public Sector" — with nothing to do with
price (that is the Pricebook's job) and nothing to do with _who may buy_
(that is the Eligibility Matrix's job). Think of it as a third, independent
filter a rep's product browser applies, in a fixed order that never
changes:

```
Pricebook + currency filter  →  Eligibility Matrix  →  Catalog selection
```

A product dropped by the pricebook/currency filter never reaches the
eligibility check; a product the Eligibility Matrix hides never reaches
catalog narrowing. A product can belong to zero, one, or several catalogs.

Only an **Active** `Catalog__c` is ever offered to a rep or returned by the
catalog list REST action — Draft and Inactive catalogs exist but are
invisible everywhere except direct record access. `CatalogProduct__c` is
the junction (master-detail to `Catalog__c`): one row per product per
catalog, with a display order and an optional "Featured" flag the tile UI
highlights.

**There is no dedicated Catalog setup screen.** An admin manages both
objects with plain Salesforce record pages — the **Catalogs** tab (in the
DeepDive CPQ app) lists `Catalog__c` records; each Catalog record's related
list is where its `CatalogProduct__c` rows are added. Nothing here is a
custom LWC wizard.

### Housekeeping — the nightly jobs behind the numbers above

Four `Schedulable` classes exist, each started the first time its related
setting is saved into the state that needs it — **none of them pre-exist in
a fresh org**, so a tester who never touches these settings will find zero
scheduled jobs, forever:

| Job (`CronJobDetail.Name`)                            | Started by                                                      | What it does                                                                                                                                                           |
| ----------------------------------------------------- | --------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `DD CPQ - purge engine history`                       | `Engine_Telemetry_Persist` **or** `Error_Log_Enabled` turned on | Runs `DdCpqHousekeepingBatch` nightly (02:30), chained: deletes `EngineRun__c` past its retention, then `ErrorLog__c`, then `RuleStat__c`, one batch job after another |
| (quote expiration job — `QuoteExpirationSchedulable`) | `Expire_Drafts_Nightly` turned on                               | Moves expired Draft quotes to Expired (see Settings table)                                                                                                             |
| (renewal draft job — `RenewalDraftSchedulable`)       | `Renewal_Draft_Days` set above 0                                | Drafts a renewal quote ahead of a contract's end date (contracts-and-acceptance.md)                                                                                    |
| (asset projection — `AssetProjectionSchedulable`)     | not wired to a Setting in this build                            | out of scope here; see amend-renew.md                                                                                                                                  |

Each `ensureScheduled()` looks for its own job by name first — calling it
twice, or saving the triggering setting twice, never creates a duplicate
cron trigger. Because `Type.forName(...)` resolves the Apex class by
string, `CpqSettings.save` can trigger all four jobs without a hard
compile-time dependency on any of them.

### Product 360 — everything DD CPQ knows about one product

In the **DeepDive CPQ** app, opening **any** Product2 record lands on
Product 360 directly — the app overrides the standard Product `View`
action, so there is no separate click to find it. There is also a
standalone **Product 360** tab with its own product search, for when
nothing is open yet. (Outside the DeepDive CPQ app, a Product record looks
like plain Salesforce — the override is app-scoped.)

Five tabs, exact on-screen labels: **Overview**, **Configuration**,
**Pricing**, **Rules**, **Quotes**. ("Configuration" is the UI label for
what the code internally calls `structure` — don't be thrown by that if you
read the Apex/JS.)

Every finding is a `code` + severity (`crit`/`warn`/`info`, plus `ok` for
the all-clear) + a human `message`, and most carry a `fixSurface`/
`fixLabel` so Overview can offer a button rather than sending the admin
hunting. The installed build emits 18 distinct codes — more than
`docs/PRODUCT_360_GUIDE.md` documents (that guide's table lists only 12;
six real codes are missing from it — noted in Limits and gotchas):

| Finding code              | Severity                                    | Message (verbatim)                                                                                                                                              | Fix button                                       |
| ------------------------- | ------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------ |
| `PRODUCT_INACTIVE`        | warn                                        | "The product is inactive, so reps cannot add it."                                                                                                               | **Make it active**                               |
| `NO_ACTIVE_PLAN`          | crit (Block) / warn (Warn)                  | the coverage reason, plus " — the engine refuses it on a quote." when Block                                                                                     | **Add a rate plan**                              |
| `DRAFT_PLAN_ONLY`         | warn                                        | "Its only plans are Draft, and the engine rates from Active plans only."                                                                                        | **Activate**                                     |
| `OPTION_NO_ROLE`          | warn (on the option) / info (on the bundle) | "It is an option in a bundle with no pricing role... Decide whether it is charged or included." / "{N} option(s) have no pricing role and count as charged."    | **Open the bundle** / **Open in Bundle Builder** |
| `NO_STANDARD_PBE`         | crit                                        | "It has no standard price book entry, so it cannot be added to any price book or quote."                                                                        | **Add a standard price**                         |
| `PBE_INACTIVE`            | warn                                        | "Its standard price book entry is inactive."                                                                                                                    | **Activate the entry**                           |
| `OPTION_CHILD_INACTIVE`   | warn                                        | "{N} option(s) point at an inactive product, which a rep cannot select."                                                                                        | **Review options**                               |
| `OPTION_CHILD_NO_PLAN`    | crit (Block) / warn                         | "{N} charged option(s) have no Active rate plan, so the bundle cannot be quoted with it/them."                                                                  | **Review options**                               |
| `BUNDLE_EMPTY`            | warn                                        | "It has features but no options, so a rep has nothing to choose."                                                                                               | **Open in Bundle Builder**                       |
| `NO_BUNDLE_DEFINITION`    | crit                                        | "It has features and options but no bundle definition, so nothing says how the bundle is priced."                                                               | **Open in Bundle Builder**                       |
| `BUNDLE_DEF_INACTIVE`     | warn                                        | "Its bundle definition is {status}, not Active."                                                                                                                | **Activate**                                     |
| `FEATURE_UNSATISFIABLE`   | crit                                        | "{feature(s)} ask(s) for more selections than it has options, so no rep can complete the bundle."                                                               | **Review features**                              |
| `PLAN_GROUP_KEY_NULL`     | warn                                        | "{N} of {activePlans} Active plan(s) have no plan group key. Saving the plan in the Rate Plan Editor stamps it from its revenue nature."                        | **Open rate plans**                              |
| `NO_DEFAULT_IN_GROUP`     | warn                                        | "No Active recurring plan is marked default, so the headline price is whichever plan loads first."                                                              | **Set a default**                                |
| `FLOOR_NO_COST`           | warn                                        | "An Active margin floor applies to this product, but {N} of its {activePlans} Active rate plan(s) have no unit cost, so the floor is never checked on it/them." | **Add a unit cost**                              |
| `RULE_IN_INACTIVE_MATRIX` | info                                        | "{N} rule(s) name it inside a matrix that is not Active, so it/they do not apply."                                                                              | **See rules**                                    |
| `NOT_IN_ANY_CATALOG`      | info                                        | "It is in no catalog, so reps browsing catalogs will not find it."                                                                                              | none                                             |
| `ALL_CLEAR`               | ok                                          | "Nothing stops this product being sold."                                                                                                                        | only shown when nothing else fired               |

**"Add a standard price" does not create the entry for you.** Clicking it
navigates to Salesforce's own standard Price Book Entries related list on
the Product record — it is a deep-link, not an inline create. (The
Pricing tab also does **not** show a price-book-entries grid at all in this
build, despite what the guide doc implies — see Limits and gotchas.)

**Sellability** rolls every finding into one verdict: `status` (Sellable /
Blocked / "Needs attention" / "Not rated on its own"), `bucket` (e.g.
`noPlan`, `draftOnly`, `decide`, `exempt`, `sellable`), the rate-plan gate's
current `enforcement` (Block/Warn/Off, from `DD_CPQ_Rate_Plan_Settings__mdt`)
and a `readyToSell` boolean — true only when **no** finding is `crit` or
`warn`. `Blocked` covers: inactive product, no standard PBE, or
(`bucket == noPlan` and enforcement is Block).

**Configuration tab** shows the bundle structure — features
(`ProductFeature__c`: display type, min/max selections) and their options
(`ProductOption__c`: pricing role, required, default-selected, discountable,
quantity rule) — plus a "Belongs to" list of every other bundle this product
is an option inside. **Pricing tab** shows a qty 1/10/100 "Price at a
glance" preview from the default Active plan, the full rate plan list (the
same data the Rate Plan Editor uses, embedded), and currency. Inline edits
on Configuration/Pricing go through four REST admin actions:
`pbe.save`, `feature.update`, `product.update`, `option.update` (all under
`/dd/v1/admin`) — `pbe.save` is admin-only (`Administer Pricing`).

## Objects and fields

| Object label (API name, unprefixed)                   | Field                                                                                         | Meaning / values                                                                                                                                             |
| ----------------------------------------------------- | --------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| Setting (`Setting__c`)                                | `Name` (standard)                                                                             | The setting's key, e.g. `Error_Log_Enabled`                                                                                                                  |
|                                                       | `Group__c`                                                                                    | The Settings-tab group label                                                                                                                                 |
|                                                       | `Value__c`                                                                                    | The saved value (text); absent = the shipped default is in force                                                                                             |
|                                                       | `ValueType__c`                                                                                | Boolean / Number / Picklist, copied from the registry at save time                                                                                           |
| Engine Run (`EngineRun__c`)                           | `TraceId__c`                                                                                  | Unique UUID; the Run meter's and every API response's id for this run                                                                                        |
|                                                       | `Reference__c`*                                                                               | not a field on this object — the `DD-XXXXXX` reference lives on `ErrorLog__c`; a run's "reference" in the API response is derived live from its `TraceId__c` |
|                                                       | `RunKind__c`                                                                                  | `preview` / `commit` / `load` / `reprice` / `ruleTest`                                                                                                       |
|                                                       | `Succeeded__c`                                                                                | False when the run raised an error (row is still kept)                                                                                                       |
|                                                       | `WorstRatio__c`                                                                               | Highest share (0–1) of any platform limit this run used                                                                                                      |
|                                                       | `PeakStage__c`                                                                                | The pipeline stage that used the most queries                                                                                                                |
|                                                       | `Reason__c`                                                                                   | Why it was kept: failed / near a limit / over budget / sampled                                                                                               |
|                                                       | `SoqlQueries__c`, `CpuMs__c`, `HeapBytes__c`, `DmlStatements__c`, `DmlRows__c`, `Callouts__c` | What the transaction used                                                                                                                                    |
|                                                       | `SoqlLimit__c`, `CpuLimit__c`, `HeapLimit__c`                                                 | The platform limit at run time                                                                                                                               |
|                                                       | `SoqlBudget__c`                                                                               | DD CPQ's own, tighter, self-imposed budget for this kind of run                                                                                              |
|                                                       | `RulesApplied__c`, `RulesBlocked__c`, `RulesJson__c`                                          | Rule outcomes — the Rules meter's data                                                                                                                       |
|                                                       | `StagesJson__c`                                                                               | Per-pipeline-stage limit use, as JSON                                                                                                                        |
|                                                       | `Quote__c`, `User__c`, `LineCount__c`                                                         | What was priced, by whom, how many lines                                                                                                                     |
| Error Log (`ErrorLog__c`)                             | `Reference__c`                                                                                | The `DD-XXXXXX` code the user was shown; unique, external id                                                                                                 |
|                                                       | `TraceId__c`                                                                                  | Links back to the `EngineRun__c` of the same request                                                                                                         |
|                                                       | `Fingerprint__c`                                                                              | Hash of code+stage+message with ids/numbers stripped — same root cause, same fingerprint                                                                     |
|                                                       | `Code__c`                                                                                     | DD CPQ error code, e.g. `CFG_OPTION_CONSTRAINT`, `FLOOR_BLOCKED`                                                                                             |
|                                                       | `Severity__c`                                                                                 | Info / Warning / Error / Fatal                                                                                                                               |
|                                                       | `Category__c`                                                                                 | Configuration / Data / Access / Limit / Platform / Unexpected                                                                                                |
|                                                       | `Handled__c`                                                                                  | True when DD CPQ carried on and said how, rather than stopping                                                                                               |
|                                                       | `Message__c`, `Detail__c`, `StackTrace__c`                                                    | What the user was told / the exception message / where in the code                                                                                           |
|                                                       | `Stage__c`, `Operation__c`, `Surface__c`                                                      | Pipeline stage; preview/commit/admin.*; LWC/REST/MCP/Apex/Batch                                                                                              |
|                                                       | `Quote__c`, `LocalKey__c`, `RuleId__c`, `RuleType__c`, `User__c`                              | What it was about, and who hit it                                                                                                                            |
| Rule Stat (`RuleStat__c`)                             | `StatKey__c`                                                                                  | `ruleId\|yyyy-mm-dd\|runKind`, unique                                                                                                                        |
|                                                       | `RuleId__c`, `RuleType__c`, `Day__c`, `RunKind__c`                                            | Which rule, which kind, which day                                                                                                                            |
|                                                       | `Evaluated__c`, `Matched__c`, `Applied__c`, `Blocked__c`, `Warned__c`, `Shadowed__c`          | Daily counters by outcome                                                                                                                                    |
| Audit Log (`AuditLog__c`)                             | `Action__c`                                                                                   | Picklist of business events — Settings Changed, Manual Price Override, Approval Approved, Margin Floor Enforced, and 20 more                                 |
|                                                       | `Actor__c`, `TargetObject__c`, `TargetRecord__c`, `Snapshot__c`                               | Who, what object/record, before/after JSON                                                                                                                   |
| Rule History (`RuleHistory__c`)                       | `RecordId__c`, `RecordObject__c`                                                              | Which rule/matrix row this entry is about                                                                                                                    |
|                                                       | `ChangeType__c`                                                                               | Created / Updated / Deleted / Restored                                                                                                                       |
|                                                       | `VersionNumber__c`                                                                            | Monotonic per `RecordId__c`, starting at 1                                                                                                                   |
|                                                       | `BeforeSnapshot__c`, `AfterSnapshot__c`, `ChangedFields__c`                                   | JSON before/after and the changed field list                                                                                                                 |
|                                                       | `RestoredFromId__c`                                                                           | When Restored, the history row the snapshot came from                                                                                                        |
| Catalog (`Catalog__c`)                                | `Status__c`                                                                                   | Draft / **Active** / Inactive — only Active is ever offered to a rep                                                                                         |
|                                                       | `Description__c`                                                                              | Shown in the catalog picker                                                                                                                                  |
| Catalog Product (`CatalogProduct__c`)                 | `Catalog__c`                                                                                  | Master-detail to its Catalog                                                                                                                                 |
|                                                       | `Product__c`                                                                                  | The included product (required by validation rule, not the field itself)                                                                                     |
|                                                       | `DisplayOrder__c`                                                                             | Sort order, lower first                                                                                                                                      |
|                                                       | `Featured__c`                                                                                 | Highlighted in the catalog tile UI                                                                                                                           |
| Engine Limits (`EngineLimits__mdt`)                   | `MaxLinesPerRequest__c`                                                                       | Largest line count priced/committed in one request before a clean refusal; default 1000                                                                      |
| Rate Plan Settings (`DD_CPQ_Rate_Plan_Settings__mdt`) | `Enforcement__c`                                                                              | **Block** (default) / Warn / Off — what happens to a line with no Active rate plan                                                                           |
| Term Settings (`DD_CPQ_Term_Settings__mdt`)           | `CoTermMode__c`                                                                               | Co-term behaviour (co-term module)                                                                                                                           |
| Cart View (`DD_CPQ_Cart_View__mdt`)                   | `IsDefault__c`, `Personas__c`, `Surface__c`, `GroupBy__c`, field-set fields                   | Which cart layout shows for which persona/surface (cart.md)                                                                                                  |
| Cart Rule (`DD_CPQ_Cart_Rule__mdt`)                   | `Field__c`, `Operator__c`, `Value__c`, `Tone__c`, `Views__c`                                  | A banner/badge rule shown in the cart (cart.md)                                                                                                              |

## Build it (tester)

**Settings.** Open the **DD CPQ Settings** tab → group **Engine Health** →
turn on **"Keep an error log"** and **"Keep a history of runs"** → Save.
(This is the only setup needed before the checks below mean anything — a
fresh org keeps nothing.) Leave the Panels (`Off` by default) alone unless
you want the Run/Rules meter visible in the cart.

**Engine Health.** Open the quote you built in the golden path
(`Q-0042 Acme`) → **Configure Products** → change a quantity by 1 and change
it back, to force a fresh pricing run → **Commit** again if you want a
`commit`-kind run kept. Open the **Engine Health** tab → **Runs** → you
should see a row for `Q-0042 Acme`. Click it to open the Run meter and read
its stage breakdown.

**Product 360, with a deliberate gap.** Products tab → **New** → Name
`AD Gap Product`, Active ticked → Save. Do **not** add a standard price or
a rate plan. Open the product record (DeepDive CPQ app) — it opens straight
into Product 360. Read the **Overview** tab's findings.

**Catalogs.** **Catalogs** tab → **New** → Name `AD Test Catalog`, Status
**Draft** → Save. On the Catalog record's related list, **New Catalog
Product** → Product `AD Gap Product` → Save. Go back to the Catalog and set
**Status = Active**. Reopen Product 360 on `AD Gap Product` — the
`NOT_IN_ANY_CATALOG` finding should be gone (verified: it disappeared and
the product's `catalogs` array listed `AD Test Catalog`, status Active).

## Check it (Claude)

Never write a `Setting__c` row, a Catalog, or a Product from here — these
are read-only checks on what the tester built.

```bash
sf data query -o dd-e2e -q "SELECT Name, DDCPQ__Group__c, DDCPQ__Value__c, DDCPQ__ValueType__c FROM DDCPQ__Setting__c"
```

An empty result is normal and correct until the tester saves a setting —
every key still reads its shipped default (see the Settings reference
table above); this is not a bug to chase.

```bash
sf data query -o dd-e2e -q "SELECT MasterLabel, DDCPQ__Enforcement__c FROM DDCPQ__DD_CPQ_Rate_Plan_Settings__mdt"
sf data query -o dd-e2e -q "SELECT MasterLabel, DDCPQ__MaxLinesPerRequest__c FROM DDCPQ__EngineLimits__mdt"
sf data query -o dd-e2e -q "SELECT DDCPQ__Status__c, DDCPQ__Description__c FROM DDCPQ__Catalog__c WHERE Name = 'AD Test Catalog'"
sf data query -o dd-e2e -q "SELECT DDCPQ__Product__r.Name, DDCPQ__DisplayOrder__c, DDCPQ__Featured__c FROM DDCPQ__CatalogProduct__c WHERE DDCPQ__Catalog__r.Name = 'AD Test Catalog'"
```

**Read-only admin REST actions** — all through the single dispatch endpoint
(`/dd/v1/admin`, POST, body `{"action": "<name>", "payload": {...}}`) or the
dedicated settings endpoint (`/dd/v1/settings`, POST). Both never write
unless the action name itself says `save`:

```bash
# every setting, grouped, with the value in force
echo '{"action":"list"}' > body.json
sf api request rest "services/apexrest/DDCPQ/dd/v1/settings" -o dd-e2e --method POST --body @body.json

# the quote's last kept engine run (needs a Quote Id)
echo '{"action":"telemetry.last","payload":{"quoteId":"<0Q0...>"}}' > body.json
sf api request rest "services/apexrest/DDCPQ/dd/v1/admin" -o dd-e2e --method POST --body @body.json

# the limits and budgets every meter measures against
echo '{"action":"telemetry.thresholds","payload":{}}' > body.json
sf api request rest "services/apexrest/DDCPQ/dd/v1/admin" -o dd-e2e --method POST --body @body.json

# everything DD CPQ knows about one product (by Id or ?code=)
sf api request rest "services/apexrest/DDCPQ/dd/v1/product360/<01t...>" -o dd-e2e --method GET
```

`errors.list` needs `Administer Pricing`; a quoting-only user gets a 403
with `"error"` set, not the error list.

## Expected numbers

All verified in cpq-pkg, 2026-10-07.

| Check                                                           | Result                                                                                                                                                                                                               |
| --------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `Setting__c` row count in a fresh org                           | **0** — every one of the 19 settings reads its shipped default                                                                                                                                                       |
| `DD_CPQ_Rate_Plan_Settings__mdt` Default `Enforcement__c`       | **Block**                                                                                                                                                                                                            |
| `EngineLimits__mdt` Default `MaxLinesPerRequest__c`             | **1000**                                                                                                                                                                                                             |
| `/dd/v1/settings {"action":"list"}` group count / setting count | **4** groups / **19** settings, `saved: false` on every one                                                                                                                                                          |
| `/dd/v1/admin {"action":"telemetry.last"}` on the golden quote  | returns a real `EngineRun__c` (`RUN-0000002`): `lineCount: 6`, `soql: 28` of limit 100, `cpuMs: 301` of limit 10000, `worstRatio: 0.28`, `peakStage: "Rules"`, 29 pipeline stages reported, `reference: "DD-3SNSRN"` |
| `/dd/v1/admin {"action":"errors.list"}`                         | returns 2 real kept errors from prior sessions: one `FLOOR_BLOCKED` (margin floor), one `UNEXPECTED` with code `DD-B8CGF3` from a bad `telemetry.last` call with no `quoteId`                                        |
| Product 360 on a product with no PBE, no plan, no catalog       | 3 findings: `NO_ACTIVE_PLAN` (crit), `NO_STANDARD_PBE` (crit), `NOT_IN_ANY_CATALOG` (info); `sellability.status = "Blocked"`, `readyToSell: false`                                                                   |
| Same product, after adding it to an Active Catalog              | `NOT_IN_ANY_CATALOG` gone; `catalogs` array lists the catalog, `status: "Active"`                                                                                                                                    |

## Troubleshooting

| Symptom                                                                                      | Cause                                                                                                                                                                                          | Fix                                                                                                              |
| -------------------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------- |
| Settings tab shows every value but a change never seems to apply                             | The admin saved it, but a **job-starting** setting (telemetry, renewal drafts, expiry) only affects the _next_ nightly run, not anything already scheduled or already computed                 | Check `CronTrigger` for the job's `NextFireTime`; nothing retroactive happens                                    |
| A tester's `sf data query` on `DDCPQ__Setting__c` returns 0 rows even after saving in the UI | Normal when nothing has been saved yet — but if the tester insists they saved something, check they have `Administer Pricing` (`CpqSettings.save` calls `DdCpqAccess.requireAdministration()`) | Re-save as the admin user, or assign the Admin permission set                                                    |
| Engine Health → Runs is empty after a quote was priced                                       | `Engine_Telemetry_Persist` is off (default `false`), or the run was a healthy preview that missed the sample                                                                                   | Turn the setting on, then reprice; a **commit** or a **failed** run is always kept regardless of the sample rate |
| An error the tester just hit doesn't show on the Errors tab                                  | `Error_Log_Enabled` is off (default `false`)                                                                                                                                                   | Turn it on before reproducing; nothing is retroactively logged                                                   |
| `errors.list`/`telemetry.worstRuns` REST call returns 403                                    | Those admin-only reads need `Administer Pricing`; `telemetry.last`/`telemetry.thresholds`/`errors.report` only need quoting access                                                             | Use the Admin permission set user for anything beyond `telemetry.last`/`thresholds`                              |
| "Add a standard price" on Product 360 doesn't add anything                                   | It's a deep-link to Salesforce's own Price Book Entries related list, not an inline create — there is no PBE grid on the Pricing tab in this build                                             | Click through and use **Add Standard Price** there, same as the golden path's Stage 4                            |
| A product still shows `NO_ACTIVE_PLAN` after adding a plan                                   | The plan is **Draft**, not Active — the engine (and Product 360) only count Active plans                                                                                                       | Open the Rate Plan Editor and flip it to Active                                                                  |
| Product 360 opens the plain Salesforce Product page, not the 360 view                        | The tester is in the wrong Salesforce app                                                                                                                                                      | Switch to **DeepDive CPQ** in the App Launcher — the 360 override is app-scoped                                  |
| A catalog's products never show up for a rep                                                 | The Catalog itself is still **Draft** or **Inactive**                                                                                                                                          | Set `Status` to **Active** on the Catalog record, not just on the products                                       |
| `NO_STANDARD_PBE` persists even after adding a price book entry                              | The entry was added to a **non-standard** price book                                                                                                                                           | The finding checks the **standard** price book entry specifically                                                |

## Limits and gotchas

- **`docs/PRODUCT_360_GUIDE.md` undercounts the health codes.** It lists 12;
  the installed build (31444bb) emits **18**. Missing from the doc:
  `NO_BUNDLE_DEFINITION`, `BUNDLE_DEF_INACTIVE`, `FEATURE_UNSATISFIABLE`,
  `PLAN_GROUP_KEY_NULL`, `NO_DEFAULT_IN_GROUP`, `FLOOR_NO_COST`. All six are
  real, already in this build — the full list above is authoritative.
- **The doc also describes a Pricing-tab price-book-entries grid with
  inline edit and a "red bar" to add a standard entry — that UI does not
  exist in the installed LWC.** The Pricing tab only shows the qty
  1/10/100 preview and the embedded Rate Plan Editor; `NO_STANDARD_PBE`'s
  fix button navigates away to Salesforce's standard related list instead.
- **The `NO_ACTIVE_PLAN` message is garbled in this build** — it reads "The
  engine will refuse it on a quote — the engine refuses it on a quote.",
  saying the same thing twice. Cosmetic, not a functional bug.
- **Two different code paths implement the same three Product 360
  inline edits.** The LWC's Apex controller (`CpqProduct360Controller`)
  calls `CpqAdminController.updateProductFeature/updateProductOption/
updateProductFields`; the REST/MCP actions `feature.update`/
  `option.update`/`product.update` call the different class
  `BundleConfigService` instead. Both are maintained, but they are not
  literally "the same method" despite what the class header comment claims
  — a field whitelisted on one side is not guaranteed whitelisted on the
  other.
- **Settings has no per-user override and no environment scoping.** Every
  value is package-wide; there is no sandbox-vs-production split, no
  profile-based variant.
- **The 60%/85% Run meter thresholds are not configurable** — they are
  Apex constants (`EngineHealthService.WARN_AT`/`CRITICAL_AT`), not
  settings, in this build.
- **`AuditLog__c` is a general business-event trail, not Engine Health
  telemetry.** It logs things like Settings Changed, Manual Price Override,
  Approval Approved/Rejected, Margin Floor Enforced — it is not wired into
  the Engine Health tab at all; don't look there for run/error data.
- **`RuleHistory__c` is Rules Studio's own version history** (create/
  update/delete/restore of rules and matrices), unrelated to pricing-run
  performance — don't confuse it with `RuleStat__c` (daily per-rule fire
  counts, which _does_ feed the Engine Health Rules tab).
- **No cron job exists in a fresh org.** All four nightly jobs
  (housekeeping purge, quote expiration, renewal drafts, and whatever
  `AssetProjectionSchedulable` needs) are started lazily, only when their
  triggering setting is first saved into the state that needs it.
- **Governor limits:** Product 360's history timeline caps at 20 rows per
  object / 50 total; Engine Health's error/run lists cap at 200 rows per
  call (`MAX_ROWS`); `RulesJson__c` on a run truncates detail rows past
  the first 300 lines (counts still cover every line).

## Questions testers ask

**Why is the Settings tab showing defaults instead of what I think I set?**
Check you actually clicked **Save** — and that you have `Administer
Pricing`. A `Setting__c` row only exists once a save succeeds; until then
every setting shows its shipped default, which is correct behaviour, not a
bug.

**Why does Engine Health show nothing even though I just priced a quote?**
`Engine_Telemetry_Persist` defaults to **off**. Turn it on in Settings,
then reprice — the next run (and any commit or failure, regardless of the
setting) will show up.

**What is the difference between a run's "reference" and its "trace id"?**
The trace id is a full UUID, generated or adopted fresh per request. The
reference (`DD-XXXXXX`) is a short, human-sayable code deterministically
derived from the first 32 bits of that trace id — same trace id, same
reference, every time. Quote either one; Claude can look up an error or a
run by whichever you give it.

**Why did "Add a standard price" on Product 360 just take me to a blank
related list instead of adding one?**
That's expected in this build — it deep-links to Salesforce's own Price
Book Entries related list rather than creating the entry inline. Click
**Add Standard Price** there.

**I set a catalog's product and it still doesn't show in Product 360 as
"in a catalog."**
Check the **Catalog's** own Status, not the catalog-product row — only an
Active Catalog counts; a Draft or Inactive one is invisible everywhere
except direct record access.

**Does Product 360 ever block a sale itself?**
No — Product 360 only _reports_. The actual block at Commit time comes from
the rate-plan gate (`Enforcement__c = Block`), which Product 360 just
surfaces as the `NO_ACTIVE_PLAN` finding and the `sellability.status =
"Blocked"` verdict.

**Where do I change how long error logs or run history are kept?**
DD CPQ Settings → Engine Health group → "Keep errors for" / "Keep runs
for" / "Keep rule counts for" — each is a day count, enforced by the same
nightly housekeeping job once it's scheduled.
