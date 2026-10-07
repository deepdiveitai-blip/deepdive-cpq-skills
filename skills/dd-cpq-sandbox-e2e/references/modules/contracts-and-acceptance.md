# Accepting a quote, contracts, co-terming, quote dates and expiry

> Installed build 0.1.0-9 · Verified 2026-10-07 in cpq-pkg: accepting a 2-line
> new-business quote (20 × CA Core Seat @ $100, 5 × CA Addon @ $50, 12-month
> term) created one Contract (00000101, Draft, 2026-10-07 → 2027-10-06) and
> two Assets (quantities 20 and 5, same dates), wrote 2 `InstallBaseChange__c`
> rows (`ChangeType = Add`, `AppliedStatus = Applied`), set `Quote.Status =
Accepted`, and created **zero** Orders, OrderItems or `AuditLog__c` rows for
> the acceptance itself. The quote's `ExpirationDate` had already been
> auto-set to Created Date + 30 days (`ExpiryIsDerived__c = true`) with no
> Setting configured — the registry default, not an admin choice.

## What it is for

Everything that happens once a quote is priced and the rep is done editing:
turning it into something the customer has (an Asset), giving it a Contract,
deciding which contract a deal joins and how its term lines up with that
contract (co-terming), and the two quote-level time boxes — the **term** the
customer is committing to, and the **expiration date** the quote itself is
only valid until. This module does not cover amending or renewing what was
accepted — that is amend-renew.md — only the first acceptance of a new deal
and the settings around it.

## How it works

### The button, exactly

The cart (`cpqLedger`) shows a section titled **Accept this sale** only on a
**New Business** quote (`Quote.TransactionType__c` blank or `New Business`).
Its one button reads **"Accept and create what they have"**. Clicking it:

1. Checks for **drift** first (`checkDrift`) — relevant to amendments, not
   new business; a new-business quote has no Baseline to drift from, so this
   step is a no-op here but still runs.
2. Calls Apex `CpqInstallBaseController.acceptAmendment(quoteId, acceptDrift)`
   → `AmendmentService.accept` → (new business, no Baseline) →
   `AmendmentService.acceptNewBusiness`.
3. Toasts "Amendment accepted" with a changes/applied/pending count — the
   same toast copy is used for every kind of acceptance in this build,
   including new business; the word "Amendment" in it is generic UI text,
   not a sign something went wrong.

Same Apex path is reachable headlessly through
`CpqInstallBaseResource` (`POST services/apexrest/DDCPQ/dd/v1/installbase`,
body `{"action":"accept","quoteId":"...","acceptDrift":false}`) and the
`dd_cpq_accept_amendment` MCP tool — there is no separate "new business
accept" endpoint; one action handles both.

### What accepting a new-business quote actually writes

In order, inside one transaction:

1. **One `InstallBaseChange__c` per committed line** (`IsOptional__c = FALSE`
   only — optional/unconfigured option lines are skipped), `ChangeType__c =
'Add'`, `AppliedStatus__c = 'Pending'` at first. A bundle option is
   ordered after its parent (`ORDER BY ParentLine__c NULLS FIRST`) so the
   parent's Asset exists before an option tries to reference it.
2. **`ContractAssignmentService.assign`** gives every new row, and the
   quote, a Contract — see "Which Contract" below. Writes `Contract` (and
   `Quote.ContractId`) and stamps `InstallBaseChange__c.Contract__c`.
3. **`NativeAssetApplier.apply`** turns every `Pending` row whose install
   base is owned by DD CPQ itself (`SourceSystem__c` blank → the
   `NativeAsset` applier) into a standard `Asset`, flips the row to
   `AppliedStatus__c = 'Applied'`, and projects today's position onto it
   (quantity, price, end date).
4. **`PromotionRedemptionService.redeem`** — this is the moment a promotion
   code actually counts as used (see discounts.md); a quote merely priced
   with a code, never accepted, never consumes it.
5. **`Quote.Status` → `'Accepted'`** — standard Quote Status, not a DD CPQ
   field. If an org restricted that picklist and removed `Accepted`, the
   update is swallowed (logged, not thrown): the ledger rows are the real
   record of acceptance, the Status field is a convenience that can be
   missing.

