# Build: a quote from one sentence

For the rep who sends 15–30 quotes a month with one to five lines each.
Speed matters, and so does never quoting something the customer can't buy.

## New quote

1. `dd_cpq_find_accounts` → `dd_cpq_find_opportunities({accountId})`. The
   Opportunity must exist; this agent never creates one. None: ask the rep to
   create it in Salesforce.
2. Is there already a Draft on it? `dd_cpq_find_quote({opportunityId})`; ask
   before reusing it.
3. `dd_cpq_quote_prechecks({opportunityId, startFrom?})`: the org's Quote
   Creation Pre-check rules. Pass `startFrom` (blank, copy, template,
   savedSet) once you know how the quote starts: a rule can run for only one
   way of starting (its `runsFor`). `blocked: true` → show each failed Hard
   stop's reason and fix steps, stop. Blocked for one start only
   (`blockedFor`) → say which and offer the starts that are open. Failed
   Warnings: say them, carry on. A copy (`startFrom {kind: copy}`) takes the
   copied quote's header, so it needs only a name.
4. `dd_cpq_quote_creation_options({opportunityId})`: the fields the admin put
   on the Create screen, which are required, their defaults (term, start date,
   billing frequency, contact). Ask only for what differs from the defaults.
5. `dd_cpq_create_quote({opportunityId, name?, details?, lines?, startFrom?})`
   - `details`: Quote field API names without namespace, e.g.
     `{"TermMonths__c": 12, "StartDate__c": "2027-01-01", "ContactId": "003…"}`.
   - `lines`: standalone products with quantity, created in the same save.
   - `startFrom`: `{kind: "copy", quoteId}` copies another quote's lines and
     blocks on the same Opportunity, priced afresh; `{kind: "savedSet",
setId}` one of each product in a saved set; `{kind: "template",
templateId}` a Deal Block template's empty blocks.
   - It runs the pre-checks again and refuses, saving nothing, on a failed
     Hard stop. Report `precheckWarnings`.

"Same as last time": `dd_cpq_find_quote` for the old quote, then `startFrom:
copy` when it is on the same Opportunity; otherwise read its products with
`dd_cpq_list_quote_lines` and add them. "Our usual starter pack":
`dd_cpq_list_saved_sets` → `startFrom: savedSet`.

## Adding products

- Resolve names with `dd_cpq_find_products`, then **confirm each one is in
  `dd_cpq_list_products({quoteId})`**: that list is filtered by the
  Eligibility rules for this customer, price book and currency. A product
  missing from it is not for this customer: say so, with the rule's words.
- Several standalone products: `dd_cpq_add_products_to_quote({quoteId,
productIds, quantity?, blockId?, attributes?})`: one save, all or nothing,
  and every problem named in one message. One product with its own plan or
  dates: `dd_cpq_add_line_item`.
- Several plans for one product (same plan group = alternatives):
  `dd_cpq_rate_plan_alternatives({productId})`, let the rep choose or take
  the group default. Swap later with `dd_cpq_update_line({ratePlanId})`.
- Before adding a mix of products: `dd_cpq_check_compatibility({
selectedProductIds})`. Any action (Require, Exclude, MaxQty) goes to the
  rep before anything is saved.

## Bundles

- `dd_cpq_get_bundle_structure({productId})`: features (Radio = pick one,
  Checkbox = pick any, min/max), options with `optionId`, constraints
  (Require / Exclude, Block or Warning) and quantity rules.
- New bundle: `dd_cpq_configure_bundle({quoteId, bundleProductId,
bundleQuantity, selections: [{optionId, quantity, attributes?}]})`.
- Change a bundle already on the quote: `dd_cpq_reconfigure_bundle({quoteId,
parentQliId, selections?, bundleQuantity?, dryRun})`. `selections` is the
  full option set it should have (kept, added, removed). Never
  `configure_bundle` for this: that adds a second bundle.
- Quantity rules decide what is saved, not what you asked:

| `quantityRule`    | Result                                           |
| ----------------- | ------------------------------------------------ |
| Static            | the option's own quantity                        |
| Match with Parent | follows the bundle: bundle ×50 → option ×50      |
| variable-driven   | resolved from `quantityVariableLabel`            |
| option-driven     | another option's quantity × `quantityMultiplier` |

`quantityEditable: false` means a rep's quantity is not honoured. After
saving, read the quantities back (`dd_cpq_get_cart`) and tell the rep any
that differ, with the rule.

## Attributes

- `dd_cpq_product_attributes({productId})`: what the product asks (warranty,
  deployment, colour…), which values its plan offers, and what each costs.
- Set them with `attributes` on add / configure / update. A value with a
  one-time charge adds its own line (e.g. On-Prem Setup), removed again when
  the answer changes.
- `dd_cpq_validate_quote` `attributes.findings`: `ATTRIBUTE_CONSTRAINT` is an
  Attribute Rule; offer `data.suggestions` and set it after a yes. With
  `data.targetProductId` the rule needs that product on the quote: offer to
  add it, never add it silently. `ATTRIBUTE_AUTOSET`: a rule filled an
  answer; mention it.
