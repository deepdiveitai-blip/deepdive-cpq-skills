# Golden path — install and the first quote

The on-ramp every tester does first. Fixed facts (install id, permission
set names, `DDCPQ__` in queries) are in SKILL.md. The tester clicks; Claude
checks.

## Stage 0 — Laptop and org

**Tester needs:** VS Code with Claude Code, the Salesforce CLI, and an org:

- **Developer Edition** — free at `https://developer.salesforce.com/signup`, or
- **Developer sandbox** — from their company's production org.

Check the CLI: `sf --version`. If missing: `npm install -g @salesforce/cli`
(needs Node.js LTS), or the installer at
`https://developer.salesforce.com/tools/salesforcecli`.

**You run** (a browser opens; the tester logs in):

```bash
sf org login web -a dd-e2e -r https://login.salesforce.com   # Developer Edition
sf org login web -a dd-e2e -r https://test.salesforce.com    # sandbox
```

**Check:**

```bash
sf data query -o dd-e2e -q "SELECT Name, OrganizationType, IsSandbox, NamespacePrefix FROM Organization"
```

| Result                                                        | Meaning                                                   |
| ------------------------------------------------------------- | --------------------------------------------------------- |
| `OrganizationType` = Developer Edition, or `IsSandbox` = true | ✅ go on                                                  |
| anything else                                                 | ❌ **stop** — this looks like production. Do not install. |
| `NamespacePrefix` = DDCPQ                                     | ❌ **stop** — that is a DEV team org, not a clean one     |

## Stage 1 — Org settings (tester, ~3 min)

1. **Turn on Quotes.** Setup → Quick Find "Quote Settings" → **Enable** →
   Save (accept the default page layouts). The package adds fields to
   Quote, so this must happen **before** install.
2. **Add an Opportunity Type of "Renewal".** Setup → Object Manager →
   Opportunity → Fields & Relationships → **Type** → Values → **New** →
   `Renewal` → Save. A fresh org has no such value, and the golden quote
   needs it for its 3% renewal discount.

**Check:**

```bash
sf data query -o dd-e2e -q "SELECT COUNT() FROM Quote"
sf data query -o dd-e2e -q "SELECT Value FROM PicklistValueInfo WHERE EntityParticle.EntityDefinition.QualifiedApiName = 'Opportunity' AND EntityParticle.QualifiedApiName = 'Type' AND Value = 'Renewal'"
```

The first must not error ("sObject type 'Quote' is not supported" means
Quotes are still off). The second must return 1 row.

## Stage 2 — Install the package (~5–10 min)

Ask the tester which they prefer:

- **You run it:**
  ```bash
  sf package install -p 04tam000006c4LhAAI -o dd-e2e -w 30 -s AdminsOnly --no-prompt
  ```
- **They click it:** the install link for their org type (Fixed facts) →
  **Install for Admins Only** → tick the third-party acknowledgement →
  Install. It may say the install continues in the background; an email
  arrives when it is done.

**Check:** `sf package installed list -o dd-e2e` lists **DD CPQ Engine
0.1.0.11**.

If the install fails, read the error to the tester word for word and use
Troubleshooting. Do not retry the same thing in a loop.

## Stage 3 — Access and the Configure Products action

**You run:**

```bash
sf org assign permset -o dd-e2e -n DDCPQ__DD_CPQ_Engine_Admin
```

