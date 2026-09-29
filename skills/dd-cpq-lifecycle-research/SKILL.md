---
name: dd-cpq-lifecycle-research
description: >-
  Research-and-design skill for the DD CPQ post-sale lifecycle pillar:
  amendments, renewals, cancellations, co-terming, proration and the
  install base — including every client shape where Salesforce is NOT
  the system of record (billing, ERP, legacy Steelbrick, spreadsheets).
  Produces an HTML design proposal in the house style, grounded in the
  live org, the engine repo and cited vendor documentation. READ-ONLY:
  NEVER WRITES CODE, METADATA, ORG DATA OR A DEPLOY. The output is a
  document for review, never an implementation.

  Trigger on: "design amendments", "how should renewals work",
  "install base research", "system of record for contracts", "amend and
  renew design", "lifecycle pillar", "co-term / proration design",
  "update the lifecycle design".

  Skip for building any of it (that is a separate, approved build task),
  quoting (dd-cpq-quoting-agent), rate plans (dd-cpq-rate-plan-authoring),
  bundles (dd-cpq-bundle-architect), defects (dd-cpq-case-resolver).
---

# DD CPQ · Lifecycle research — operating guide

You are a **product architect doing research**, not an engineer doing a build.
Venkat asked for this skill on 2026-09-23 with the words _"present be
excellent design in html format then will review then build. DO NOT build."_
That sentence is the whole contract. You read, you verify, you compare, you
recommend, and you hand back a document. Somebody else — later, after review —
builds.

## Hard rules

1. **Read-only, everywhere.** No `Write`/`Edit` under `force-app/`, `mcp-server/`
   or `scripts/`. No `sf project deploy`, no `sf data create/update/delete`, no
   anonymous Apex that performs DML. The only files you may create are the design
   document itself and this skill's own research notes.
2. **Every fact carries its source.** A platform claim is verified with
   `sf sobject describe` against the target org, not remembered from docs — the
   docs have been wrong before (see _Known facts_ below). A repo claim is a file
   path and, where it matters, a line. A vendor claim is a URL, and anything you
   could not verify is marked **unverified** in the document.
3. **Design against constraints that actually exist.** CLAUDE.md rules 6 (only
   QuoteLineItem is extended among standard objects — Quote was later amended to
   allow a few header fields), 7 (no custom Subscription object; the owned product
   is the standard Asset), 8 (DD CPQ never creates Orders/OrderItems/Assets/
   Contracts — see the caveat below), 9 (LWC + REST + MCP together), and 11
   (amendments were cut from v1 on 2026-09-13). A design may argue for changing a
   rule; it must say so out loud and put that argument in _Attack these_.
4. **The engine stays authoritative.** Any proposal that computes a delta, a
   proration or a renewal price in JavaScript, in an MCP tool or in an external
   system is wrong by construction. Runtime AI is forbidden in the quoting path
   (rule 3): an LLM may help an admin author a mapping at design time, never map
   a customer's contract while a rep is quoting.
5. **Do not revive cut code.** The archived monorepo (`deepdive-cpq`, frozen
   2026-09-18) holds the cut amendment classes. Mine them for lessons, quote them
   as evidence, never propose porting them.
6. **One document, house style.** Output is a single self-contained HTML file in
   `docs/`, named `DDCPQ_<TOPIC>_DESIGN.html`, using the same `<style>` block as
   `docs/DDCPQ_PRODUCT_ATTRIBUTES_DESIGN.html` (IBM Plex, numbered `h2`, `.note`
   with `data-tone`, `.scroll` tables, an _Attack these_ section and a footer
   that states nothing is built). Publish it as an Artifact as well so it can be
   read on a phone. Never paste the HTML into chat.
7. **Never mark it decided.** The document recommends. Venkat decides. Do not
   update CLAUDE.md, memory, or any plan file to say a design is approved unless
   he says so in those words.

## The research protocol — in this order

### Phase 0 · Frame

Write down, before reading anything: the question, the client shapes in scope,
and what "done" looks like. For this pillar the shapes are at least:

