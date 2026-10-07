# The cart — cpqLedger and friends

> Installed build 0.1.0-9 · Verified 2026-10-07 in an installed org (cpq-pkg):
> committed a CT quote (CRM Suite Pro bundle qty 10 + 4 options + Compliance
> Add-on qty 5) through `/dd/v1/commit`, read it back through
> `/dd/v1/cart` (bootstrap action) and `/dd/v1/blocks` (suggest action) —
> totalArr $99,000, bundle parent passthrough waterfall confirmed, Phase-split
> suggestion returned two blocks with correct per-block ARR ($90,000 /
> $9,000).

## What it is for

The cart is where a rep (or Claude/an agent through REST) turns priced
selections into committed QuoteLineItems. It is the **only** place lines are
written — every edit, every bundle pick, every discount round-trips through
the engine's preview/commit pair so the number on screen is always a number
the engine produced, never one the client computed (CLAUDE.md rule 2). The
component that renders it is `cpqLedger` ("the schema-driven quote cart");
`cpqCart` / `cpqCartModal` / `cpqCartLauncher` are the shell that decides
_where_ it opens, and `cpqWaterfall` / `cpqDealBlocks` / `cpqRunMeter` /
`cpqRulesMeter` are its satellite panels.

The cart does not know what a quote line _is_. Every column, every header
field, every grouping and every row rule comes from Salesforce metadata —
Field Sets and four custom metadata types — so a subscriber admin configures
it entirely in Setup, with no code and no deploy.

## How it works

### Opening it

