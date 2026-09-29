---
name: dd-cpq-ui-builder
description: Scaffold a custom Quote / Cart / Configurator UI on top of the DeepDive CPQ public API, or configure the shipped schema-driven cart (cpqLedger) instead of building one. Use when a customer says "build me a healthcare quote UI", "make a mobile ordering app on DD CPQ", "generate a React storefront that calls DD", or asks for any custom UI that consumes the /dd/v1 REST surface. Ships with the DD CPQ public API surface documented inline so you don't need to guess endpoint shapes.
---

# DD CPQ UI Builder Skill

You are helping a customer build a **custom UI** on top of DeepDive CPQ. DD ships a
default React admin + LWCs, but the whole engine is exposed as a public REST API so
customers can build anything &mdash; a healthcare-vertical quote wizard, a self-serve
storefront, a mobile field-sales app, a Slack bot, whatever the deal needs.

Your job: understand what they want, pick the right endpoints, generate the UI code,
and hand it back with a working end-to-end sample they can iterate on.

## Golden rules

1. **Never build pricing logic in the UI.** All pricing math (list price, system
   discounts, volume discounts, waterfall, margin floor) MUST come from
   `/dd/v1/price` and `/dd/v1/reprice`. The DD engine is authoritative; the UI is
   a client. Same rule as `CLAUDE.md` #2 in the DD repo.
2. **Only `/dd/v1/commit` writes QuoteLineItems.** Never POST directly to the
   standard Salesforce QLI endpoint &mdash; that bypasses the engine (skips
   pricing, discounts, floor, constraints, audit log).
3. **Use the exact schemas from `docs/DD_CPQ_API_REFERENCE.html`.** Don't invent
   field names. The `LineDraft` shape (localKey / parentLocalKey / productId /
   quantity / pricebookEntryId / manualDiscountPct / billingFrequency / termMonths)
   is canonical across `/price`, `/commit` and `/cart`.
4. **No AI in the quoting runtime.** Runtime AI was cut on 2026-09-13 with
   `/prompt-cart`. If a customer wants natural-language quoting, that is an MCP
   client (Claude, or any agent) driving the same `dd_cpq_*` tools a rep's UI
   uses &mdash; not an LLM call inside the cart.
5. **Salesforce Ids are 15 or 18 chars.** Never truncate them. Never generate
   fake ones for testing &mdash; use real ones from `/dd/v1/catalog?quoteId=...`.

## The DD CPQ API in one page

**Base URL:** `{instanceUrl}/services/apexrest/dd/v1`
**Auth:** `Authorization: Bearer {salesforceAccessToken}` (see `docs/DD_CPQ_AUTH_GUIDE.html`)

### Discovery

- `GET /catalog?quoteId={id}` &rarr; products eligible for this Quote (with `pricebookEntryId`, `listPrice`, `isBundle`)
- `GET /catalogs` &rarr; curated product groupings
- `GET /pricebooks` &rarr; available Pricebooks

### Configurator

- `GET /configure/{productId}` &rarr; bundle structure: features + options + option constraints (requires / excludes)
- `POST /compatibility` &rarr; live check for selected option combinations

### Pricing (called on every UI interaction, live)

- `POST /price` &rarr; run the full pricing pipeline on drafts, return net prices (NO DML)
- `POST /price?includeWaterfall=true` &rarr; adds 6-stage waterfall per line with rule-id citations
- `POST /reprice` &rarr; recompute prices for committed lines after data changes
- `POST /discount` &rarr; apply / preview a manual discount
- `POST /pvm` &rarr; Price / Volume / Matrix preview for a single product

### Cart operations

- `POST /commit` &rarr; **only endpoint that writes QLIs.** Runs full pipeline + inserts.
- `POST /cart` with `{quoteId}` &rarr; bootstrap: the priced lines, totals, account context
- `POST /cart` with `{action: "schema", viewName}` &rarr; what the schema-driven cart renders FROM:
  columns (from a Field Set on QuoteLineItem, FLS-filtered, typed off the describe), header
  fields (a Field Set on Quote) + computed KPI keys, rules, label overrides, the views this user may open
