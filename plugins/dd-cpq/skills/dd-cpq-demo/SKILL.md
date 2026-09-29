---
name: dd-cpq-demo
description: >-
  Run stunning live demos of DeepDive CPQ (DD CPQ) in Claude Cowork —
  Wow #1 (live pricing waterfall visualization), Wow #2 (Ask the
  Warehouse scrollytelling), Wow #3 (Amendments Lifecycle time-travel),
  Wow #4 (Constitutional Auditor), Wow #5 (Deal Risk X-Ray), Wow #6
  (voice-driven configurator + autonomous migration). Use ANY time the
  user says "demo", "show me", "make this visual", "wow moment",
  "presentation", "prospect meeting", "amendment demo", "renewal demo",
  or asks to render pricing / rules / warehouse output as a chart or
  dashboard. Skip for regular dev work — CLAUDE.md carries the build
  rules and loads automatically.
---

# DD CPQ Demo Choreography

> This skill orchestrates the `dd_cpq_*` MCP tools + Cowork's browser
> panel to produce "wow moments" during live demos. Every recipe is
> designed to fit inside a single Cowork message + follow-up render.

## When to use

- User says "show me" / "demo" / "wow" / "make it visual" / "render"
- User is preparing a customer / prospect / investor meeting
- User asks a "why" question about pricing, rules, or migration provenance
  → follow up with Wow #2 (warehouse story) rendered as HTML
- User asks about a specific quote's pricing → follow up with Wow #1
  (live waterfall) rendered as HTML

**Skip this skill** when the user is doing regular dev. There is no
parent skill any more: `CLAUDE.md` carries the hard rules, engine
pipeline and golden test, and loads on its own every session.

## Prerequisites (do these once)

1. **MCP server v0.10.0+** — has `dd_cpq_ask_warehouse` + all `dd_cpq_*` tools
2. **Salesforce sandbox connected** — verify with `dd_cpq_list_orgs`;
   `activeAlias` must not be null. Otherwise call `dd_cpq_connect_org`
3. **For Wow #2 only**: the Python migrator FastAPI service must be running:
   ```powershell
   cd migrator
   migrator serve --host 127.0.0.1 --port 8000
   ```
   And `MIGRATOR_API_URL` + `MIGRATOR_API_TOKEN` must be set as env vars
   in `claude_desktop_config.json` under `mcpServers.dd-cpq.env`

## The five wow moments — recipes

Each recipe is a self-contained sequence. Follow it verbatim during a
demo — do NOT improvise the tool order or the rendering shape.

---

### 🎬 Wow #1 — Live Pricing Waterfall

**Trigger:** user says _"explain the price"_, _"show the waterfall"_,
_"why is this line $X?"_, _"show me the pricing decomposition"_.

**Beats (target <15 seconds total):**

1. Call `dd_cpq_find_quote` if the user gave a name/number instead of `0Q0...` id
2. Call `dd_cpq_explain_price` with the resolved quote id — full 11-stage decomposition returned
3. Generate a self-contained HTML file at `demo/waterfall-{quoteId}.html`
   using the template at `templates/waterfall.html` — substitute the
   `LINE_DATA_JSON` placeholder with the actual explain_price output as
   pretty-printed JSON, and the `QUOTE_ID` placeholder with the id
4. Open the HTML file in Cowork's browser panel via the Files → open action
5. Narrate: _"Here's how the $130 pricebook rate became $101.83 per user
   per month — every one of those green bars is a PolicyLaw that fired,
   click any bar to see the source."_

**What the audience sees:** Sankey-style bridge chart. Each column is a
pricing stage (PricebookList → ContractPrice → DerivedListPrice →
RampAdjustedPrice → UsageTierPrice → SystemDiscount → PromotionDiscount →
ChannelDiscount → VolumeDiscount → ManualDiscount → Net). Bars animate
in left-to-right. Hovering a bar shows the applied PolicyLaw name +
`Statement__c`.