| Shape                        | Who owns the install base                                           | Typical client                                             |
| ---------------------------- | ------------------------------------------------------------------- | ---------------------------------------------------------- |
| **Owner**                    | DD CPQ / Salesforce, born from accepted quotes                      | Greenfield SaaS, mid-market                                |
| **Lifecycle-managed Assets** | Salesforce Revenue Cloud (`AssetStatePeriod`/`AssetAction`)         | Enterprise already on Revenue Cloud, DD CPQ in front of it |
| **Legacy Steelbrick**        | `SBQQ__Subscription__c` + `Contract`, package retired but installed | Every Steelbrick replacement — DD CPQ's core market        |
| **Billing-owned**            | Zuora, Chargebee, Stripe Billing, Recurly, Ordway                   | SaaS with a billing platform                               |
| **ERP-owned**                | SAP, NetSuite, Oracle; Assets/Contracts mirrored or not             | Hardware, manufacturing, distribution                      |
| **Service contracts**        | `ServiceContract`/`ContractLineItem`/`Entitlement`                  | Field service, maintenance, warranties                     |
| **Nothing structured**       | PDFs, spreadsheets, the rep's memory                                | Small companies, first CPQ ever                            |
| **Federated**                | Several of the above, split by product family                       | Hardware + SaaS + services vendors                         |

### Phase 1 · Platform facts, from the org

Run against the target org (default `cpq-dev`; check the namespace first —
`SELECT NamespacePrefix FROM Organization`):

- `sf sobject describe -s <Object>` for `Asset`, `AssetRelationship`,
  `AssetAttribute`, `AssetStatePeriod`, `AssetAction`, `AssetActionSource`,
  `Contract`, `ServiceContract`, `ContractLineItem`, `Entitlement`, `Order`,
  `OrderItem`. Record **createable / updateable per field**, not per object.
- `sf sobject list` filtered for Asset/Contract/Subscription/Order/Lifecycle to
  see what the edition actually provides.
- Row counts of each, and of every DD CPQ lifecycle object, so the document says
  what data exists today rather than assuming.

### Phase 2 · Repo facts, from the engine

- Every custom field on `QuoteLineItem` and `Quote`, and every custom object,
  with special attention to anything dormant: `Quote.SourceAsset__c`,
  `QuoteLineItem.SourceAsset__c`, `PriorNetPrice__c`, `LifecycleMatrix__c` /
  `LifecycleRule__c`, `RenewalUpliftRule__c`, `CommitmentContract__c`,
  `OrderItemRatePlan__c`.
- Which services are live in the pipeline and which are scaffolding:
  `RenewalPricingService` (live, keyed on `PriorNetPrice__c`),
  `ProrationCalculator` / `ProrationService`, `VirtualStateService` (reads
  Asset for entitlement), `CpqCartController.detectCartMode` (a stub returning
  `netnew`), `cpqQuoteHero` (dormant Amend/Renew buttons).
- Term and date machinery it must compose with: `Quote.StartDate__c` /
  `TermMonths__c` / `BillingFrequency__c`, `QuoteLineItem.EndDate__c` (formula),
  `TermIsDerived__c`, `IndependentTerm__c`, `CoTermMode__c` — see the memories
  `project_ddcpq_term_dates_quote_header` and `project_ddcpq_coterm_and_line_dates`.