- `POST /cart` with `{action: "ledger", quoteId, viewName}` &rarr; bootstrap + schema + stored
  field-set values in one call — what `cpqLedger` loads. Build a custom grid off this, not off SOQL.

Cut 2026-09-13 and **not in this package** — do not call or build against: `/check-laws`,
`/deal-score`, `/deal-sculptor`, `/prompt-cart`, `/recommend-deals`, `/agreement`, `/amendment`,
`/agreement-studio`.

### Admin (usually not customer-facing)

- `POST /admin` &rarr; CRUD for bundles, catalogs, variables, matrices

### Customizing the shipped cart instead of building one

Before writing a grid, check whether `cpqLedger` already does it: a column is a field in a
Field Set, a persona is a `DD_CPQ_Cart_View__mdt` record, a row stripe is a
`DD_CPQ_Cart_Rule__mdt`, a vertical's vocabulary is `DD_CPQ_Cart_Label__mdt` — none of it is
code. See `docs/LEDGER_CONFIGURATION.md`. Build custom only when the interaction model differs,
not when the columns do.

## The three UI archetypes

99% of custom builds fall into one of these three shapes. Pick the closest match, then
customize.

### Archetype 1: Quote Hero (top-of-record summary card)

**When to build:** customer wants a compact card on the Quote record page showing
totals, KPIs, lines needing attention, and a "Launch Configurator" CTA.

**Endpoints:**

- `POST /cart {quoteId}` &rarr; priced lines, totalArr, totalTcv, account context
- `POST /cart {action: "schema"}` &rarr; the rules that decide "needs attention",
  so the card and the ledger agree on the count

**Data flow:**

```
onLoad(quoteId):
  1. POST /cart {quoteId}                     # committed lines, already priced
  2. POST /cart {action: "schema"}            # rules + header KPI keys
  3. Evaluate rules client-side (gt/lt/eq/ne on line values — display, not pricing)
  4. Render KPI grid + line summary + CTAs
```

### Archetype 2: Interactive Configurator (bundle picker)

**When to build:** customer wants a step-by-step wizard where reps pick a bundle,
configure its options, watch prices update live, and commit. This is
`force-app/main/default/lwc/cpqBundleStudio` (the Bundle Studio: feature rail,
guided / expert mode, engine-priced options via a hypothetical-cart preview,
constraint reasons with one-click fixes, presets, undo). `cpqBundleConfigurator`
is the older drawer it replaced; only the unplaced `cpqCart` still uses it.

**Endpoints:**

- `GET /configure/{productId}` &rarr; bundle structure (once per bundle open)
- `POST /price` (on every quantity/option change) &rarr; live prices
- `POST /compatibility` (on option toggle) &rarr; violations
- `POST /commit` (once at end) &rarr; persist as QLIs

**Data flow:**

```
onProductPick(productId):
  1. GET /configure/{productId}
  2. Initialize draft state: parent line + default-selected options as children
  3. onOptionChange:
       POST /compatibility {quoteId, selectedOptionIds}   # violations?
       POST /price {quoteId, drafts}                      # live prices
       re-render
  4. onSave: POST /commit {quoteId, drafts}
```

### Archetype 3: Schema-driven Ledger (a grid the customer configures, not codes)

**When to build:** almost never — it ships as `force-app/main/default/lwc/cpqLedger`. What a
customer asks for is usually a column, a persona view, a row-stripe rule or a vocabulary
("MRR reads as MRC"), and every one of those is a Field Set or a `DD_CPQ_Cart_*` custom
metadata record. Walk them through `docs/LEDGER_CONFIGURATION.md` first.

**Endpoints, if a genuinely different grid is needed:**

- `POST /cart {action: "schema", viewName}` &rarr; columns typed off the describe, FLS-filtered
- `POST /cart {action: "ledger", quoteId, viewName}` &rarr; schema + priced lines + stored values
- Edits &rarr; `POST /price` to reprice, then `POST /commit` with `mode: "reconcile"` — and
  send **every** line back, because reconcile deletes what it does not see.

Two former archetypes — the Prompt-First Cart and the Agreement Generator — were cut on
2026-09-13 with the runtime-AI and Agreement Studio features. Do not offer them.