1. The **Configure Products** quick action lives on the Quote page layout
   (added once, by hand, in the golden path's Stage 3). It is headless — no
   screen of its own — and calls `cpqCartLauncher.invoke()`.
2. The launcher asks the server for the active view's **Surface**
   (`CpqCartController.getCartSurface`):
   - `page` (default) — navigates to the **DD CPQ Cart** tab with the quote
     Id in page state (`c__quoteId`). The cart owns the whole window, the
     Bundle Studio opens _in place_ of the cart rather than as a second
     dialog, and browser Back returns to the quote.
   - `popup` — opens the cart in a modal dialog over the Quote record
     (`cpqCartModal`); the Bundle Studio then opens _over_ that dialog. The
     rep never leaves the quote record.
   - If the server call fails for any reason, the launcher falls back to the
     popup unconditionally — a cart that cannot ask for its surface still
     opens.
3. Opened on the **DD CPQ Cart** tab with no `c__quoteId` at all, the
   component shows the **recent-quotes picker** instead of a cart: a plain
   list, heading **"Open a quote"**, one row per quote with its name, account
   name and line count. Clicking a row loads that quote.

### Layout

- **Header** — an eyebrow row (**Ledger**, the cart's mode label, and, on an
  amendment, a scope chip like "Contract 00000101 · 3 lines"), the quote name
  and the account line, a row of **View** tabs, then a strip of KPI tiles
  (computed, not stored — the installed build adds no roll-up fields to
  Quote) plus any header fields the view's Header Field Set adds. **Choose
  fields** lets the rep add more from the field set's full list; a
  **Collapse/Expand** pill appears once the header gets long.
- **Grid** — the line table (or the Timeline — see below), grouped per the
  view's **Group By**.
- **Toolbar**, above the grid: a **Search lines** box (`Ctrl K` opens the
  command palette from the same box), the **Table / Timeline** segmented
  control, **Group by**, the Deal Blocks **Build blocks ▾** control (see
  below), **Display ▾** (column/density choices), **Undo**, **Add
  products**, and the **Commit** button. An **Unsaved changes** flag appears
  next to Undo whenever the cart has edits not yet committed.
- **Footer** of every block-settings/build-blocks drawer repeats the same
  reminder: **"Prices update as you change. Nothing is saved until
  Commit."** — that sentence is the cart's whole safety model: every preview
  is free, nothing is destructive until the one Commit click.

### Add products drawer

**Add products** opens a drawer that asks the engine's own catalog builder
(`/dd/v1/catalog`, the same one `dd_cpq_list_products` uses) — so a rep sees
exactly what an agent would see. What is offered is the intersection, in
order, of:

1. **The quote's price book** — inherited from the Opportunity. A product
   with no active Pricebook Entry there is never offered.
2. **The Eligibility Matrix**, evaluated against the Account and
   Opportunity — the same filter the engine itself applies, not a second
   copy of it.
3. **A Catalog**, only if the rep picks one.

A **Catalog** dropdown appears only once at least one Active `Catalog__c`
record exists (hidden otherwise). It lists **All products** first, then each
Active catalog with a count — how many of its `CatalogProduct__c` members
survived the price book + eligibility filter — so the rep knows what picking
it would cost before picking it. Switching catalogs is instant; no extra
round trip. A product in no catalog is still sellable, just never shown
under a named one. A `Featured__c` catalog member shows a **featured** tag.

**Search** matches the product list by name live. **Group by** controls what
the chip row above the list groups on:

| Group by   | Chips are                                         |
| ---------- | ------------------------------------------------- |
| **Family** | `Product2.Family`                                 |
| **Kind**   | what adding the product _does_ — see badges below |

Every row also carries a badge for what pressing **Add** will do:

| Badge                   | Meaning                                                                                 | Add does                                                   |
| ----------------------- | --------------------------------------------------------------------------------------- | ---------------------------------------------------------- |
| **Standalone SKU**      | no features                                                                             | one line                                                   |
| **Configurable bundle** | at least one feature still has room to choose (`MinSelections__c` < `MaxSelections__c`) | adds a parent, then opens the Bundle Studio                |
| **Static bundle**       | every feature fully constrained                                                         | adds a parent and its fixed children, nothing to configure |

A bundle row's second line shows its feature/option counts, Pricing Mode and
Header Treatment, or **"no bundle definition"** when the product has none
(the legacy roll-up path).

### Search and the command palette

`Ctrl K` (shown as a `Ctrl K` chip in the search box) opens the **Commands**
palette — a dialog with its own search box (placeholder **"Type a command or
a product name"**) and a grouped command list, each row showing a group
label, the command label, and a keyboard hint. It is the fast path to
anything the toolbar buttons also do, plus jumping straight to adding a named
product.

### Views, Group by, Table/Timeline, Kind, Window

- **Views** are tabs across the header, one per `DD_CPQ_Cart_View__mdt`
  record the running user's permission set and the quote's Transaction Type
  qualify for (see Admin configuration, below). Switching views changes
  columns, grouping, density, row actions and even the cart's Surface — all
  from metadata, no reload of the record.
- **Group by** (the toolbar select, not the Add-products one) groups the
  grid by `none`, `bundle`, `charge`, or `family`, per the view's default or
  the rep's own pick.
- **Table / Timeline** is a segmented control. **Table** is the row grid.
  **Timeline** renders the same lines as the Invoice Timeline — one column
  per billing period, ramp and trial modulation shown as per-period cells —
  with its own **Explain** icon per line opening the identical waterfall
  the table's does.
- **Kind** and **Window** are not cart-wide filters — they are **Deal Block**
  settings (a block's Kind: Section/Phase/Site/Scenario/Other; a block's
  Window: Follow the deal or its own start date + term). See Deal Blocks
  below.

### Bundles — expand/collapse, and the plan picker

A bundle parent row expands to show its children (its options), indented
and tagged with the parent's name. Collapsing a bundle hides its children
but not its totals. A **Configure** row action on the parent re-opens the
Bundle Studio to re-pick options; see `bundles.md` for the Studio itself —
the cart only launches it and receives its saved picks back (options that
match an existing child line are updated in place, not deleted and
re-inserted).

A line whose product carries more than one Active rate plan shows a **plan
picker** inline — picking a different plan reprices that line through the
same preview path as every other edit, nothing is written until Commit.

### Editing a line

Only these fields are ever editable in the grid, because only these
round-trip through the engine's `LineDraft` on every commit:

`Quantity`, `ManualDiscount__c` (reads as a `%` because its API name ends
`Discount__c`), `TermMonths__c`, `BillingFrequency__c`, `EffectiveDate__c`,
`EndDate__c`, `IndependentTerm__c`.

Every other field-set column is **read-only** display. `EndDate__c` is a
formula column: typing a date into it does not write that field — it sets
`TermMonths__c` to match, and the formula recomputes the date shown. A date
that doesn't land on a month boundary snaps down to the whole month below
it, because every pricing stage (term curves, ramps, MinCommitMonths, the
normalized monthly rate) is defined in whole months.

### Line actions

Up to four buttons per row, chosen by the view's **Row Actions** setting
(blank = all four, `none` = none):

| Button                       | Opens                                                          |
| ---------------------------- | -------------------------------------------------------------- |
| **Configure** (bundles only) | the Bundle Studio                                              |
| **Ramp**                     | the ramp editor (Y1/Y2/Y3 overrides)                           |
| **Explain price**            | that line's price waterfall                                    |
| **Remove**                   | drops the line and its children — nothing deletes until Commit |

Selecting several lines (checkboxes) raises a bulk banner:
**"N selected"**, with **Discount %**, **Term**, and **Qty** fields each
with its own **Apply**, a **Move to block…** select when the quote has
blocks, **Remove N** (only if the view grants delete access), and **Clear**.
A bulk discount/term/qty Apply re-previews every selected line in one round
trip.

### Unsaved changes and Commit

Any edit — a quantity, a bundle save, a block move — sets `isDirty` and
shows the **Unsaved changes** flag. **Undo** steps back through edits one
at a time (disabled with nothing to undo). The **Commit** button's own
label (`commitLabel`) and its disabled state follow the engine's own
read — e.g. disabled while a required rate plan is missing, while a floor
block is unresolved, or while there is nothing dirty to write. Commit posts
every pending draft to `/dd/v1/commit` in one call; a failed commit rolls
back to a savepoint server-side, so a partial write never survives (no
QLIs are left half-written).

### Run meter and Rules meter

Two collapsible strips below the toolbar, each remembering its own
open/closed state per user (`saveRunMeterOpen` / `saveRulesMeterOpen`):

- **Run meter** — "what the last price/commit run cost": a kind label
  (preview vs commit), line count, query count, CPU time, a trace reference,
  and a **Failed** or **Over budget** flag when the run tripped the preview
  SOQL budget (35 queries) or failed outright. Exists so an admin debugging
  "why is this quote slow" doesn't need debug logs.
- **Rules meter** — "what every rule did on this run", grouped by matrix,
  each row an outcome pill (matched / shadowed / blocked / applied / not
  applicable) next to the rule's own label. The verified commit above showed
  one evaluated discount rule with 0 matched/applied (correctly — the CT
  quote's account has no qualifying criteria), confirming the meter reports
  real per-rule outcomes, not just totals.

### The price waterfall panel

Click any line's net price, or its **Explain price** row action, to open
`cpqWaterfall`: a dialog titled with the line and the subline **"Every
stage the engine ran, and what each one did"**, an ordered list of stages,
and a **Net** summary pill. Each stage row shows the running value after
that stage and (when the stage changed the number) which rule fired. A
bundle **parent** line's stages read `passthrough` with a note like
_"Bundle parent skips [stage] by design — roll-up bundle, the children carry
the price"_ — confirmed verbatim in the REST bootstrap payload captured
above. A stage that cannot apply to this line's pricing model at all (e.g.
`UsageTierPrice` on a flat PerUnit line) reads `not-applicable` instead.

Installed build 0.1.0-9 waterfall stages, in order: `PricebookList` →
`ContractPrice` → `DerivedListPrice` → `RampAdjustedPrice` →
`TermAdjustedPrice` → `UsageTierPrice` → `PricingModel` → `SystemDiscount`
→ `PromotionDiscount` → `ChannelDiscount` → `VolumeDiscount` →
`ManualDiscount` → `CommitDiscount` → `Net`. (DDCPQ-69, which names the
_rule_ behind each step inline, and DDCPQ-112, which tightens the
discount-% rounding shown, land after this build — the stage list and
values above are what 0.1.0-9 actually returns.)

## Deal Blocks

A **Deal Block** (`DealBlock__c`) is a named grouping of lines that is
_not_ a bundle and _not_ a quote — a free-form bucket for anything a rep
wants to see, price or negotiate as one unit: a renewal's Phase 2, a
multi-site deal's "Austin office", three alternative packages the customer
is choosing between. Blocks can nest (`ParentBlock__c`) and a bundle's
children, every ramp year of a line, and a hybrid's sibling charges always
share their leader's block automatically.

### What a block does to pricing

A block's own **Block discount %** (`AdditionalDiscountPct__c`) is taken
off every rateable line in it, **after** that line's own manual discount
and **before** the margin floor check — shown as its own named step in
each line's waterfall (`CommitDiscount`/block stage), and a sub-block's
discount multiplies with its parent's. Marking a block **Optional**
(`IsOptional__c`) keeps its lines priced and visible but drops them from
the quote total and from acceptance — the way to show a "nice to have"
add-on without it counting yet.

### Kinds and Pick one

`Kind__c` is a label only (`Section` default, `Phase`, `Site`, `Scenario`,
`Other`) — it never changes a price, but Phase and Site drive the
Build-blocks suggestions, the Invoice Timeline, and how renewals rebuild
the block later.

A block can be **Pick one** (`SelectionMode__c = ExactlyOne`): its
sub-blocks become alternatives and only the one with `IsSelected__c = true`
counts toward the total — the mechanism behind "compare these three
packages side by side" (the **compare** drawer opened from the block
settings).

### Building blocks (tester)

The toolbar's **Build blocks ▾** control offers three tabs (`buildVm.tabs`):

1. **New** — type a **Name** (placeholder _"For example, Phase 2 –
   Expansion"_), pick a **Kind**, pick **Inside** (nest it under an
   existing block or leave it top-level), then **Create** or **Create and
   add products** (creates the block, then opens Add products pre-targeted
   at it).
2. **Suggest** — "What each split would build from the lines on this quote,
   with the engine's figures. Nothing changes until you Apply." Each
   suggested way to split (e.g. _by bundle_, _by phase_) shows its blocks
   with per-block line names and values; **Apply** commits that split in
   one action. _No useful split: every line would land in the same block_
   is shown when nothing would change.
3. **Templates** — `DealBlockTemplate__c` records an admin built on the
   **Deal Block Templates** tab (`StructureJson__c`, `IsActive__c`). Picking
   one adds its empty blocks (dates counted from the quote's start); your
   existing lines are not moved. _"No active templates. An admin adds them
   on the Deal Block Templates tab"_ shows when none are Active.

**Add to block** (the row/bulk action) adds selected lines to an existing
block. **Move to block…** (the bulk-selection select) moves ticked lines —
"A bundle moves whole, with its ramp years and sibling charges." **Make
these real blocks** appears once a suggestion has been previewed, to
convert the preview into saved blocks without re-running Suggest.

### Block settings

Each block's **⚙ Block settings** drawer (opened from its header in the
grid) edits: **Name**, **Kind** (radio group — "A label; it never changes
prices. Phase and Site drive suggestions, the timeline and renewals."),
**Inside** (reparent), **Window** — **Follow** (the block starts/ends with
the deal) or **Own window** (its own Starts date + Months; with "One
contract per phase" turned on elsewhere, an own-window block becomes its
own contract), **Block discount %** (placeholder _"None"_), **Optional**
checkbox, **Pick one** checkbox (only shown when the block has sub-blocks,
`canPickOne`), **Location** (_"Site or ship-to"_ — feeds the _By site_
suggestion and renewal rebuilds), and **Description** (_"Text for the
proposal"_). **Delete block…** sits in the drawer's footer next to the
same _"Prices update as you change. Nothing is saved until Commit"_
reminder; **Done** closes it.

### Verified in cpq-pkg

Calling `/dd/v1/blocks` with `{"quoteId": "<CT quote>", "action":
"suggest"}` against the CT quote (bundle qty 10 + options, standalone qty
5, no blocks yet saved) returned a **Phase** split with two proposed
blocks: "Phase 1" (the bundle + its 4 options, ARR $90,000) and "Phase 2"
(the standalone product, ARR $9,000) — matching the quote's totalArr of
$99,000 exactly (90,000 + 9,000). This confirms Suggest computes real,
engine-sourced per-block totals rather than a placeholder.

## Admin configuration of the cart

Everything below is Setup-only — no deploy, no code — and is what a
subscriber admin can change in their own org (see
`docs/LEDGER_CONFIGURATION.md` in the repo for the full writer's version of
this guide).

| What                                           | Where                                                                                                                                                                                                  | Admin can change                                                                                                                                                                                                                                    |
| ---------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Add/remove a column                            | Setup → Object Manager → **Quote Line Item** → **Field Sets** (`DD_CPQ_Cart_Rep`, `DD_CPQ_Cart_DealDesk`, `DD_CPQ_Cart_Finance`, `DD_CPQ_Cart_Delivery`, `DD_CPQ_Cart_Customer`, `DD_CPQ_Cart_Detail`) | Yes — drag any field in; its label, data type, formula/read-only state and FLS all drive the column automatically                                                                                                                                   |
| Add/remove a header field                      | Setup → Object Manager → **Quote** → Field Sets (`DD_CPQ_Header_Default`, and any view-specific Header Field Set)                                                                                      | Yes                                                                                                                                                                                                                                                 |
| Add a computed column (margin, per-seat, etc.) | a new formula field on QLI, dropped into a field set                                                                                                                                                   | Yes                                                                                                                                                                                                                                                 |
| Add/edit a **View**                            | Setup → **Custom Metadata Types** → **DD CPQ Cart View** (`DD_CPQ_Cart_View__mdt`) → Manage                                                                                                            | Yes — Line/Header/Detail Field Set, Group By, Sort, Density, Default Filter, Row Actions, **Surface** (`page`/`popup`), Configurator Mode (`expert`/`guided`), Pack, Personas (Permission Set API names), Transaction Types, Sort Order, Is Default |
| Row stripe / "needs attention" rule            | **DD CPQ Cart Rule** (`DD_CPQ_Cart_Rule__mdt`): Field, Operator (`gt lt eq ne`), Value, Tone (`warn`/`crit`), optional Views/Pack scoping                                                              | Yes. Shipped: `BelowMarginFloor` (crit), `ManualDiscountOver15` (warn)                                                                                                                                                                              |
| Rename a column for a vertical                 | **DD CPQ Cart Label** (`DD_CPQ_Cart_Label__mdt`), scoped to a View's Pack                                                                                                                              | Yes. Shipped example: `NMR` label record                                                                                                                                                                                                            |
| Where Configure Products opens                 | the View's **Surface** field (above)                                                                                                                                                                   | Yes                                                                                                                                                                                                                                                 |
| Row action buttons                             | the View's **Row Actions** field — `configure;ramp;explain;remove`, blank = all four, `none` = none                                                                                                    | Yes                                                                                                                                                                                                                                                 |
| Build a Deal Block Template                    | **Deal Block Templates** tab → `DealBlockTemplate__c` (Name, Description, Structure JSON, Is Active)                                                                                                   | Yes                                                                                                                                                                                                                                                 |
| Placing the cart component                     | Lightning App Builder, the `cpqLedger` component's **View** property (a picklist of View records)                                                                                                      | Yes — two placements of the same component can show two different Views                                                                                                                                                                             |

What an admin **cannot** change without code: which fields round-trip to
the engine on Commit (the fixed inline-editable list above — adding one
means teaching `CommitService.LineDraft` a new field), the waterfall's
stage list, or the pipeline order (CLAUDE.md "Engine pipeline" section).

## Objects and fields

| Object label (API name, unprefixed)          | Field                                                                                                                   | Meaning / values                                                                                          |
| -------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------- |
| Deal Block (`DealBlock__c`)                  | `Quote__c`                                                                                                              | the quote this block belongs to (required)                                                                |
|                                              | `Kind__c`                                                                                                               | `Section` (default) / `Phase` / `Site` / `Scenario` / `Other` — a label only                              |
|                                              | `SelectionMode__c`                                                                                                      | `All` (every sub-block counts) / `ExactlyOne` ("Pick one" — only the selected alternative counts)         |
|                                              | `IsSelected__c`                                                                                                         | on a sub-block of a Pick-one block: the chosen alternative                                                |
|                                              | `IsOptional__c`                                                                                                         | priced and shown, excluded from total and acceptance                                                      |
|                                              | `AdditionalDiscountPct__c`                                                                                              | Block discount %, applied after line discounts, before the margin floor; multiplies with a parent block's |
|                                              | `ParentBlock__c`                                                                                                        | nesting — a sub-block's container                                                                         |
|                                              | `BlockKey__c`                                                                                                           | stable identity that survives renewal/amendment copies; minted once, never edited                         |
|                                              | `Location__c`                                                                                                           | site/ship-to text; feeds the _By site_ suggestion and renewal rebuilds                                    |
|                                              | `StartDate__c` / `TermMonths__c` / `EndDate__c`                                                                         | the block's own window, used only when Window = Own window                                                |
|                                              | `SortOrder__c`                                                                                                          | display order among sibling blocks                                                                        |
|                                              | `ARR__c` / `MRR__c` / `NetTotal__c` / `ContractValue__c` / `OneTimeTotal__c` / `LineCount__c`                           | computed roll-ups shown in block headers and the deal map                                                 |
| Deal Block Template (`DealBlockTemplate__c`) | `StructureJson__c`                                                                                                      | the empty-block tree a template stamps onto a quote                                                       |
|                                              | `IsActive__c`                                                                                                           | only Active templates appear in the Build-blocks → Templates tab                                          |
|                                              | `Description__c`                                                                                                        | shown next to the template name in the picker                                                             |
| Quote Line Item (standard)                   | `DealBlock__c`                                                                                                          | the block this line belongs to; blank = ungrouped                                                         |
|                                              | `BlockDiscount__c`                                                                                                      | effective % applied by the block-discount stage (block + its parent combined)                             |
|                                              | `BlockSortOrder__c`                                                                                                     | position inside the block, lowest first                                                                   |
|                                              | `BlockedByFloor__c`                                                                                                     | set by `MarginFloorService` when a Block-action margin floor rule fires on this line                      |
| Catalog (`Catalog__c`)                       | `Status__c`                                                                                                             | only `Active` catalogs appear in the Add-products drawer's Catalog dropdown                               |
|                                              | `Description__c`                                                                                                        | shown in the picker                                                                                       |
| Catalog Product (`CatalogProduct__c`)        | `Catalog__c` / `Product__c`                                                                                             | the junction — which products belong to which catalog                                                     |
|                                              | `Featured__c`                                                                                                           | shows a **featured** tag on the product tile                                                              |
|                                              | `DisplayOrder__c`                                                                                                       | tile ordering within the catalog                                                                          |
| DD CPQ Cart View (`DD_CPQ_Cart_View__mdt`)   | `LineFieldSet__c` (required)                                                                                            | the grid's columns                                                                                        |
|                                              | `HeaderFieldSet__c`, `HeaderKpis__c`, `DetailFieldSet__c`                                                               | header strip fields; KPI keys `MRR;ARR;OneTime;LineCount;Attention`; line-drawer Details tab              |
|                                              | `GroupBy__c`                                                                                                            | `none` / `bundle` / `charge` / `family`                                                                   |
|                                              | `Density__c`                                                                                                            | `comfortable` / `compact`                                                                                 |
|                                              | `DefaultFilter__c`                                                                                                      | `none` / `attention`                                                                                      |
|                                              | `RowActions__c`                                                                                                         | `configure;ramp;explain;remove`, blank = all, `none` = none                                               |
|                                              | `Surface__c`                                                                                                            | `page` / `popup`, blank = page                                                                            |
|                                              | `ConfiguratorMode__c`                                                                                                   | `expert` / `guided`, blank = expert                                                                       |
|                                              | `Personas__c`                                                                                                           | semicolon-separated Permission Set API names; blank = everyone                                            |
|                                              | `TransactionTypes__c`                                                                                                   | semicolon-separated Quote Transaction Types; blank = every type                                           |
|                                              | `SortOrder__c` / `IsDefault__c`                                                                                         | switcher order; which view opens with no `viewName`                                                       |
| DD CPQ Cart Rule (`DD_CPQ_Cart_Rule__mdt`)   | `Field__c`, `Operator__c` (`gt lt eq ne`), `Value__c`, `Tone__c` (`warn`/`crit`), `Views__c`, `Pack__c`, `SortOrder__c` | row stripe + Needs-attention filter + Attention KPI                                                       |
| DD CPQ Cart Label (`DD_CPQ_Cart_Label__mdt`) | `FieldApiName__c`, `Label__c`, `Pack__c`                                                                                | renames a column header when the active view's Pack matches                                               |

## Build it (tester)

This assumes the golden path is done (products CRM Suite Pro bundle +
options, Compliance Add-on, Account "Acme Corp", quote "Q-0042 Acme" all
exist and are committed — see `golden-path.md`). To exercise Deal Blocks
and the richer cart surface on top of that:

1. Open **Q-0042 Acme** → **Configure Products**.
2. Toolbar → **Build blocks ▾** → **New**. Name it `Expansion` (anything —
   this is your own cart exploration, not graded data). Kind **Phase**.
   Inside: top-level. Click **Create and add products**.
3. In the Add products drawer that opens, add **Compliance Add-on**,
   quantity 25. It lands inside the new block.
4. Back in the grid, tick the **CRM Suite Pro** bundle row (and nothing
   else) → the bulk banner appears → **Move to block…** → pick
   `Expansion`. Confirm: the bundle _and all four of its options_ move
   together (their rows now sit under the `Expansion` header) — this is
   the "a bundle moves whole" rule.
5. Click the `Expansion` block's **⚙** (block settings). Set **Block
   discount %** to `10`. Click any line now inside it → **Explain price**
   → confirm a block-discount step appears in the waterfall, after the
   line's own system/volume discounts.
6. Toggle **Table / Timeline** to **Timeline** — confirm the same lines
   render as billing-period columns. Toggle back.
7. Press **Ctrl K** in the search box → confirm the Commands palette
   opens with a list of commands/products, grouped.
8. **Commit.** Confirm **Unsaved changes** disappears and the block
   discount is reflected in the committed lines' `BlockDiscount__c`.

## Check it (Claude)

```bash
sf data query -o dd-e2e -q "SELECT Id, DDCPQ__Kind__c, DDCPQ__SelectionMode__c, DDCPQ__AdditionalDiscountPct__c, DDCPQ__IsOptional__c, DDCPQ__ARR__c FROM DDCPQ__DealBlock__c WHERE DDCPQ__Quote__c = '<quote Id>'"
sf data query -o dd-e2e -q "SELECT Product2.Name, Quantity, DDCPQ__DealBlock__c, DDCPQ__BlockDiscount__c, DDCPQ__BlockSortOrder__c, DDCPQ__NetPrice__c FROM QuoteLineItem WHERE Quote.Name = 'Q-0042 Acme'"
sf data query -o dd-e2e -q "SELECT DeveloperName, DDCPQ__Surface__c, DDCPQ__RowActions__c, DDCPQ__GroupBy__c, DDCPQ__IsDefault__c FROM DDCPQ__DD_CPQ_Cart_View__mdt"
sf data query -o dd-e2e -q "SELECT DeveloperName, DDCPQ__Field__c, DDCPQ__Operator__c, DDCPQ__Value__c, DDCPQ__Tone__c FROM DDCPQ__DD_CPQ_Cart_Rule__mdt"
```

Expect every line that moved into a block to share one `DDCPQ__DealBlock__c`
Id (the bundle parent and all four options), a `DDCPQ__BlockDiscount__c` of
10 on each, and the two shipped `DD_CPQ_Cart_Rule__mdt` rows
(`ManualDiscountOver15`, `BelowMarginFloor`).

A read-only price preview (never commit — rule in SKILL.md) to see a
block's effect on a _hypothetical_ line without touching the saved cart:

```bash
sf api request rest "services/apexrest/DDCPQ/dd/v1/price?includeWaterfall=true" -o dd-e2e --method POST --body @body.json
```

using the same selections shape as the golden path's preview body.

## Expected numbers

From the verification run in cpq-pkg (CT quote: CRM Suite Pro bundle qty
10 + 4 options @ $150 each, Compliance Add-on qty 5 @ $150, no discounts
configured on CT's account):

| Measure                                                | Value                                        |
| ------------------------------------------------------ | -------------------------------------------- |
| totalArr (bootstrap)                                   | $99,000                                      |
| totalTcv (bootstrap)                                   | $99,000                                      |
| CRM Suite Pro (parent) netPrice                        | $150 (passthrough — bundle parent)           |
| CRM Suite Pro totalArr (qty 10 × $150 × 12)            | $18,000                                      |
| Suggested "Phase" split — Phase 1 (bundle + 4 options) | ARR $90,000                                  |
| Suggested "Phase" split — Phase 2 (Compliance Add-on)  | ARR $9,000                                   |
| Phase 1 + Phase 2                                      | $99,000 — matches bootstrap totalArr exactly |
| `/dd/v1/commit` on the 6-line selection                | `linesInserted: 6`, no errors                |

The golden path's own worked numbers (Sales Cloud net $101.83, etc.) are
the canonical pricing proof — see `golden-path.md`; this module validates
the cart's _data layer_ (bootstrap, blocks, waterfall shape) around that
same math, not a second pricing scenario.

## Troubleshooting

| Symptom                                                                | Cause                                                                                                                                    | Fix                                                                                                                                                                                    |
| ---------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Configure Products does nothing / no tab opens                         | the active View's Surface could not be read, or no `DD CPQ Cart` tab visibility                                                          | confirm the user's permission set grants the Cart tab; the launcher falls back to the popup modal if the surface call fails, so a total blank screen points at a deeper access problem |
| Opening the Cart tab directly shows "Open a quote" instead of the cart | the tab was opened with no `c__quoteId` in page state                                                                                    | pick the quote from the recent-quotes list, or re-open via Configure Products from the record                                                                                          |
| A column you added to the field set never shows                        | Field-Level Security hides it from the running user — "Nothing announces it; that is the point for customer-facing views"                | grant FLS read access, or confirm that's the intended behaviour for that persona                                                                                                       |
| Moving a bundle's option alone, not the whole bundle                   | not possible by design — ticking any child still moves the whole bundle: "a bundle moves whole, with its ramp years and sibling charges" | select and move the parent; options cannot be split from their bundle into a different block                                                                                           |
| Commit refused after a block discount was set                          | a line in the block tripped the margin floor (`BlockedByFloor__c` true)                                                                  | open that line's waterfall; margin floor rules run after the block discount stage                                                                                                      |
| `/dd/v1/cart` bootstrap returns 403                                    | `DdCpqAccess.AccessDeniedException` — the calling user lacks the permission set this feature checks                                      | assign `DDCPQ__DD_CPQ_Engine_Admin` or `DDCPQ__DD_CPQ_Engine_User`                                                                                                                     |
| `/dd/v1/cart` bootstrap returns a generic 500 with a `type` field      | an unhandled server exception — read the `error` and `type` values verbatim, they are the real Apex exception message and class          | report the exact message; do not retry blindly                                                                                                                                         |
| Catalog dropdown never appears in Add products                         | no `Catalog__c` record is `Active`                                                                                                       | an admin activates at least one catalog, or this org intentionally has none                                                                                                            |
| A rep sees fewer products in Add products than expected                | the Eligibility Matrix or the Opportunity's price book is filtering them, same as the engine would                                       | check both — see `eligibility.md`                                                                                                                                                      |

## Limits and gotchas

- The grid has **no virtualization** by design — the ceiling the component
  itself documents is **2,000 lines**, with bundle-aware pagination at 100
  rows per page (a parent and its children never split across a page).
- Separately, every `/dd/v1/price` and `/dd/v1/commit` **REST** request is
  capped at **1,000 lines per request** by default
  (`DdCpqLimits.maxLinesPerRequest()`, from the `EngineLimits__mdt` custom
  metadata record `Default` → `MaxLinesPerRequest__c`). Exceeding it returns
  HTTP **413** with: _"<operation>: N lines exceeds the limit of 1000. Split
  the request into smaller batches. An administrator can raise the limit on
  the Engine Limits custom metadata record, but the platform enforces a hard
  ceiling of its own that no setting can move."_ An admin can raise this
  metadata value; the cart's own UI in this build does not yet surface that
  ceiling as a setting the rep sees, and there is **no background-commit
  mode** for an oversized cart — that UI (an Automatic/Wait/Background
  choice) and any cart-side "Max lines per request" admin control are
  **DDCPQ-67, not in this build. Arrives in the next build.**
- DDCPQ-69 (naming the rule behind each waterfall step) and DDCPQ-112
  (tighter discount-% rounding in the waterfall) are also **not in this
  build** — the waterfall shows stage values and passthrough notes only, no
  inline rule name.
- A `Pick one` block's comparison view only activates once the block has
  sub-blocks — `canPickOne` is false on a flat block even if you check the
  box in settings first; build the sub-blocks before expecting the compare
  drawer.
- Only `Quantity`, `ManualDiscount__c`, `TermMonths__c`,
  `BillingFrequency__c`, `EffectiveDate__c`, `EndDate__c`,
  `IndependentTerm__c` are ever written back on Commit. Any other field
  dropped into a field set is for display only, however editable it looks
  in Setup.
- `EndDate__c` is a formula; typing into it converts to a term write, not a
  date write, and a non-month-boundary date silently snaps down.

## Questions testers ask

**Why did my new column not round-trip to the engine when I edited it?**
Only the fixed inline-editable field list does. Everything else in a field
set is read-only in the grid, on purpose — `CommitService` would need
teaching before a new field could be written.

**Why does my bundle's option show a price as if it were standalone?**
It isn't standalone — bundle parent rows intentionally skip pricing stages
(passthrough) and show the **parent's own** price; check the child rows for
each option's real price.

**Can I give a customer-facing rep the DealDesk view?**
Only if their permission set is listed in that view's `Personas__c`; blank
means every persona can see it.

**Why can't I move just one option out of a bundle into its own block?**
Bundles move as a unit with their options, ramp years and hybrid siblings —
there's no cart action to split one option into a different block.

**Is there a hard limit on quote size?**
Two different ones: the grid's own documented ceiling (2,000 lines,
paginated) and the REST request ceiling (1,000 lines per call by default,
raisable in Engine Limits custom metadata). DDCPQ-67's dedicated "Max lines
per request" setting and background commit are not installed yet.

**Does the Block discount stack with the line's own manual discount?**
Yes — the block discount is taken after the line discount, shown as its own
waterfall step, and a sub-block's discount multiplies with its parent
block's.

**Why did Suggest show "no useful split"?**
Every line on the quote would land in the same bucket for that split (e.g.
everything is one bundle, so a "by bundle" split has nothing to separate).

**Does Timeline show different numbers than Table?**
No — both read the same priced lines; Timeline only changes the layout
(per-period columns) and its own Explain icon opens the identical
waterfall.
