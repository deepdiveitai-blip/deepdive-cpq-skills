---
name: dd-cpq-scratch-refresh
description: Spin up a fresh DD CPQ scratch org from scratch — create, deploy all metadata, set active v2 constitution, run demo seeds, verify pricing model math. Use when the current scratch org is expiring, when the org is corrupted, or when a new dev needs their own copy. Repeatable in ~10 minutes end-to-end.
---

# DD CPQ scratch org refresh — the repeatable playbook

**Context:** Salesforce scratch orgs auto-die on a fixed date (default 7 days,
we use 30). Before that day, we need a fresh org with the same code + demo
seeds + v2 pricing engine active. This skill is the exact sequence that works.
Learned the hard way on 2026-07-29 by running through it and hitting every
Salesforce gotcha in order.

## When to invoke

- Current scratch org expires in <7 days.
- The current scratch org is in a bad state (corrupt data, botched deploy).
- A new dev needs their own personal copy for isolated testing.
- User says: "spin up a fresh org", "new scratch", "refresh the scratch", "org expiring".

## Skip this skill when

- The current org is fine and expires in >7 days — no rush, don't churn.
- You just need to redeploy code (that's `sf project deploy start`, not this).
- You want to seed one specific demo (that's just `sf apex run --file scripts/apex/seed-X.apex`).

## The full sequence — copy-paste ready

Every step below has been run against a fresh org. Order matters. Do NOT
skip steps because "this one usually works" — the gotchas are exactly at
the "usually works" boundaries.

### Step 1 — Create the org (2 min)

```powershell
sf org create scratch `
  --target-dev-hub PartnerDev `
  --alias cpq-scratch-v2 `
  --definition-file config/project-scratch-def.json `
  --duration-days 30 `
  --set-default
```

**Alias convention:** `cpq-scratch-vN` where N increments. Never overwrite the
current alias — testers may be mid-session on the old org. Cutover the alias
name only once the new org is fully verified.

**Grab a memorable password immediately** (default password is CLI-generated
and long):
```powershell
sf org generate password --target-org cpq-scratch-v2 --length 12
sf org display --target-org cpq-scratch-v2 --verbose  # capture URL + username + password + expiration
```

### Step 2 — Deploy in two waves (5 min)

**Wave 1: external credentials + named credentials FIRST.** Permsets that
reference these fail to deploy if the credential doesn't exist yet. As of
2026-08-04 the external credential also declares the `Default` NamedPrincipal
in source, so Wave 1 creates it — which is what lets Wave 2 deploy ALL four
permission sets (see below). It also wires the `x-api-key` header; only the
key VALUE is manual (Step 7).

```powershell
sf project deploy start --target-org cpq-scratch-v2 `
  --source-dir force-app/main/default/externalCredentials `
  --source-dir force-app/main/default/namedCredentials `
  --ignore-conflicts --wait 10
```

**Wave 2: everything except the types that break a fresh org.**

- ~~`standardValueSets`~~ — **no longer excluded as of 2026-08-04.** The old "inserts into standard picklists aren't supported" error was caused by a **misnamed file**: `AccountIndustry.standardValueSet-meta.xml`. There is no `AccountIndustry` standard value set — the real one is `Industry`, so Salesforce read the deploy as an attempt to CREATE a new standard value set, which is never allowed. Renamed to `Industry.standardValueSet-meta.xml`; both it and `AccountType` now deploy clean. Include the directory.
- `connectedApps` — **now permanently excluded via `.forceignore`.** Salesforce disabled connected app creation org-wide (not just scratch), and the New Connected App button is gone from App Manager, so there is no manual fallback either. Superseded by the four External Client App directories below, which DO deploy — include them.
- ~~`permissionsets/DD_CPQ_Admin` / `DD_CPQ_Amendments`~~ — **no longer excluded as of 2026-08-04.** Wave 1 now creates the `Anthropic_API-Default` principal from source, so all four permission sets deploy clean. Deploy the whole `permissionsets` directory. (If you ever see "The Default parameter value doesn't exist" again, Wave 1 didn't run or failed — fix that rather than excluding the permsets.)

```powershell
sf project deploy start --target-org cpq-scratch-v2 `
  --source-dir force-app/main/default/applications `
  --source-dir force-app/main/default/aura `
  --source-dir force-app/main/default/cachePartitions `
  --source-dir force-app/main/default/classes `
  --source-dir force-app/main/default/contentassets `
  --source-dir force-app/main/default/customMetadata `
  --source-dir force-app/main/default/externalClientApps `
  --source-dir force-app/main/default/extlClntAppOauthSettings `
  --source-dir force-app/main/default/extlClntAppGlobalOauthSets `
  --source-dir force-app/main/default/extlClntAppOauthPolicies `
  --source-dir force-app/main/default/flexipages `
  --source-dir force-app/main/default/layouts `
  --source-dir force-app/main/default/lwc `
  --source-dir force-app/main/default/objects `
  --source-dir force-app/main/default/pages `
  --source-dir force-app/main/default/settings `
  --source-dir force-app/main/default/staticresources `
  --source-dir force-app/main/default/tabs `
  --source-dir force-app/main/default/triggers `
  --source-dir force-app/main/default/permissionsets `
  --ignore-conflicts --ignore-warnings --wait 30
```

Expected: `Status: Succeeded`, ~825 components.

### Step 3 — Deploy DD_CPQ_Admin with external-cred block temporarily stripped

The Admin permset is the ONLY one that grants full FLS on all DD CPQ objects.
Seeds need it. Strip the credential block, deploy, restore the file:

```powershell
$permset = 'force-app/main/default/permissionsets/DD_CPQ_Admin.permissionset-meta.xml'
Copy-Item $permset "$env:TEMP\DD_CPQ_Admin.bak"

# Strip the block via a here-string replacement (avoids sed on Windows)
(Get-Content $permset -Raw) `
  -replace '(?s)\s*<externalCredentialPrincipalAccesses>.*?</externalCredentialPrincipalAccesses>', '' `
  | Set-Content $permset -Encoding UTF8

sf project deploy start --target-org cpq-scratch-v2 --source-dir $permset --ignore-conflicts --wait 5

# Restore original source
Copy-Item "$env:TEMP\DD_CPQ_Admin.bak" $permset -Force
```

Then assign it:
```powershell
sf org assign permset --target-org cpq-scratch-v2 --name DD_CPQ_Selling
sf org assign permset --target-org cpq-scratch-v2 --name DD_CPQ_Admin
```

### Step 4 — Insert active v2 PolicyConstitution (1 min)

Without a v2 constitution, `PricingModelService` never fires and every quote
returns `list × qty` regardless of the profile's PricingModel. This is THE
trap that made testers report "the 5 pricing models don't work."

```powershell
$apex = @'
PolicyConstitution__c c = new PolicyConstitution__c(
    Name = 'DD CPQ Bootstrap',
    Status__c = 'Active',
    PricingEngineVersion__c = 'v2'
);
insert c;
System.debug('constitution active v2: ' + c.Id);
'@
$apex | sf apex run --target-org cpq-scratch-v2
```

Expected `constitution active v2: <id>` in the debug log.

### Step 5 — Run the seed scripts (2 min)

Only the demos that don't depend on customer catalogs. If you need Microsoft
or AWS catalogs, run those first — they're prerequisites for the term-discount
demo.

```powershell
sf apex run --file scripts/apex/seed-pricing-model-demo.apex --target-org cpq-scratch-v2
sf apex run --file scripts/apex/seed-hybrid-charge-profile-demo.apex --target-org cpq-scratch-v2

# Optional: customer catalogs before their laws + demos
# sf apex run --file scripts/apex/seed-microsoft-1-catalog.apex --target-org cpq-scratch-v2
# sf apex run --file scripts/apex/seed-microsoft-4-simple-laws.apex --target-org cpq-scratch-v2
# sf apex run --file scripts/apex/seed-term-discount-demo.apex --target-org cpq-scratch-v2
```

### Step 6 — Verify all 5 pricing models produce the expected totals

The gold-standard smoke test. If this passes, the org is fully working:

```powershell
foreach ($M in @('Flat Fee', 'Per Unit', 'Tiered', 'Volume', 'Package')) {
$apex = @"
Id qId = [SELECT Id FROM Quote WHERE Name = 'DDCPQ Model . $M' LIMIT 1].Id;
QuoteLineItem qli = [SELECT Id, Product2Id, PricebookEntryId, Quantity FROM QuoteLineItem WHERE QuoteId = :qId LIMIT 1];
CpqEngine.Request req = new CpqEngine.Request();
req.quoteId = qId; req.selections = new List<CpqEngine.Selection>();
CpqEngine.Selection s = new CpqEngine.Selection();
s.localKey='k1'; s.productId=qli.Product2Id; s.quantity=qli.Quantity;
req.selections.add(s);
List<CommitService.LineDraft> drafts = CpqEngine.previewDrafts(req);
System.debug('RESULT $M: total=' + (drafts[0].netPrice * drafts[0].quantity));
"@
$apex | sf apex run --target-org cpq-scratch-v2 2>&1 | Select-String 'RESULT'
}
```

**Expected output** (list=$2K, qty=10 for all):
| Model | Expected total |
|---|---|
| Flat Fee | $2,000 |
| Per Unit | $20,000 |
| Tiered | $17,500 |
| Volume | $15,000 |
| Package | $18,000 |

If ALL 5 match — org is ready. If any two match or any number is wrong,
check `PolicyConstitution.Status__c = 'Active'` and
`PricingEngineVersion__c = 'v2'` first.

### Step 7 — Manual UI steps (defer until testers actually need them)

One thing you cannot deploy from source — **the Anthropic API key value**:

1. **`ApiKey` authentication parameter.** The `Default` principal itself now
   deploys in Wave 1, and the `x-api-key` header is wired in source to
   `{!$Credential.Anthropic_API.ApiKey}`. But the `Custom` auth protocol does
   **not** accept an `AuthParameter` via metadata (deploy error: *"The
   authentication protocol Custom doesn't support the following external
   credential parameter type(s): AuthParameter"*), so the parameter is created
   together with its secret value in the UI:

   Setup -> Named Credentials -> **External Credentials** tab -> Anthropic API
   -> Principals -> `Default` -> Edit -> Authentication Parameters -> Add:
   Name = `ApiKey` (exact case — the merge field is case-sensitive),
   Value = the `sk-ant-...` key from https://console.claude.com/settings/keys

   Verify with a real callout, not a dry run (a dry run only validates XML and
   never opens a connection):

   ```apex
   HttpRequest req = new HttpRequest();
   req.setEndpoint('callout:Anthropic_API/v1/messages');
   req.setMethod('POST');
   req.setHeader('Content-Type', 'application/json');
   req.setHeader('anthropic-version', '2023-06-01');
   req.setBody('{"model":"claude-haiku-4-5-20251001","max_tokens":16,'
     + '"messages":[{"role":"user","content":"Reply with the single word OK."}]}');
   HttpResponse res = new Http().send(req);
   System.debug('STATUS=' + res.getStatusCode() + ' BODY=' + res.getBody());
   ```

   Expect `STATUS=200`. A **401 means the merge field resolved empty** — the
   parameter is missing or misnamed.

**No longer manual (2026-08-04): the MCP OAuth app.** It used to be an 8-step
UI walkthrough for a Connected App. It is now an **External Client App** that
deploys with Wave 2 above — nothing to click. Grab the org-generated Consumer
Key for the MCP server's `SF_CLIENT_ID` with:

```powershell
sf project retrieve start -o cpq-scratch-v2 `
  -m "ExtlClntAppGlobalOauthSettings:DD_CPQ_MCP" `
  --target-metadata-dir ./tmp-eca --unzip
```

Then read `<consumerKey>` from the retrieved `.ecaGlblOauth` file. See
`force-app/main/default/connectedApps/MANUAL_SETUP.md` for the full mapping.

### Step 8 — Cutover for testers

Once verified, communicate the new creds to testers:

- Update `deepdive-cpq-workspace/SETUP-LAPTOP-2.html` — the credentials
  card in Module 5. Replace username / password / Instance URL / Expiration.
- Update `deepdive-cpq-workspace/OneDrive/Documents/DeepDive CPQ/SETUP-LAPTOP-2.html`
  if you keep the OneDrive copy.
- Commit + push the workspace repo.
- Slack the testers: "cpq-scratch is refreshed, expires <date>, creds updated
  in SETUP-LAPTOP-2.html — pull the workspace repo."
- Once every tester has migrated, delete the old scratch org:
  `sf org delete scratch --target-org cpq-scratch --no-prompt`

## Gotchas learned on 2026-07-29 (in order encountered)

1. **`AccountIndustry StandardValueSet` deploy error** — you can't insert
   standard value sets. Skip that folder.
2. **`DD_CPQ_MCP ConnectedApp` deploy error — RESOLVED 2026-08-04.** The cause
   was never scratch-org-specific: Salesforce disabled connected app creation
   org-wide and removed the New Connected App button, so the "do the manual UI
   setup" advice was impossible to follow. `connectedApps/` is now in
   `.forceignore` and the app ships as four External Client App directories
   that deploy in Wave 2. If you see this error again, something re-added
   `connectedApps` to a deploy — remove it, don't try to click through it.
3. **`DD_CPQ_Admin` + `DD_CPQ_Amendments` permset errors: "The Default
   parameter value doesn't exist" — RESOLVED 2026-08-04.** The external
   credential now declares the `Default` NamedPrincipal in source, so Wave 1
   creates it and both permsets deploy clean. **Do NOT strip the
   `externalCredentialPrincipalAccesses` block** (the old workaround) — that
   silently ships permsets which can't reach the Anthropic credential, so
   Compile / Ask AI / Ask the Warehouse fail at runtime with no deploy error.
   If the error reappears, Wave 1 didn't run before Wave 2.
4. **"Field does not exist: PricingEngineVersion__c on PolicyConstitution__c"
   when running Apex** — the user's assigned permset lacks FLS on that field.
   Even System Administrator profile does NOT auto-grant FLS on custom
   fields since Winter '23. `DD_CPQ_Selling` had incomplete grants; that's
   why `DD_CPQ_Admin` (with full FLS) is essential — hence the strip-and-
   redeploy dance in step 3.
5. **"No such relation 'TargetProduct__r' on entity 'UsageTier__c'"** — same
   FLS issue. Grants are on `DD_CPQ_Admin` only. Assign it before running seeds.

## Rotation schedule

- Every scratch org lasts 30 days.
- Refresh 7-10 days before expiration to leave a cutover window.
- Set a calendar reminder for `<expiration - 10 days>` when you create the org.
- Keep both orgs alive during cutover so testers finishing sessions on the
  old org aren't disrupted.

## Files this skill uses

- `config/project-scratch-def.json` — org shape (Developer edition, Quote enabled)
- `sfdx-project.json` — package config
- `force-app/main/default/**` — all metadata
- `scripts/apex/seed-pricing-model-demo.apex` — 5 pricing model quotes at qty=10
- `scripts/apex/seed-hybrid-charge-profile-demo.apex` — 4 hybrid products, 9 profile rows
- `scripts/apex/seed-microsoft-*.apex` — customer catalog for term discount + amendment demos
- `deepdive-cpq-workspace/SETUP-LAPTOP-2.html` — where to publish new tester creds
- `force-app/main/default/connectedApps/MANUAL_SETUP.md` — UI walkthrough for the ConnectedApp