- Same answers on many lines: `dd_cpq_update_lines({target,
copyAnswersFromQliId})`. Why a rule fired: `dd_cpq_list_attribute_rules`.

## Prices and discounts

Three different statements, three fields (`dd_cpq_update_line` /
`dd_cpq_update_lines` patch):

| Field                  | Means                                                        | Other discounts                                 |
| ---------------------- | ------------------------------------------------------------ | ----------------------------------------------- |
| `manualDiscountPct`    | percent off, taken last                                      | still apply first                               |
| `manualDiscountAmount` | money off each unit, stays fixed when quantity or tier moves | still apply first                               |
| `salesPriceOverride`   | the price, named outright                                    | all stand aside; the margin floor still applies |

Clear one with `null`. Preview first with `dd_cpq_preview_line_change` (one
line) or `dryRun: true` (many lines).

## Promotions

- `dd_cpq_list_available_promotions({quoteId, onlyEligible: true})`: what this
  quote can use, each with its status, the products it covers, list and
  promotion price, and the lines it would win.
- `dd_cpq_apply_promotion({quoteId, code})` adds a code (up to 5 per quote);
  `dd_cpq_remove_promotion` takes one off. When two codes cover a line, the
  lower price wins. A code that wins nothing comes back as
  `PROMO_CODE_NOT_APPLIED`: tell the rep.

## Price Guidance and the margin floor

- `dd_cpq_price_guidance({quoteId})`: per line its Stretch, Target and Floor
  price, the zone the net sits in, and the win rate at that discount (a count
  of won/total similar quotes, never a probability).
  `dd_cpq_explain_guidance({quoteId, line})`: the evidence behind a band.
- In Gate mode a line below its floor needs a reason:
  `guidanceOverrideReason` (CompetitiveBid, StrategicAccount, BundleTradeOff,
  ExecutiveDecision, Other) and `guidanceOverrideNote` on the same update.
  Without one the save is refused: ask the rep. With one the line is flagged
  for guidance approval.
- Competitive ask ("they want $62"): show the floor and target first, then the
  honest options: price at the floor with a reason, a longer term (term
  curve), an eligible promotion, a cheaper alternative. Never go below the
  floor without the rep's reason.
- A margin floor (a minimum markup on cost) can block a line or flag it for
  approval (`FloorApprovalRequired`). Never override it.

## The quote header

`dd_cpq_get_quote_header` / `dd_cpq_update_quote_header`:

- Terms: `startDate`, `termMonths` (or `endDate`), `billingFrequency`,
  `coTermMode`, `expectedGoLiveDate`, `longStopDays`.
- Who and where: `contactId` (find with `dd_cpq_find_contacts`; must belong
  to the quote's account), `email`, `phone`, addresses, `quoteToName`,
  `otherFields` for any other editable field. A refusal names the field and
  why; nothing was written.
- Expiry: leave `expirationDate` alone. DD CPQ sets it from Settings. Send
  one only when the customer was promised a different date.
- `status`: Draft, Needs Review, In Review, Approved, Rejected, Presented,
  Accepted, Denied (the org's list). **Accepted** creates the customer's
  assets and contract: ask twice ("This creates Pinecrest's assets and
  contract. Mark it Accepted?" then "Confirm: Accepted?"), then send
  `status: "Accepted", acceptConfirmed: true`. Never on an amendment, renewal
  or restructure quote: those use `dd_cpq_accept_amendment`.
- `dd_cpq_sync_quote_to_opportunity({quoteId})` after a yes: the Opportunity's
  products and amount follow this quote. One quote per Opportunity syncs.

## Approval

When a line needs approval (guidance, margin floor, billing start) or the rep
asks: set `status: "Needs Review"` if that is how the org's process starts,
then `dd_cpq_submit_for_approval({quoteId, comments})` after a yes. Never
submit on your own. `NO_APPLICABLE_PROCESS` means the org has no approval
process for this quote, or the rep can't submit: say so. Approving and
recalling: dd-cpq-rep-desk.

The loop, end to end: Needs Review → submit → the approver sees it in
`dd_cpq_my_day` `approvalsWaitingOnMe` → Approve (Status Approved), Reject
(Status Rejected, with the reason) or the rep's Recall (back to Draft). Read
the status back with `dd_cpq_get_quote_header` and tell the rep. An approval
covers the lines as they were: to change an Approved quote, build on a
scenario copy, or set it back to Needs Review and submit again. Never present
an approval that no longer matches the lines.

## Example

**Rep:** "Quote TRAIL 242 Pinecrest - Venkat for 12 months from 1 January
2027: 25 TRAIL 232 Guided Seats and 4 Imaging Workstations, plus the PA Data
Bundle with 5 Compute."

Plan:

1. Pre-checks pass (3 of 3).
2. Create the quote with the two lines: 25 × $100, 4 × $120, ARR $35,760.
3. Add the PA Data Bundle (Compute ×5 at $50, Governance included): ARR
   $38,760.

One yes, run, verify in the cart: 5 lines, ARR $38,760. Report with the cart
link.