## Sample: minimal React Quote Hero (Archetype 1)

Give this to the customer as a starter. It's 100 lines, plain React, no dependencies
beyond `fetch`. They copy, customize their branding, ship.

```tsx
import { useEffect, useState } from "react";

// --- Config ---
const API_BASE =
  import.meta.env.VITE_SF_INSTANCE_URL + "/services/apexrest/dd/v1";
const TOKEN = () => localStorage.getItem("sf_token"); // set by your OAuth callback

// --- Types (subset of docs/DD_CPQ_API_REFERENCE.html) ---
interface CartLine {
  localKey: string;
  qliId: string | null;
  productName: string;
  quantity: number;
  netPrice: number;
  totalArr: number | null;
  manualDiscount: number;
  blockedByFloor: boolean;
  [field: string]: unknown;
}
interface CartRule {
  field: string; // bare api name, e.g. ManualDiscount__c
  operator: "gt" | "lt" | "eq" | "ne";
  value: string;
  tone: "warn" | "crit";
}

// --- Tiny client ---
async function ddPost<T>(path: string, body: object): Promise<T> {
  const res = await fetch(`${API_BASE}${path}`, {
    method: "POST",
    headers: {
      Authorization: `Bearer ${TOKEN()}`,
      "Content-Type": "application/json"
    },
    body: JSON.stringify(body)
  });
  if (!res.ok) throw new Error(`${path} ${res.status}: ${await res.text()}`);
  return res.json();
}
async function ddGet<T>(path: string): Promise<T> {
  const res = await fetch(`${API_BASE}${path}`, {
    headers: { Authorization: `Bearer ${TOKEN()}` }
  });
  if (!res.ok) throw new Error(`${path} ${res.status}: ${await res.text()}`);
  return res.json();
}

// The same LIVE map cpqLedgerUtil uses: a rule names a field, the cart
// line carries the engine's value for it under a camelCase key.
const LIVE: Record<string, keyof CartLine> = {
  ManualDiscount__c: "manualDiscount",
  BlockedByFloor__c: "blockedByFloor",
  NetPrice__c: "netPrice",
  Quantity: "quantity"
};
function fires(rule: CartRule, line: CartLine): boolean {
  const raw = line[LIVE[rule.field] ?? rule.field];
  if (raw === null || raw === undefined) return false;
  const n = Number(raw),
    v = Number(rule.value);
  const numeric =
    !Number.isNaN(n) && !Number.isNaN(v) && typeof raw !== "boolean";
  switch (rule.operator) {
    case "gt":
      return numeric && n > v;
    case "lt":
      return numeric && n < v;
    case "eq":
      return numeric
        ? n === v
        : String(raw).toLowerCase() === rule.value.toLowerCase();
    case "ne":
      return numeric
        ? n !== v
        : String(raw).toLowerCase() !== rule.value.toLowerCase();
  }
}

// --- Component ---
export function MyQuoteHero({ quoteId }: { quoteId: string }) {
  const [arr, setArr] = useState<number | null>(null);
  const [lines, setLines] = useState<CartLine[]>([]);
  const [attention, setAttention] = useState(0);
  const [loading, setLoading] = useState(true);

  useEffect(() => {
    (async () => {
      // The cart bootstrap is already priced by the engine - no /price call,
      // no SOQL, no client-side math. The schema's rules decide "attention"
      // the same way the shipped ledger does, so both surfaces agree.
      const [cart, schema] = await Promise.all([
        ddPost<{ lines: CartLine[]; totalArr: number }>("/cart", { quoteId }),
        ddPost<{ rules: CartRule[] }>("/cart", { action: "schema" })
      ]);
      setArr(cart.totalArr);
      setLines(cart.lines);
      setAttention(
        cart.lines.filter((l) => schema.rules.some((r) => fires(r, l))).length
      );
      setLoading(false);
    })();
  }, [quoteId]);

  if (loading) return <div>Loading…</div>;
  return (
    <div className="quote-hero">
      <h1>Quote {quoteId}</h1>
      <div className="kpi-grid">
        <Kpi label="ARR" value={`$${arr?.toLocaleString()}`} />
        <Kpi label="MRR" value={`$${((arr ?? 0) / 12).toLocaleString()}`} />
        <Kpi label="Lines" value={String(lines.length)} />
        <Kpi
          label="Needs attention"
          value={String(attention)}
          tone={attention ? "warn" : ""}
        />
      </div>
      <button
        onClick={() => (location.href = `/configurator?quoteId=${quoteId}`)}
      >
        Configure products
      </button>
    </div>
  );
}
```