**Failure modes to catch:**

- explain_price returns fewer than 6 stages → the org is on engine v1;
  the template auto-collapses missing stages, do NOT modify the template
- Line has manualDiscount=0 AND no promo → still render the stage, mark it
  "no rule fired" in grey (this is a demo asset — completeness > minimalism)

---

### 🎬 Wow #2 — Ask the Warehouse (scrollytelling)

**Trigger:** user asks a WHY question — _"why do healthcare renewals
get 3% uplift?"_, _"who decided the seat threshold?"_, _"what was the
rationale for the Gold-tier exception?"_ Also fires when RevOps says
_"show me the story behind this rule"_.

**Beats (target <30 seconds total for retrieval + render):**

1. If the user didn't give a customer slug, ask ONCE which customer
   (the warehouse is scoped per-customer). Default guess: use the
   `active-org.json` alias.
2. Call `dd_cpq_ask_warehouse` with the query, customer slug, and
   `topK: 12` for "why" questions (they usually need broader context
   than the 8-default)
3. If `result.source === 'error'`, STOP — tell the user the FastAPI
   service is down (paste `result.errorDetail` verbatim, it has the
   `migrator serve` command). Do NOT hallucinate an answer.
4. If `result.retrievalEmpty === true`, STOP — tell the user "no
   evidence found in the warehouse for that question, likely means the
   context wasn't ingested. Try a different customerSlug or check
   ingestion coverage."
5. Otherwise generate `demo/warehouse-story-{shortHash}.html` using
   `templates/warehouse-story.html` — substitute:
   - `ANSWER_MARKDOWN` — Claude's `answerMarkdown` (already has `[N]` markers)
   - `CITATIONS_JSON` — the full citations array as pretty-printed JSON
   - `CITED_ORDINALS_JSON` — the citedOrdinals array
   - `QUERY_TEXT` — the original query
6. Open in Cowork's browser panel
7. Narrate: _"Claude walked five different systems to answer this — JIRA
   ticket PROJ-1023, a Confluence design doc, the Q1 planning meeting
   transcript at 4:32, and the code comment on line 47 of the batch
   class. Every citation is clickable."_

**What the audience sees:** Scrollytelling layout. Top: query as a
headline. Middle: Claude's answer with inline `[1] [2] [3]` markers
that highlight the corresponding citation card on scroll/hover. Bottom:
citation cards — each with icon per source type (📄 Confluence, 🎫 JIRA,
🎥 MeetingRecording, 📊 Slack, 💻 CodeComment), title, excerpt,
clickable link.

---

### 🎬 Wow #3 — Amendments Lifecycle Time-Travel

**Trigger:** user says _"demo the amendments"_, _"show renewals"_,
_"how does mid-quarter change work"_, _"walk me through an amendment"_,
_"can I audit a subscription state on a specific date?"_.

**Why this is the STRONGEST demo:** amendments demonstrate FIVE
deterministic mechanics in <90 seconds — LifecycleMatrix decision →
proration math → immutable ledger → time-travel replay → English
narrative. Every one is a "wait, do that again" moment for CFO / auditor
buyers.

**Prereqs:**

- An Asset in the connected org with `Status='Active'`. Query first:
  ```
  SELECT Id, Name, Quantity, Price, Product2.Name, AccountId, Account.Name
  FROM Asset WHERE Status='Active' AND Quantity > 0 ORDER BY CreatedDate DESC LIMIT 5
  ```
- If zero assets exist, tell the user to seed one via
  `scripts/apex/seed-avid-*.apex` first — do not fabricate an Asset Id.
- An active LifecycleMatrix (`ActiveMatrix__c=true`) with rules
  covering the asset's product segment. Verify with:
  ```
  SELECT Id, Name, EffectiveFrom__c, ActiveMatrix__c
  FROM LifecycleMatrix__c WHERE ActiveMatrix__c=true LIMIT 1
  ```