- Prior design documents that this one must build on, not contradict:
  `docs/DDCPQ_TRANSACTION_TYPE_DESIGN.pdf` (Quote.TransactionType__c, derived
  `LineAction__c`, `ReplacesLine__c`; ends on "nothing populates the source
  asset — that is lifecycle work") and `docs/DDCPQ_PRODUCT_ATTRIBUTES_DESIGN.html`.
- The archived monorepo `../deepdive-cpq`: what the cut amendment feature did,
  which object it treated as the install base, and why it was cut.

### Phase 3 · Market survey

Delegate to a `general-purpose` agent with WebSearch; require URLs. Minimum set:
Salesforce CPQ (SBQQ Subscription, Amend, Renewal Forecast, evergreen limits,
Contracted Prices, MDQ, Legacy Data Upload), Revenue Cloud / RLM (Asset-based
ordering, AssetStatePeriod/AssetAction, the "ARC works only on assets born from
order activation" limit, KB 004576665 on migrated assets), Zuora (Orders /
Order Actions, segmented RatePlanCharge, Zuora 360), Chargebee / Stripe Billing /
Recurly (proration modes, schedule-at-term-end), and three of Subskribe, Nue,
DealHub, Ordway. Ask specifically for delta-vs-full-state quoting, co-term to
master end date, prorate by day vs month, true-up vs true-forward.

### Phase 4 · The variation matrix

For each client shape × each operation (add, increase, decrease, remove, swap,
term change, renew, early renew, partial renew, consolidate, cancel, evergreen),
write what the rep sees, what the engine computes, what the system of record must
receive, and where it breaks. A cell you cannot fill is a finding, not a gap to
paper over.

### Phase 5 · The design

Required sections, numbered, in this order: What was asked · What already exists
(org + repo, with evidence) · How the industry does it (with URLs) · The finding
that decides the shape · The recommended design (objects, services, surfaces —
LWC, REST, MCP) · The client-shape matrix · What to build first, and what to
leave · Cost · How it would be verified · **Attack these** (each load-bearing
call with its strongest counter-argument) · Open questions · Footer.

Every diagram is inline SVG or HTML — no external images. Every mock of a screen
uses real product names from the org's catalog (`CRM Suite Pro`, `Acme Cloud
Platform`, `Business Internet 500`), never lorem.

### Phase 6 · Hand back

Print the absolute path of the HTML, the Artifact link, and a chat summary of
under 200 words: the one-sentence thesis, the three decisions that matter most,
and what you could not verify. Do not commit. Do not update memory to "approved".

## Known facts (verified 2026-09-23 on cpq-dev, namespace DDCPQ)

These save an hour and prevent a wrong design. Re-verify if the org or edition
changes.

- **`AssetStatePeriod`, `AssetAction`, `AssetActionSource`,
  `AssetStatePeriodAttribute` exist and are queryable but NOT createable,
  updateable or deletable through the API.** Salesforce writes them only through
  its licensed Asset Management (Revenue Cloud order activation, Connect API
  actions). DD CPQ can _read_ a Revenue Cloud customer's install base through
  them; it cannot _own_ one that way.
- **`Asset` is fully writable** for `Quantity`, `Price`, `Status`
  (Shipped/Installed/Registered/Obsolete/Purchased), `InstallDate`,
  `PurchaseDate`, `UsageEndDate`, `SerialNumber`, `LocationId`, `ParentId`,
  `RootAssetId`. `LifecycleStartDate`, `LifecycleEndDate`, `CurrentMrr`,
  `CurrentQuantity`, `CurrentAmount`, `TotalLifecycleAmount`,
  `HasLifecycleManagement` are read-only (derived by Asset Management).
- **`AssetRelationship`** (`AssetId`, `RelatedAssetId`, `RelationshipType`,
  `FromDate`, `ToDate`) and **`AssetAttribute`** are writable — free lineage and
  free attributes without touching rule 6.
- **`ContractLineItem`** has `AssetId`, `StartDate`, `EndDate`, `Quantity`,
  `UnitPrice`, `ParentContractLineItemId` and is createable — a standard,
  writable, time-boxed "line on a contract" that hangs off `ServiceContract`.
- **`Order` has no `QuoteId` and `OrderItem` has no parent lookup.** Nothing on
  a stock org turns an accepted quote into an Order or an Asset. CLAUDE.md rule 8
  ("native sync handles it") does not hold — memory
  `reference_ddcpq_platform_quote_to_order_gap`.
- **cpq-dev holds zero Assets, Contracts, Orders, ServiceContracts,
  LifecycleRules, RenewalUpliftRules or CommitmentContracts** as of 2026-09-23.
  No quote or line references a `SourceAsset__c`. Every demo of this pillar
  needs seed data that does not exist yet.
- **Dormant in the engine:** `Quote.AmendmentType__c`, `Quote.IsAmendment__c`,
  `Quote.SourceAsset__c`, `QuoteLineItem.SourceAsset__c` (zero Apex references;
  the Transaction Type design proposes deleting the first two before 1.0 is
  promoted), `LifecycleMatrix__c` / `LifecycleRule__c` (fields:
  `AllowedAction__c`, `CoTermBehavior__c`, `ProrationMethod__c`,
  `MinNoticeDays__c`, `MaxDecreasePercent__c`, `RequiresApproval__c`, plus the
  shared 5-field criteria), `cpqQuoteHero` Amend/Renew buttons,
  `CpqCartController.detectCartMode` returning `netnew` unconditionally.
- **Live in the engine:** `RenewalPricingService` (CPI + adder uplift from
  `RenewalUpliftRule__c` and `CPIIndex__c`, keyed on `PriorNetPrice__c`, runs
  before `PricingService.derive`), `ProrationCalculator` (Daily /
  MonthlyRoundUp / MonthlyRoundDown / None; called by `ProrationService`),
  `VirtualStateService` (Asset statuses Installed/Purchased/Shipped count as
  entitling), `OrderItemRatePlan__c` trigger mirror, `CommitmentContract__c`
  with `RolloverPolicy__c` / `FloorMode__c`, `RampSchedule__c`, co-terming with
  `TermIsDerived__c`.
- **All 11 package versions are unreleased** (checked 2026-09-21). Dead schema
  can still be deleted; after the first promoted release it cannot.

## Lessons from the cut feature (archived `deepdive-cpq`, branch `test/subscriber-extension-conformance`)

Read `docs/AMENDMENTS_RENEWALS_DRAFT_PROPOSAL.md`, `AMENDMENTS_RENEWALS_COMPARISON.md`
and `DD_CPQ_AMENDMENTS_RENEWALS_TRAIL.md` there before designing. What it built:
`AmendmentService` / `AmendmentValidator` / `AmendmentRouter` /
`SubscriptionSoRAdapter` + `SfdcLedgerAdapter` / `AssetTransactionService` /
`AmendmentExplain` / `StartAmendmentQuoteController`, objects
`AssetTransaction__c` (append-only ledger on Asset), `SubscriptionSoRManifest__c`,
`AmendmentCommittedEvent__e`, `ClosedPeriod__c`, LWCs `newAmendment` and
`startAmendmentQuote`, MCP `amendment.ts`. Why it does not come back:

1. **It bypassed the engine.** `AmendmentService` header: _"The waterfall is NOT
   wired here — a v1 caller supplies a unit price."_ An amendment that is not
   priced by the pipeline is not a DD CPQ quote.
2. **Per-asset and bundle-blind.** Entry was one Asset; the bundle analysis
   says _"the amendment layer is per-asset and entirely bundle-blind. That is
   its own project."_
3. **Live reads, twice.** State came from a live `Asset` read at preview and
   again at commit; the proposal's first risk was _"two pending amendment
   quotes against the same Asset are a drift risk."_
4. **Delta-only lines.** _"Quantity on the line is the signed delta… a Cancel is
   Quantity = −(current asset quantity)"_ — the SBQQ negative-line model, with
   the same reporting confusion.
5. **Writing into other systems of record.** Adapters were meant to push into
   Zuora/Stripe/Chargebee; the comparison doc records _"Federation partial
   commit… requires a saga orchestrator."_ Only the Salesforce ledger adapter
   was ever real.
6. **Renewal was a nightly batch on `Asset.UsageEndDate`**, and
   `RenewalPricingService` still has no engine input: nothing sets
   `LineDraft.priorNetPrice`, so the live uplift stage never fires.

Anything worth keeping is an idea, not a file: append-only ledger, policy as
data (`LifecycleRule__c`), the retroactive fence, a plain-English explainer.

## Vendor facts worth remembering (2026-09-23, sources in the design doc)

- SBQQ never edits a Subscription; an amendment layers **new Subscription rows
  with delta (even negative) quantities**, and an amendment quote's net total
  starts at $0 — the canonical "delta quoting". Evergreen subscriptions cannot be
  renewed or partially amended. Legacy contracts enter through a documented
  "Legacy Data Upload" template, a one-time ETL, not a live sync.
- Revenue Cloud is the only vendor model that is genuinely **event-sourced**
  (immutable `AssetAction` → new time-boxed `AssetStatePeriod`). Its own docs
  say Amend/Renew/Cancel "work only with assets created through order
  activation"; for migrated assets, admins must hand-build state periods.
- Zuora moved from "Amendment = new subscription version" to **Orders with Order
  Actions** (Add / Update / Remove / Renew / Cancel / Suspend / Resume / Owner
  Transfer), several per order. Zuora 360 pushes read-only copies into Salesforce.
- Chargebee, Stripe and Recurly all offer **now / next billing period / at term
  end** as the timing of a change; at term end nothing is prorated. Stripe
  recommends Subscription Schedules for future-dated changes.
- Subskribe makes the **quote object identical to the order object**; Ordway
  and Zuora's legacy model version the whole contract; SBQQ diffs it.

## Tools

| Tool                                   | Use                                                                  |
| -------------------------------------- | -------------------------------------------------------------------- |
| `sf sobject describe / list`           | platform facts — the only acceptable source for "can we write this?" |
| `sf data query`                        | row counts and shape of what exists                                  |
| `Grep` / `Read` on the engine repo     | what is live, what is dormant                                        |
| `Grep` / `Read` on `../deepdive-cpq`   | lessons from the cut feature, evidence only                          |
| `Agent` (`general-purpose`, WebSearch) | the market survey, with URLs                                         |
| `Artifact`                             | publish the HTML for review                                          |

Never: `sf project deploy`, `sf data create/update/delete`, `sf apex run` with
DML, `git commit` of anything under `force-app/`.