Give the customer this file plus their CSS. That's a working custom Quote Hero on
DD CPQ in under 120 lines, and every number on it came from the engine.

## When the customer asks for something exotic

- **Mobile / offline-first?** Use the same endpoints; cache `/catalog` responses locally, batch `/price` calls, defer `/commit` until online.
- **Slack bot / voice / "just let me type it"?** That is an agent, not a UI: point Claude (or any MCP client) at the `dd-cpq` MCP server and the `dd-cpq-quoting-agent` skill. The tools are the same ones the LWCs call.
- **Multi-quote comparison UI?** Loop `/cart {quoteId}` per quote, diff `totalArr` / `totalTcv` and the per-line `waterfallJson`.
- **White-label partner portal?** Route their API calls through a proxy that injects a per-partner run-as user token; scope permissions via permsets. Field-level security then decides which ledger columns the partner even sees.
- **Approval workflow?** The engine flags `blockedByFloor` per line and the schema's `DD_CPQ_Cart_Rule__mdt` records mark `crit` rows; gate `/commit` on those and use the standard Salesforce approval process on the Quote.

## What NOT to build

- **A pricing calculator.** Call `/price`. It's a solved problem.
- **A discount-stacking engine.** Same.
- **A conditional rules DSL.** Row-level rules are `DD_CPQ_Cart_Rule__mdt`; eligibility / compatibility / pricing rules are the matrices behind `/admin`, all evaluated by `CriteriaEvaluator`.
- **A PDF renderer.** Cut with Agreement Studio on 2026-09-13. Salesforce's own Quote PDF templates, or the customer's document tool, own this.
- **A grid with hard-coded columns.** `cpqLedger` reads its columns from a Field Set. If the ask is "add a column", it is a five-minute admin task, not a build.

## Files to reference

- **API contract:** `docs/DD_CPQ_API_REFERENCE.html`
- **Auth setup:** `docs/DD_CPQ_AUTH_GUIDE.html`
- **Configuring the shipped cart:** `docs/LEDGER_CONFIGURATION.md`
- **Reference implementations to copy from:**
  - Schema-driven cart: `force-app/main/default/lwc/cpqLedger/` (+ `cpqLedgerUtil` for the pure helpers and their Jest tests)
  - Bundle Studio (configurator): `force-app/main/default/lwc/cpqBundleStudio/` (+ its Jest tests, which mount it for real)
  - Bundle Builder (admin): `force-app/main/default/lwc/cpqBundleBuilder/`
  - Rate Plan Editor (admin): `force-app/main/default/lwc/cpqRatePlanEditor/`

- **The engine principles (must-obey):**
  - `CLAUDE.md` at repo root &mdash; especially rules #1 (no `DD__` prefix), #2 (engine is authoritative), #3 (no AI in runtime), #6 (only QLI extended)
  - `feedback_no_qli_triggers.md` in memory &mdash; every QLI write must route through `/dd/v1/commit`

## Delivery checklist

Before handing the built UI back to the customer, verify:

- [ ] All prices come from `/dd/v1/price` or the `/cart` bootstrap, not client-side math
- [ ] Commits happen via `/dd/v1/commit`, not standard SF QLI POST — and reconcile mode sends every line back
- [ ] "Needs attention" comes from the schema's rules and the engine's `blockedByFloor`, not from thresholds typed into the UI
- [ ] Columns the customer asked for were tried as a Field Set change before any code was written
- [ ] Auth uses Bearer token from Connected App OAuth (not embedded credentials)
- [ ] `LineDraft` shape matches `docs/DD_CPQ_API_REFERENCE.html` exactly (no invented fields)
- [ ] Sample renders end-to-end against `cpq-dev` before delivery