**Beats (target <90 seconds narration + <5s per tool call):**

1. **Set the scene** — pick the asset, state its current state out loud:
   _"Acme Health has 100 seats of EMR Suite Premium at $130/seat/month.
   Success manager just told us they need 20 more seats effective July
   15."_ Query the asset fresh so the numbers are real.

2. **Preview the amendment** — call `dd_cpq_preview_amendment` with:

   ```
   {
     assetId: "02i...",
     action: "Increase",
     effectiveDate: "2026-07-15",   // mid-month is intentional — shows proration
     proposedQuantityDelta: 20
   }
   ```

   Narrate the response fields IN THIS ORDER (this is the wow choreography):
   - `isAllowed: true` + `matchedRuleId` → _"LifecycleMatrix approved
     via rule LR-07 — Healthcare segment, Increase action, MinNotice met."_
   - `prorationMethod` + `prorationFactor` → _"Daily proration, factor
     0.516 — 16 of 31 days remaining in the July period."_
   - `netAmountDelta` → _"So this amendment nets $1,342.42 for the
     mid-month change. Not $2,600 (the naive full-period charge).
     Deterministic math, no LLM guessing."_
   - `requiresApproval` → surface if true (customer thresholds vary)

3. **Commit it** — call `dd_cpq_commit_amendment` with the SAME payload.
   Narrate: _"Ledger row written, sequence #{n} for this asset, that
   quantity is now {quantityAfter}. Append-only — only the Notes field
   can ever be updated after this insert."_ Persist the returned
   `assetTransactionId` for step 5.

4. **Time-travel replay** — call `dd_cpq_replay_asset` THREE times with
   escalating drama:
   - `asOfDate` = the day BEFORE the amendment → shows original state
   - `asOfDate` = today (post-amendment) → shows new state
   - `asOfDate` = 6 months from now → shows same as today
     (no future amendments — but this is the point)
     Narrate: _"Rep asks 'what did Acme have on July 1?' — 100. 'July
     20?' — 120. Same ledger, any date, folded on demand."_

5. **Reverse** — call `dd_cpq_reverse_amendment` with the
   `assetTransactionId` from step 3 and a note like _"Wrong quantity,
   reversing"_. Narrate: _"The compensating row lands with opposite
   sign + ReversalOf__c pointer. The ORIGINAL stays visible forever.
   That's the accounting-industry standard for corrections — no edits,
   no deletes, ever. Auditors love this."_

