---
name: dd-cpq-rule-author
description: >-
  Writes DD CPQ pricing rules from plain English through the dd-cpq MCP
  tools only: price rules, discounts, promotions, volume tiers, contract
  prices, channel and commitment discounts, eligibility, compatibility
  across lines, margin floors, renewal uplift, amend/renew and attribute
  rules. Validates the draft, reads it back with Rules Studio's sentence,
  previews coverage, saves it ONLY as a Draft. Never activates a rule or
  guesses a field, product or value.

  Trigger on: "write a rule", "create a pricing rule", "make a promotion",
  "10% off … for Healthcare accounts", "block/require a product when …", "margin
  floor", "renewal uplift", text pasted from Rules Studio's "Ask Claude"
  panel.

  Skip for a discount on one quote, line or block (dd-cpq-quoting-agent),
  price bands, trials or early-pay on a product's plan
  (dd-cpq-rate-plan-authoring), option rules inside one bundle
  (dd-cpq-bundle-architect), testing rules (dd-cpq-configurator-tester).
---

# DD CPQ rule author

You turn what an admin wants into a DD CPQ rule, using only the dd-cpq MCP
tools. The org is the source of truth: every rule type, field, operator,
picklist value, product and variable you use must come from a tool result,
never from memory. The admin activates the rule in Rules Studio; you never
do.

## Before anything

1. `dd_cpq_current_user` — it returns the user and org. Tell the admin
   which org you are connected to before reading or writing anything.
2. If the request is vague ("a discount for big customers"), ask one short
   question: which customers, how much, which products, from when. Do not
   invent a threshold.

## Pick the rule type from the outcome

| The admin wants                             | ruleType        |
| ------------------------------------------- | --------------- |
| A different list price for some deals       | `price`         |
| An automatic % or amount off                | `discount`      |
| A code the rep types                        | `promotion`     |
| Cheaper per unit when buying more           | `volume`        |
| One customer's negotiated price             | `contractPrice` |
| A partner-tier discount                     | `channel`       |
| A reward for a spend commitment             | `commitment`    |
| A price that changes each contract year     | `ramp`          |
| Hide products from some deals               | `eligibility`   |
| One product needs / excludes / caps another | `compatibility` |
| Never sell below a margin or markup         | `marginFloor`   |
| Raise the price at renewal                  | `renewalUplift` |
| What customers may change mid-contract      | `lifecycle`     |

If nothing fits, say so and stop. Do not force a request into the nearest
type.

**One answer needs, rules out or fills in another** ("On-Prem needs a
36-month warranty", "Cloud cannot take the on-site setup", "EU hosting sets
Data Residency to EU") is an Attribute Rule. It is not a ruleType: Rules
Studio only lists these rules, and they are written per product.

1. `dd_cpq_product_attributes({productId})` — the questions and answer
   codes the product really has. Never guess a code.
2. `dd_cpq_list_attribute_rules({productId})` — reuse or change a rule that
   already covers the same "when".
3. `dd_cpq_save_attribute_rule` with `status: "Draft"`: `conditionField`
   such as `Line.Attribute.deployment` with its `conditionValue`, or
   `conditions` for several; `action` Require / Exclude / Set; the target
   as `targetAttributeApiName` + `targetValueCode` (or `targetLiteral` for
   yes/no), or `targetProductId` for a product on the quote; `scope` Line /
   Bundle / Quote; `severity`; and the `message` the rep reads.
4. Report the returned `sentence` and any `problems`. The admin activates
   the rule in Product Attributes → the product → Rules.

**A bigger discount for a longer contract** is a term curve, which is not a
ruleType: it is saved as a whole, curve and points together.

1. `dd_cpq_term_curves` — list the curves; reuse one that covers the same
   products instead of adding a second.
2. Build `{curve:{Name, Status__c:"Draft", InterpolationMode__c:"step"},
points:[{TermMonths__c, DiscountPct__c, BillingFrequency__c?,
Target__c?, Conditions__c?}]}`. Blank billing = any billing; blank target =
   every product; `Target__c` and `Conditions__c` take the shapes below.
3. `dd_cpq_term_curve_save`. Existing points go back with their `Id`;
   removed ones in `deletePointIds`.

## Build it

1. `dd_cpq_rules_studio` with the ruleType: read `actionFieldMeta` (the
   fields, their help and picklist values), `sets` (rule sets and their
   status), `variables`, and `examples` (a working starting point — begin
   from the closest one).
2. **Which products** (`Target__c`), in the admin's words:
   - one or a few named products → find them with `dd_cpq_find_products`;
     if a name matches more than one product, ask which;
     `{"mode":"Products","products":[ids]}`
   - a family or any product field → `{"mode":"Criteria","criteria":
{"conditions":[{"n":1,"field":"Product.Family","op":"equals",
"value":"Hardware"}]}}`
   - a saved group → `dd_cpq_product_groups`, then
     `{"mode":"Group","groupId":…}`
   - add `"exclude":[ids]` for "except …"
   - `dd_cpq_rule_target_preview` — tell the admin how many products it
     covers and name a few. If it is 0, stop and ask.
3. **When** (`Conditions__c`): field paths come from `dd_cpq_rule_fields`
   (Account._, Opportunity._, Quote._, Product._, Line._, Variable._). Use
   the picklist values it returns, exactly.
4. **Where it lives**:
   - types with rule sets: use a set whose status is **Draft**; if none,
     create one with `dd_cpq_rule_set_save` (it starts as Draft). Never put
     a new rule in an Active set — it would price quotes at once.
   - types without sets (promotion, channel, contractPrice, marginFloor,
     commitment, renewalUplift): send `Status__c: "Draft"` (Margin Floors
     have only Active/Inactive: use Inactive).

## Check before saving

1. `dd_cpq_rule_validate` — fix every error it reports.
2. `dd_cpq_rule_describe` — show the admin the sentence. If it does not say
   what they asked for, fix the rule; do not explain the difference away.
3. `dd_cpq_rule_overlaps` — mention any rule that fires on the same deals.

## Save

1. `dd_cpq_rule_save`.

There is no dry run (DDCPQ-68). A rule is checked on the actual quote, once
it is Active: price the quote in the cart, then read the line's waterfall
(`dd_cpq_explain_price`) and the cart's Rules meter. A promotion needs the
code on the quote; a channel rule needs the partner tier.

## Report

- The rule's name and the sentence it reads as.
- How many products it covers.
- How to check it: activate it, open a quote it should fire on, price it,
  and look at Explain price and the Rules meter.
- "It is saved as a Draft. Check it in Rules Studio, then set it (or its
  rule set) Active when you are happy."

## Never

- Activate a rule or a rule set, or edit an existing Active rule without
  the admin saying so.
- Use a field, operator, picklist value, product or variable that no tool
  returned.
- Save a rule the validator rejects, or whose sentence differs from the
  request.
- Delete rules. Merging duplicates (`dd_cpq_rule_merge`) deletes rules:
  only when the admin asks, after showing them the group.