**What does NOT happen:** no `AuditLog__c` row is written for acceptance
itself in this build — verified: a fresh accept added zero new
`AuditLog__c` rows. The only `AuditLog__c` entry in this flow is
`QuoteFinalized`, written earlier at **Commit** (pipeline step 12), not at
Accept. If you are looking for an audit trail of _who accepted what and
when_, the `InstallBaseChange__c` rows (with `CreatedDate`/`CreatedById`)
are it — not `AuditLog__c`.

**Idempotent, once.** Accepting the same quote twice throws "This quote has
already been accepted." (checked by counting existing
`InstallBaseChange__c` rows for that quote) — there is no silent no-op and
no way to re-run acceptance to pick up a line added afterward; that is what
an **amendment** is for.

### Reconciling this with CLAUDE.md rule 8

The `deepdive-cpq` (old monorepo) CLAUDE.md rule 8 — the copy auto-loaded for
this session — reads: _"Native sync handles Order → OrderItem → Asset →
Contract. Do not write any code that creates Orders, OrderItems, Assets, or
Contracts."_ **That sentence is false for this product and is not how it
behaves.** DD CPQ writes `Contract` (`ContractAssignmentService.assign`) and
`Asset` (`NativeAssetApplier`, whose own doc-comment calls itself _"the one
place the package writes Asset"_) directly, on accept — never through a
Salesforce Order/OrderItem sync, because nothing creates an Order at all (see
below). This is not a bug: it is current, intentional design, and this
engine repo's **own** `CLAUDE.md` (the one that governs this codebase) has
the corrected rule 8: _"Never create Orders or OrderItems. Two approved
exceptions: `NativeAssetApplier` writes Asset... and quote acceptance
creates or joins a standard Contract per the Contracting_Method setting."_
The old monorepo's copy is stale/frozen (deepdive-cpq rule 11), never
updated after the design changed. If a tester is coached from the
monorepo's CLAUDE.md, tell them plainly: that rule 8 text describes a
handoff that was designed but never built; the engine repo's rule 8 is the
accurate one.

### Native Salesforce pieces — what DD CPQ does and does not touch

| Native feature               | What DD CPQ does                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                  |
| ---------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Opportunity "Start Sync"** | Nothing DD CPQ-specific. It is the standard feature (syncs `QuoteLineItem` ↔ `OpportunityLineItem`) and still works because DD CPQ's lines are still real `QuoteLineItem` rows with standard `Product2Id`/`Quantity`/`UnitPrice` populated — but it only ever reaches the **Opportunity**, never an Order.                                                                                                                                                                                                                                                                        |
| **Quote PDF**                | DD CPQ ships no PDF generator, no custom Quote template, nothing under "docs/DocRaptor" (that integration is cut, per the monorepo's rule 11). A tester gets a Quote PDF only from the **standard** Salesforce Quote Template / "Create PDF" button on the Quote detail page — unrelated to anything DD CPQ wrote.                                                                                                                                                                                                                                                                |
| **Order / OrderItem**        | **Never created.** Verified: `SELECT COUNT() FROM Order` in cpq-pkg returns 0 after an accept. Standard `Order` has no `QuoteId` and no `OpportunityId` field on a stock org — there is no platform feature that turns a Quote into an Order without a separate licensed product (Revenue Cloud / Salesforce CPQ) or custom code, and DD CPQ deliberately does not add that code (rule 8). If a tester expects "accept the quote, an Order appears," that expectation is simply wrong for this product on a stock org — say so plainly, don't look for a setting that enables it. |

### Which Contract — the `Contracting_Method` setting

DD CPQ Settings tab → **Contracts** group → **Contracting method**
(`CpqSettings.CONTRACTING_METHOD`, picklist, default **`PerQuote`**):

| Value                      | Label                  | Behaviour                                                                                                                                          |
| -------------------------- | ---------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------- |
| `PerQuote` (**default**)   | One per quote          | Every accepted quote gets its own new Contract                                                                                                     |
| `ByEndDate`                | Group by end date      | New lines join an existing Active contract of the same Account + currency that ends on the **same day**; otherwise a new one per distinct end date |
| `SingleContract`           | One master contract    | Everything the Account buys in one currency lands on one Contract, extended (never shortened) to cover the new lines                               |
| `ContractPerWindowedBlock` | One contract per phase | Each Deal Block with its own window gets its own Contract; lines outside any windowed block share the main one                                     |

A rep can override the method for one deal with the **Contract** picker in
the cart header (visible only once the Account already has at least one
Contract — a brand-new Account's first quote never shows it, since there is
nothing yet to join). Options read **"New contract"** (marked "(suggested)"
when nothing else is) or **"`<ContractNumber>` - ends `<EndDate>`"** (marked
"(suggested)" when the engine's own suggestion under the org's method
matches it). Picking an existing contract sets `Quote.ContractId` and
**co-terms the deal to it**: `TermMonths__c` is recalculated from today to
that contract's `EndDate`, and `CoTermMode__c` is forced to `AlignToDealEnd`.

A second setting, **DD CPQ activates contracts** (`Contract_Set_Status`,
boolean, default **off**), decides whether a Contract DD CPQ creates is set
to `Activated` immediately or left `Draft` for the org's own
signature/approval process. Verified: with the setting off (its shipped
default, and untouched in cpq-pkg), the created Contract's `Status` was
**`Draft`**.

### Co-terming

`Quote.CoTermMode__c` (blank = follow the org default in
`DD_CPQ_Term_Settings__mdt`) says what happens when a line's own dates do
not match the deal's:

| Value                 | Label                       | Effect                                                                                                                                                                    |
| --------------------- | --------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `AlignToDealEnd`      | Align to deal end           | The line is shortened to finish when the deal does                                                                                                                        |
| `ExtendDealToLongest` | Extend deal to longest line | The deal's end moves out to the longest line and every other line is carried with it — **proposed to the rep, never applied silently** (it rewrites lines nobody touched) |
| `Independent`         | Independent terms           | Every line keeps its own dates; nothing is forced to match                                                                                                                |

`QuoteLineItem.IndependentTerm__c` overrides this for one line regardless of
the quote's mode. Start/Term/End are three views of two facts
(`CoTermService`): change any one of `StartDate__c`/`TermMonths__c`/its
derived end date and the others are recalculated — the **end date is
authoritative** (`endFrom(start, months) = start.addMonths(months) - 1
day`), so a 12-month term starting 2026-10-07 ends **2027-10-06**, not
2027-10-07 — verified exactly in the Contract created above.

### Quote term header

`Quote.StartDate__c` ("Subscription Start Date") and `Quote.TermMonths__c`
("Term (Months)") are two of the deal-level Quote fields the installed build adds: `StartDate__c`,
`TermMonths__c`, `BillingFrequency__c`, `CoTermMode__c`,
`ExpiryIsDerived__c`, `PromotionCode__c`, `TransactionType__c`,
`Baseline__c`. Every line inherits `StartDate__c`/`TermMonths__c` as its own
Effective Date/term unless the line sets its own — that line-level override
is what makes a co-termed deal actually co-termed. There is deliberately no
`Quote.EndDate__c`: it is always start + term, computed, never stored.

### Quote expiry — a different time box from the term

The **term** (above) is how long the _subscription_ runs. **Expiration**
(`Quote.ExpirationDate`, standard field) is how long the **quote itself**
(the offer/pricing) stays valid — unrelated concepts that happen to both
live on the header. DD CPQ Settings → **Quote Expiration Date** group:

| Setting                                | Default          | Meaning                                                                                                                                                  |
| -------------------------------------- | ---------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Quote Expiration Date                  | **On**           | Every new quote gets a date automatically                                                                                                                |
| Rep can edit the date                  | **On**           | A typed date is never auto-overwritten                                                                                                                   |
| Default validity                       | **30 days**      | Offset added to the basis                                                                                                                                |
| Count from                             | **Created date** | or **Last commit** — a derived date then moves forward on every commit                                                                                   |
| When the date has passed               | **Warn**         | **Off** nothing happens · **Warn** the cart shows a banner offering "Reprice to today" · **Block** same banner, and **Commit is refused** until repriced |
| Move expired drafts to Expired nightly | **Off**          | A Draft quote past its date becomes `Expired` each night; repricing brings it back to `Draft`                                                            |

There is **no install seed** for any of this — a `Setting__c` row exists
only once an admin opens the Settings tab and saves it. Verified: in
cpq-pkg, with **zero** `Setting__c` rows saved (an admin never touched the
tab), a brand-new quote still got `ExpirationDate = CreatedDate + 30 days`
and `ExpiryIsDerived__c = true` — the registry's own defaults apply with no
row present, not just after an admin saves one.

`Quote.ExpiryIsDerived__c` records who set the date: `true` when DD CPQ
worked it out, `false` the moment a rep (or integration) types one — a
typed date is permanent and never silently replaced. The date is stamped at
the **first** DD CPQ touch — opening the cart on a quote with no lines yet,
a commit, a header save, or New Quote — there is no Quote trigger.

**Block enforcement's exact refusal** (`QuoteExpirationService.
requireNotExpired`, thrown from inside Commit): `"Expired quote, reprice it
first."` — this is checked at **Commit**, not at Accept; an already-committed
but not-yet-accepted quote that later expires can still be accepted (Accept
has no expiry gate of its own in this build).

## Objects and fields

| Object label (API name)                         | Field                                                                                     | Meaning / values                                                                                                    |
| ----------------------------------------------- | ----------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------- |
| Quote (standard, extended)                      | `StartDate__c`                                                                            | Subscription Start Date; every line's default Effective Date                                                        |
|                                                 | `TermMonths__c`                                                                           | Term (Months) the customer is committing to                                                                         |
|                                                 | `CoTermMode__c`                                                                           | `AlignToDealEnd` / `ExtendDealToLongest` / `Independent`; blank = org default                                       |
|                                                 | `TransactionType__c`                                                                      | `New Business` (blank reads as this) / `Renewal` / `Amendment` / `Cancellation` / `Restructure`                     |
|                                                 | `Baseline__c`                                                                             | The sealed Baseline this is quoted against; **blank on new business**                                               |
|                                                 | `ExpiryIsDerived__c`                                                                      | True = DD CPQ set `ExpirationDate`; false = a rep typed it                                                          |
|                                                 | `ExpirationDate` (standard)                                                               | The quote's own validity date, not the subscription term                                                            |
|                                                 | `ContractId` (standard)                                                                   | Set by `ContractAssignmentService` on accept, or earlier by the rep's pick                                          |
|                                                 | `Status` (standard)                                                                       | `Accepted` once acceptance succeeds, if the org's picklist allows it                                                |
| Contract (standard, **no custom fields**)       | `StartDate`/`EndDate`                                                                     | From the earliest/latest line it carries                                                                            |
|                                                 | `Status`                                                                                  | `Draft` unless **DD CPQ activates contracts** is on, then `Activated`                                               |
| Asset (standard, **no custom fields**)          | `Quantity`, `Price`, `PurchaseDate`, `UsageEndDate`, `Status`                             | A projection of the ledger's position **today** — kept current by `NativeAssetApplier.project`, never the future    |
|                                                 | `ParentId`                                                                                | A bundle option's Asset points at its parent's Asset                                                                |
| Baseline (`Baseline__c`)                        | `Account__c`, `AsOf__c`, `Scope__c`, `Status__c`                                          | A sealed snapshot of what the Account had — **only exists for amendments/renewals**, never new business             |
| Baseline Line (`BaselineLine__c`)               | `Product__c`, `Quantity__c`, `SourceAsset__c`, `Contract__c`, `TermStart__c`/`TermEnd__c` | One row per owned position as of the snapshot                                                                       |
| Install Base Change (`InstallBaseChange__c`)    | `ChangeType__c`                                                                           | `Add` / `Increase` / `Decrease` / `Remove` / `Renew` / `Swap` — append-only, never edited for billing state         |
|                                                 | `AppliedStatus__c`                                                                        | `Pending` → `Applied` (or stays `Pending` if the install base belongs to another system)                            |
|                                                 | `Asset__c`, `Contract__c`                                                                 | Stamped once `NativeAssetApplier`/`ContractAssignmentService` run                                                   |
|                                                 | `QuantityAfter__c`, `MrrAfter__c`, `QuantityDelta__c`, `MrrDelta__c`                      | Position after the change, and the delta from before                                                                |
|                                                 | `EffectiveDate__c`, `TermEndAfter__c`                                                     | When the change takes effect and when that position's term ends                                                     |
| Promotion Redemption (`PromotionRedemption__c`) | `PromotionRule__c`, `Quote__c`, `Account__c`                                              | One row per rule per **accepted** quote — see discounts.md; a promo code on a never-accepted quote never writes one |
| AuditLog (`AuditLog__c`)                        | `Action__c`                                                                               | `QuoteFinalized` is written at **Commit**; acceptance itself writes none in this build                              |

## Build it (tester)

A new-business scenario, reusing or extending the golden-path quote.

1. **Products** — create **Core Seat** (Standard Price $100) and **Add-on**
   (Standard Price $50), each Active, each with an Active rate plan
   (Recurring, Per Unit, Monthly) — same pattern as golden-path Stages 4 and 6.
2. **Account** `Contracts Test Co`. **Opportunity** `Contracts Test Co - New
Business`, Stage Proposal/Price Quote.
3. **Quote** `Contracts New Business` on that Opportunity. Open **Configure
   Products**. Note the header's expiration date is already filled in
   (today + 30 days) — nobody typed it.
4. **Set the term.** On the quote header, set **Subscription Start Date**
   to today and **Term (Months)** to **12**.
5. **Add products** → Core Seat qty **20**, Add-on qty **5**. **Commit.**
6. Scroll to **Accept this sale** → click **"Accept and create what they
   have"**.
7. Confirm the toast reads "Amendment accepted" with "2 changes, 2 applied,
   0 pending" (the word "Amendment" is generic copy, not an error).
8. Go to the **Account** → open the **Account Install Base** record page
   (if it is not the default page shown, use the gear icon → **Edit Page**
   once, or App Launcher search for the Account and switch Lightning page to
   "Account Install Base") — confirm two Assets with quantities 20 and 5.
9. Open the new **Contract** related to the Account; confirm Start/End
   dates are today and (today + 12 months − 1 day), and Status is **Draft**
   (DD CPQ activates contracts is off by default).
10. **Co-term check**: start a second quote on the same Account. In the cart
    header you should now see a **Contract** dropdown with the first
    contract listed as "`<number>` - ends `<date>`"; picking it co-terms the
    new quote's term to that contract's end date.

## Check it (Claude)

```bash
sf data query -o dd-e2e -q "SELECT Id, Status, ContractId, DDCPQ__StartDate__c, DDCPQ__TermMonths__c, DDCPQ__TransactionType__c, DDCPQ__Baseline__c, ExpirationDate, DDCPQ__ExpiryIsDerived__c FROM Quote WHERE Name = 'Contracts New Business'"

sf data query -o dd-e2e -q "SELECT Id, ContractNumber, Status, StartDate, EndDate FROM Contract WHERE AccountId IN (SELECT Id FROM Account WHERE Name = 'Contracts Test Co')"

sf data query -o dd-e2e -q "SELECT Id, Name, Product2.Name, Quantity, Price, PurchaseDate, UsageEndDate, Status, ParentId FROM Asset WHERE AccountId IN (SELECT Id FROM Account WHERE Name = 'Contracts Test Co')"

sf data query -o dd-e2e -q "SELECT DDCPQ__ChangeType__c, DDCPQ__Product__r.Name, DDCPQ__QuantityAfter__c, DDCPQ__MrrAfter__c, DDCPQ__AppliedStatus__c, DDCPQ__Asset__c, DDCPQ__Contract__c, DDCPQ__EffectiveDate__c, DDCPQ__TermEndAfter__c FROM DDCPQ__InstallBaseChange__c WHERE DDCPQ__Quote__r.Name = 'Contracts New Business'"

sf data query -o dd-e2e -q "SELECT COUNT() FROM Order"

sf data query -o dd-e2e -q "SELECT COUNT() FROM DDCPQ__Baseline__c WHERE DDCPQ__Account__r.Name = 'Contracts Test Co'"
```

The last two should both read **0** right after a new-business accept — any
Order row, or any Baseline row for an account that has only ever done new
business, means something unexpected happened; investigate rather than
assume it is fine.

**Read-only check of what Accept will do before the tester clicks it** —
never call the accept action yourself; only read:

```bash
sf data query -o dd-e2e -q "SELECT COUNT() FROM QuoteLineItem WHERE QuoteId = '<quote Id>' AND DDCPQ__IsOptional__c = false"
```

A quote with 0 such rows is why acceptance would fail with "This quote has
no lines, so there is nothing for the customer to have." — tell the tester
to commit first.

## Expected numbers

All from the verification run above, cpq-pkg, 2026-10-07, 12-month term
starting that same day.

| What                                          | Value                                                                                                  |
| --------------------------------------------- | ------------------------------------------------------------------------------------------------------ |
| Contract Start / End                          | 2026-10-07 / **2027-10-06** (12 months − 1 day)                                                        |
| Contract Status (default setting)             | **Draft**                                                                                              |
| Core Seat Asset                               | Quantity **20**, Price **100**, PurchaseDate 2026-10-07, UsageEndDate 2027-10-06, Status **Purchased** |
| Add-on Asset                                  | Quantity **5**, Price **50**, same dates                                                               |
| InstallBaseChange rows                        | **2**, both `ChangeType = Add`, `AppliedStatus = Applied`                                              |
| Quote.Status after accept                     | **Accepted**                                                                                           |
| Orders created                                | **0**                                                                                                  |
| Baseline rows for this account                | **0** (new business never seals one)                                                                   |
| New AuditLog rows from the accept call itself | **0**                                                                                                  |
| ExpirationDate with no Setting row saved      | CreatedDate **+ 30 days**, `ExpiryIsDerived__c = true`                                                 |
| Block enforcement refusal text                | `Expired quote, reprice it first.`                                                                     |

## Troubleshooting

| Symptom                                                                                                                                   | Cause                                                                                                                                                   | Fix                                                                                                        |
| ----------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------- |
| "This quote has no lines, so there is nothing for the customer to have."                                                                  | Lines were added but never **Commit**ted — Accept only reads real `QuoteLineItem` rows, not cart drafts                                                 | Commit first, then Accept                                                                                  |
| "This quote has already been accepted."                                                                                                   | An `InstallBaseChange__c` row already exists for this quote — accept is one-shot                                                                        | Nothing to do; if the customer needs more, start an amendment, not a second accept                         |
| "This `<transactiontype>` quote was not started from a Baseline, so there is nothing to compare it with. Start it from Install Base 360." | `TransactionType__c` is `Renewal`/`Amendment`/`Cancellation` but `Baseline__c` is blank                                                                 | That quote was created the wrong way — renewals/amendments must start from Install Base 360, not New Quote |
| No Contract/Asset appears after accept, but the toast said success                                                                        | Looking at the wrong Account, or the page hasn't refreshed — `NativeAssetApplier` only skips a change when `SourceSystem__c` names another system       | Re-query by `AccountId`, not by memory of which page you were on                                           |
| Contract Status shows Draft when you expected Activated                                                                                   | **DD CPQ activates contracts** setting is off (the shipped default)                                                                                     | Turn it on in Settings, or activate the Contract by hand — this is normal, not a defect                    |
| Commit refused with "Expired quote, reprice it first."                                                                                    | **When the date has passed** is set to **Block** and `ExpirationDate` is in the past                                                                    | Tell the tester to use the cart's "Reprice to today" banner, or widen **Default validity**                 |
| Expiration date never changes after repeated commits                                                                                      | **Count from** is set to **Created date** (the default) — only **Last commit** moves it forward                                                         | Expected; switch the basis if a moving date is wanted                                                      |
| "Each needs an Active rate plan" when adding a product, unrelated to this module                                                          | The Rate Plan Gate (pricing-models.md), unrelated to acceptance, runs earlier in the pipeline                                                           | Add an Active rate plan, then retry — nothing to do with contracts                                         |
| Tester expects an **Order** after accept and sees none                                                                                    | Rule 8's real behaviour: nothing turns a Quote into an Order on a stock org                                                                             | Expected — see "Native Salesforce pieces" above; do not look for a setting                                 |
| Contract picker missing in the cart header on a brand-new Account                                                                         | `showContractPicker` only renders once `contractChoices.contracts` is non-empty — a first-ever quote for an Account has no contracts to choose from yet | Expected on the first deal; it appears from the second quote onward                                        |

## Limits and gotchas

- **No custom fields on `Asset` or `Contract`**, by rule — anything DD CPQ
  needs to remember about an accepted position lives on
  `InstallBaseChange__c`/`BaselineLine__c`, not bolted onto the standard
  objects.
- **`NativeAssetApplier` only applies rows whose install base it owns**
  (`SourceSystem__c` blank, resolved to the `NativeAsset` applier). A row
  whose system of record is external stays `Pending` forever in this org —
  by design, for an owner to apply and acknowledge outside Salesforce. A
  tester working a single clean org will never see this path; it matters
  only once another system is wired in. Such rows' `AppliedStatus__c` stays
  `Pending`; "applied" vs "changes" in the toast tells you how many of each
  you got.
- **Accept has no expiry gate of its own.** Only **Commit** checks Block
  enforcement (`QuoteExpirationService.requireNotExpired`). A quote
  committed before its expiration date passed can still be accepted after
  it has — that is intentional (the price was already locked in at commit),
  not a bug to report.
- **The toast text says "Amendment" for every acceptance**, including new
  business. This is cosmetic, not a sign the quote was mis-typed as an
  Amendment — check `TransactionType__c` directly if in doubt.
- **No seed data, no default Setting rows.** Every number in the "Quote
  Expiration Date" / "Contracts" groups is the registry's hard-coded
  default until an admin opens the Settings tab and saves a change — do not
  assume an org "must have" a `Setting__c` row to behave correctly.
- **Governor limits:** acceptance is bulk-safe for the number of lines one
  quote can realistically carry (no SOQL/DML per line; `NativeAssetApplier`
  batches all its Asset inserts/updates in two statements regardless of
  line count) — nothing here is a limits risk a tester would hit by hand.

## Questions testers ask

**I clicked Accept and the toast said "Amendment accepted" — did I
accidentally create an amendment?**
No. That copy is shared by every kind of acceptance in this build,
including ordinary new business. Check `Quote.TransactionType__c` if you
want to be sure — it still reads `New Business` (or blank).

**Where's the Order?**
There isn't one, and there never will be on a stock org without a separate
licensed product. DD CPQ's Accept creates a Contract and Assets directly —
that is the whole handoff. Don't look for an Order Settings toggle; none
exists for this.

**I set the Contracting method to "One master contract" but a second
quote still made a new Contract.**
Check the method actually saved (Settings tab), and check the two quotes
are for the same Account **and** the same currency — `SingleContract` never
spans currencies.

**Why is my new Contract sitting in Draft when the quote clearly says
Accepted?**
The **DD CPQ activates contracts** setting is off by default — the package
deliberately leaves contract activation to your own signature/approval
process unless you turn that setting on.

**I typed a shorter Expiration Date by hand and it keeps getting
overwritten.**
It shouldn't — a typed date sets `ExpiryIsDerived__c` to false and the
service never touches it again. If it's moving anyway, check whether
**Count from** is **Last commit**: that only moves a _derived_ date, so
confirm `ExpiryIsDerived__c` really is false on that quote.

**Can I accept a quote twice to add more lines later?**
No — "This quote has already been accepted." is permanent for that quote.
Additional lines are a new amendment started from Install Base 360, not a
second accept.

**Does the term (12 months) or the expiration date (30 days) control when
my quote stops being editable?**
Neither directly controls "editable" — the term is the subscription length
once accepted; expiration only affects whether **Commit** is blocked
(under the Block setting) or just warned about. A quote can be fully
editable long past its expiration date under Warn or Off.

**My Account's first-ever quote shows no Contract picker — is that
broken?**
No — the picker only lists contracts the Account already has. A first deal
always creates a new one; the picker appears starting with the second
quote.