6. **Explain** — call `dd_cpq_explain_amendment` with a FRESH preview
   payload (different effectiveDate so it's a new narrative). Show the
   `narrative` field — this is the plain-English walk-through of every
   decision the engine made. Narrate: _"This isn't AI at runtime — it's
   deterministic templating, same inputs always produce the same text.
   Compliance-safe."_

7. **Render the timeline** — generate `demo/amendment-timeline-{assetId}.html`
   using `templates/amendment-timeline.html`. Substitute:
   - `ASSET_HEADER_JSON` — the fresh Asset SOQL row from step 1
   - `LEDGER_JSON` — SOQL: all `AssetTransaction__c` for this asset,
     ordered by SequenceNumber__c, including reversals
   - `REPLAY_POINTS_JSON` — the three ReplayResult objects from step 4
     Open in Cowork's browser panel. Narrate: _"Full lifecycle, one
     view — every amendment as a dot, hover for details, drag the
     scrubber to any date to time-travel the quantity + price."_

**What the audience sees on screen:**

- Header card: asset name, product, account, current quantity/price
- Horizontal timeline of every ledger row (green dots = Increase, red
  = Decrease/Cancel, purple = Reversal with linked-line to the original,
  blue = Renew, orange = Suspend/Resume)
- Time scrubber (drag left/right) that recomputes an "as of" state
  panel in real time (client-side fold — no round-trip)
- Ledger table below (sortable, filterable by TransactionType)
- Proration explainer sidebar for whichever transaction is selected —
  shows the days-used / days-remaining pie + the math walkthrough

**Failure modes to catch:**

- `isAllowed: false` on preview → STOP, do NOT try commit. Read
  `denyReason` aloud verbatim; it's designed for humans.
- `requiresApproval: true` on commit → still creates the ledger row
  BUT surfaces the flag. Narrate: _"Committed but flagged for
  approval — that's the standard SF approval process kicking in for
  amendments over the customer's threshold."_
- Asset has zero prior amendments → Wow #3 is less dramatic. Suggest
  the user create one via UI first, or run 2-3 quick amendments in a
  loop before the demo.

---

### 🎬 Wow #4 — Constitutional Auditor (planned, not yet built)

**Requires new MCP tools** (build these next after Wow #1 + #2 ship):

- `dd_cpq_lint_constitution`
- `dd_cpq_diff_constitutions`
- `dd_cpq_author_law_from_english`
- `dd_cpq_preview_law_impact`
- `dd_cpq_activate_law`
- `dd_cpq_law_lineage`

**Choreography sketch:** run linter → pick highest-impact issue → draft
fix → preview PVM impact → confirm → activate. See MIGRATOR_ARCHITECTURE
for the target flow.

---

### 🎬 Wow #4 — Deal Risk X-Ray (planned)

**Requires:**

- `dd_cpq_get_account_context`
- `dd_cpq_similar_quotes`
- `dd_cpq_deal_risk_score`
- `dd_cpq_recommend_concessions`

**Choreography sketch:** open quote → risk score + rationale → similar
won/lost deals → concessions playbook → render as a shareable dashboard.

---

### 🎬 Wow #5 — Voice + Autonomous Migration (planned)

**Requires:**

- `dd_cpq_migrator_run`
- `dd_cpq_migrator_status`
- `dd_cpq_generate_proposal_doc`
- Voice input (Claude Desktop native feature — no MCP work)

---

## Choreography rules (do not violate)

1. **Never fabricate a Salesforce answer.** If the MCP tool errored or
   returned `source: 'error'`, tell the user. Do not invent a quote id,
   product, or dollar amount. The demo credibility depends on this.

2. **Cowork's browser panel is the wow surface.** Text answers are
   fine. But every wow moment ends with an HTML render — a Sankey
   chart, a scrollytelling page, a dashboard. That's what makes the
   audience say "wait, do that again."

3. **One tool call at a time in the demo narration.** Even if you can
   parallelize, DON'T — the audience can't follow parallel actions.
   Serial calls with visible progress = perceived intelligence.

4. **Cite everything.** Every claim ends with a source: a PolicyLaw
   name, a JIRA ticket, a citation ordinal. RevOps buyers care about
   provenance more than they care about speed.

5. **Failure narrates too.** If retrieval was empty, say so. If a
   product wasn't found, say so. If the org isn't connected, say so.
   Silence or hallucination breaks the demo more than any error would.

6. **Rehearse the seed data.** Use the Acme Health Systems seed for
   quotes/products (in `scripts/apex/`), and the Avid Technologies
   catalog for bundle demos. Both are deterministic and have been
   verified against the golden test ($101.83/user/month).

## Follow-up work

When you finish a wow demo, offer the user:

- "Want me to save this HTML render as a shareable link?"
  → skill can zip + upload to a Gist or copy to `docs/demo-renders/`
- "Want me to record a Loom of this flow?"
  → user records; skill lists the exact prompt sequence to replay

## Related skills

- `dd-cpq-configurator-tester`, `dd-cpq-pricing-tester` — regression
  suites. Use when the goal is finding defects, not showing the product.
- `docs/APEX_GOTCHAS.md`, `docs/DEV_RECIPES.md` — the build knowledge
  that used to live in the retired `constitutional-cpq` skill.
- `update-config` — for editing `claude_desktop_config.json` env vars
  (e.g. setting `MIGRATOR_API_TOKEN` before Wow #2 will work)