Tell the tester why: DD CPQ checks its own permissions, so **even a System
Administrator** is refused ("Working on a quote needs the "Build Quotes"
permission") without it.

**Tester adds the cart button to Quote** — the package ships the action
but cannot place it on their layout:

Setup → Object Manager → **Quote** → Page Layouts → **Quote Layout** →
in the palette pick **Mobile & Lightning Actions** → drag **Configure
Products** into "Salesforce Mobile and Lightning Experience Actions" (if
that section says it uses predefined actions, click the override link
first) → **Save**.

**Check:**

```bash
sf data query -o dd-e2e -q "SELECT PermissionSet.Name FROM PermissionSetAssignment WHERE Assignee.Username = '<their username>' AND PermissionSet.NamespacePrefix = 'DDCPQ'"
```

Get the username from `sf org display -o dd-e2e`. The layout cannot be
read back cheaply — ask the tester to confirm they saved it. From now on
they work in the **DeepDive CPQ** app (App Launcher → "DeepDive CPQ").

## Stage 4 — Products and prices (tester, ~10 min)

Create six products, **Active** ticked, each with a **Standard Price of
$150** (Products tab → New → Save; then the product's Related tab → Price
Books → **Add Standard Price** → 150 → Active → Save):

| Product           | Role in the scenario                         |
| ----------------- | -------------------------------------------- |
| CRM Suite Pro     | the bundle                                   |
| Sales Cloud       | bundle option — the line we price to $101.83 |
| Service Cloud     | bundle option                                |
| Storage +100GB    | bundle option                                |
| Premium Support   | bundle option                                |
| Compliance Add-on | sold on its own                              |

Create all six here, including CRM Suite Pro, **before** Bundle Builder.
A bundle product created inside Bundle Builder without a plan gets a $0
price.

**Check:**

```bash
sf data query -o dd-e2e -q "SELECT Product2.Name, UnitPrice, IsActive FROM PricebookEntry WHERE Pricebook2.IsStandard = true AND Product2.Name IN ('CRM Suite Pro','Sales Cloud','Service Cloud','Storage +100GB','Premium Support','Compliance Add-on')"
```

Expect 6 rows, all 150, all active.

## Stage 5 — The bundle (tester, Bundle Builder tab, ~10 min)

1. **Use an existing product** → CRM Suite Pro.
2. Pricing: **Options carry the price**.
3. **+ Add feature** → name it `Core`, **Pick several**.
4. **+ Add option** four times: Sales Cloud, Service Cloud, Storage +100GB,
   Premium Support. For each: **Charged separately**, on by default,
   required, Quantity **Match the bundle quantity**.
5. Each option shows a **No plan** badge. Click it → **Add a rate plan**:
   Charge type **Recurring**, Price **150**, Priced as **Per unit**,
   Billed **Monthly**.
6. The setup rail should be green. Click **Create bundle**.

**Check:**

```bash
sf data query -o dd-e2e -q "SELECT DDCPQ__ChildProduct__r.Name, DDCPQ__QuantityRule__c, DDCPQ__DefaultSelected__c, DDCPQ__Required__c FROM DDCPQ__ProductOption__c WHERE DDCPQ__ParentProduct__r.Name = 'CRM Suite Pro'"
sf data query -o dd-e2e -q "SELECT DDCPQ__TargetProduct__r.Name, DDCPQ__Status__c, DDCPQ__RevenueNature__c, DDCPQ__PricingModel__c, DDCPQ__BillingSchedule__c, DDCPQ__UnitPrice__c FROM DDCPQ__RatePlan__c"
```

Expect 4 options following the bundle quantity, and 4 **Active**
Recurring / PerUnit / Monthly plans at 150.

## Stage 6 — A rate plan for the standalone product (tester, Rate Plan Editor tab)

DD CPQ will not commit a priced line without an **Active** rate plan.
Compliance Add-on is not in the bundle, so it still has none — the
Coverage view lists it under products that need attention.

Open Compliance Add-on → add a plan: **Recurring**, **Per unit**,
**Monthly**, price **150** → set it **Active** → Save.

**Check:** rerun the RatePlan query from Stage 5. Expect 5 Active plans.

## Stage 7 — Price and discount rules (tester, Rules Studio tab, ~15 min)

Build these three rule sets. Rules Studio saves a new rule set as
**Draft** — each one must be switched to **Active** or the engine ignores
it.

| Rule set                                                             | Rule(s) — all target **Sales Cloud** | Condition                          |
| -------------------------------------------------------------------- | ------------------------------------ | ---------------------------------- |
| Price rule set `Default Pricing`                                     | Absolute price **$130**              | Account.Industry equals Healthcare |
| Discount rule set `Default System Discounts`, stacking **all stack** | **5%**                               | Account.Industry equals Healthcare |
|                                                                      | **3%**                               | Opportunity.Type equals Renewal    |
| Volume tiers `Default Volume`                                        | **15%** from quantity **100**        | (no condition)                     |

**Check:**

```bash
sf data query -o dd-e2e -q "SELECT Name, DDCPQ__Status__c, (SELECT DDCPQ__TargetProduct__r.Name, DDCPQ__PriceType__c, DDCPQ__PriceValue__c, DDCPQ__FieldApiName__c, DDCPQ__Operator__c, DDCPQ__Value__c FROM DDCPQ__PricingRules__r) FROM DDCPQ__PricingMatrix__c"
sf data query -o dd-e2e -q "SELECT DDCPQ__Matrix__r.Name, DDCPQ__Matrix__r.DDCPQ__Status__c, DDCPQ__Matrix__r.DDCPQ__StackingStrategy__c, DDCPQ__TargetProduct__r.Name, DDCPQ__DiscountValue__c, DDCPQ__FieldApiName__c, DDCPQ__Value__c FROM DDCPQ__SystemDiscountRule__c"
sf data query -o dd-e2e -q "SELECT DDCPQ__Matrix__r.Name, DDCPQ__Matrix__r.DDCPQ__Status__c, DDCPQ__TargetProduct__r.Name, DDCPQ__MinQuantity__c, DDCPQ__DiscountValue__c FROM DDCPQ__VolumeDiscountTier__c"
```

If the subquery relationship name is rejected, query `DDCPQ__PricingRule__c`
directly with `DDCPQ__Matrix__r.Name`. Every rule set must read **Active**
and every rule must target Sales Cloud.

## Stage 8 — The quote (tester, ~10 min)

1. **Account** `Acme Corp`, Industry **Healthcare**.
2. **Opportunity** `Acme Q3 Renewal` on that account, Type **Renewal**,
   Stage Proposal/Price Quote, any close date.
3. From the Opportunity's Quotes list → **New Quote** `Q-0042 Acme`.
4. Open the quote → **Configure Products**. The DD CPQ cart opens.
5. **Add products** → CRM Suite Pro, quantity **100**. Its four options
   come with it at 100 each.
6. **Add products** → Compliance Add-on, quantity **100**.
7. Read **Sales Cloud**: net price **$101.83**. Click the price to open
   the waterfall: $150 list → $130 price rule → −5% → −3% → −15% volume →
   $101.83.
8. **Commit.**

**Check:**

```bash
sf data query -o dd-e2e -q "SELECT Product2.Name, Quantity, UnitPrice, DDCPQ__DerivedListPrice__c, DDCPQ__NetPrice__c, DDCPQ__ARR__c, DDCPQ__ParentLine__r.Product2.Name FROM QuoteLineItem WHERE Quote.Name = 'Q-0042 Acme'"
```

## The finish line

Verified on installed 0.1.0-9 and 0.1.0-11 orgs (0.1.0-11 in a single-currency org). A correct build gives exactly this:

| Line              | Derived list | Net        | ARR         | Parent line   |
| ----------------- | ------------ | ---------- | ----------- | ------------- |
| CRM Suite Pro     | 150          | 150        | 180,000     | —             |
| Sales Cloud       | **130**      | **101.83** | **122,196** | CRM Suite Pro |
| Service Cloud     | 150          | 150        | 180,000     | CRM Suite Pro |
| Storage +100GB    | 150          | 150        | 180,000     | CRM Suite Pro |
| Premium Support   | 150          | 150        | 180,000     | CRM Suite Pro |
| Compliance Add-on | 150          | 150        | 180,000     | —             |

6 lines, 4 of them pointing at the bundle. When it matches, congratulate
the tester and give them a scorecard: each stage, ✅/❌, minutes taken if
they told you, and every place they got stuck — those are the findings.

## Troubleshooting

**Sales Cloud is not $101.83** — the number tells you which piece is missing:

| Net shown | What is missing            | Look at                                                   |
| --------- | -------------------------- | --------------------------------------------------------- |
| ≈ 117.49  | the $130 price rule        | Default Pricing Active? Account Industry = Healthcare?    |
| ≈ 104.98  | the 3% renewal discount    | Opportunity Type = Renewal? rule value spelled `Renewal`? |
| ≈ 107.19  | the 5% Healthcare discount | Account Industry; discount set Active                     |
| ≈ 119.80  | the 15% volume tier        | quantity is 100? volume tiers Active?                     |
| 130       | all three discounts        | discount and volume sets still Draft                      |
| 150       | everything                 | rules target the wrong product, or every set is Draft     |

**Read-only price preview** — to see what the engine itself says,
independent of the cart, query the quote, account and product Ids, write
the body to a file, and run:

```bash
sf api request rest "services/apexrest/DDCPQ/dd/v1/price?includeWaterfall=true" -o dd-e2e --method POST --body @body.json
```

```json
{
  "quoteId": "<0Q0…>",
  "selections": [
    {
      "localKey": "p",
      "productId": "<CRM Suite Pro 01t…>",
      "quantity": 100,
      "sourceBundleId": "<CRM Suite Pro 01t…>"
    },
    {
      "localKey": "c1",
      "parentLocalKey": "p",
      "productId": "<Sales Cloud 01t…>",
      "quantity": 100,
      "sourceBundleId": "<CRM Suite Pro 01t…>"
    }
  ]
}
```

The path has **no leading slash**. Preview writes nothing. **Never** call
`dd/v1/commit` — committing is the tester's step.

| Symptom                                                      | Cause and fix                                                                     |
| ------------------------------------------------------------ | --------------------------------------------------------------------------------- |
| Install fails naming Quote or QuoteLineItem                  | Quotes not enabled — Stage 1                                                      |
| Install fails naming Order or OrderItem                      | Setup → Order Settings → Enable Orders, then install again                        |
| "needs the "Build Quotes" permission" / "Administer Pricing" | Stage 3 permission set not assigned to this user                                  |
| No **Configure Products** on the quote                       | layout step in Stage 3; meanwhile open the **DD CPQ Cart** tab and pick the quote |
| Commit refused: "…Each needs an Active rate plan…"           | a priced product has no Active plan — Stages 5 and 6                              |
| Bundle options missing in the cart                           | options not on by default, or not saved — rerun the Stage 5 check                 |
| Options do not follow quantity 100                           | Quantity not set to **Match the bundle quantity**                                 |
| Renewal not offered on the Opportunity                       | Stage 1, step 2                                                                   |
| `sf` says the org is expired or auth failed                  | `sf org login web` again with the same alias                                      |
