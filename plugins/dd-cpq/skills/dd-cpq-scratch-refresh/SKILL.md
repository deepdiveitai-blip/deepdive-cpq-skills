---
name: dd-cpq-scratch-refresh
description: >-
  Rotate the DD CPQ dev scratch org before it expires: create the new
  namespaced org, deploy force-app and org-config, seed the trails and tester
  datasets, copy the hand-built catalogs, settings and pre-check rules from the
  old org, prove it with RunLocalTests and the GOLD replay, then cut over the
  Worker, the trail page, Jira and the skills. About half a day, most of it
  waiting.

  Trigger on: "rotate the org", "new scratch org", "the dev org is expiring",
  "refresh the org", "start the rotation".

  Skip for a tester's own sandbox or Developer Edition org (dd-cpq-sandbox-e2e),
  and for a package install check org (see the last section here).
---

# DD CPQ dev org rotation

Scratch orgs die on a fixed date (we create them for 30 days). Rotate **7–10
days before** the old one expires, and keep both alive until testers have moved.
This is the procedure that ran on 2026-11-01 (cpq-dev2 → cpq-dev3). Every step
was needed.

Commands are bash (git-bash on Windows), run from the root of
`deepdive-cpq-engine`. `OLD` and `NEW` are org aliases; set them first:

```bash
OLD=cpq-dev3 NEW=cpq-dev4
F=org-config/main/default/approvalProcesses/Quote.DD_CPQ_Needs_Review.approvalProcess-meta.xml
```

## Ground rules

- **Engine repo only** (CLAUDE.md rule 11). Every deploy names its directory
  with `-d`. Never deploy unfiltered.
- **Never print or store a password** in memory, chat, Jira or a file. Read it
  with `sf org display -o $NEW --verbose` when someone needs it.
- **Decisions go to Venkat one at a time**, as A/B. What to copy and when to
  cut over are his call. The defaults below are what he chose last time.
- **Nothing outside Jira:** Venkat tells the testers himself.

## 1. Create the org

```bash
sf org create scratch -v PartnerDev -f config/project-scratch-def.json -a $NEW -y 30 -w 20
sf data query -o $NEW -q "SELECT NamespacePrefix, Id FROM Organization"   # must say DDCPQ
sf org generate password -o $NEW --length 16
sf org display -o $NEW --verbose     # URL, username, org id, expiry. Do not paste the password anywhere.
```

Write down the org id (00D…), the My Domain (`https://<name>.scratch.my.salesforce.com`)
and the expiry date. Step 9 needs them.

## 2. Deploy

```bash
sf project deploy start -o $NEW -d force-app -l NoTestRun --ignore-conflicts --wait 60
sf org assign permset -o $NEW -n DD_CPQ_Engine_Admin
sf data create record -o $NEW --sobject CurrencyType --values "IsoCode=EUR ConversionRate=0.92 DecimalPlaces=2 IsActive=true"
```

Expect `Succeeded`, about 1,830 components. Add any other currency the old org
has: `sf data query -o $OLD -q "SELECT IsoCode, ConversionRate FROM CurrencyType"`.

**org-config** holds the Quote approval process, and that process names an
approver by username. The username in source belongs to a dead org, so the
deploy fails until you swap it. Swap it in the working copy only, then restore:

```bash
ME=$(sf org display -o $NEW --json | python -c "import sys,json;print(json.load(sys.stdin)['result']['username'])")
grep -n "@example.com" $F                                 # the approver in source
sed -i "s/[0-9_]*test-[a-z0-9]*@example.com/$ME/" $F
sf project deploy start -o $NEW -d org-config -l NoTestRun --ignore-conflicts --wait 30
git checkout -- org-config && git status --short          # must be clean
sf org assign permset -o $NEW -n DD_CPQ_Trail_FM
```

Expect about 29 components. Step 4 points the approver at Riya Rep.

## 3. Seed

Seeds come **before** the catalog copy (step 5). A seed that finds its product
already there may skip it or fail.

```bash
A=scripts/apex
sf apex run -o $NEW -f $A/seed-ddcpq-245-trail-qa-catalog.apex
for W in GOLD Venkat Santhosh Thirisha Abinash Darwin Rahul; do bash scripts/reset-ddcpq-256-dataset.sh $W $NEW; done
for f in seed-ddcpq-rules-trail seed-ddcpq-59-ramp-trail; do sf apex run -o $NEW -f $A/$f.apex; done
for i in 1 2 3 4 5 6; do sf apex run -o $NEW -f $A/seed-ddcpq-89-deal-blocks-trail.apex | grep -q "BLOCKS TRAIL: done" && break; done
sf apex run -o $NEW -f scripts/rotation/attributes.apex
for f in seed-ddcpq-promo-trail seed-ddcpq-attribute-rules-trail seed-ddcpq-field-map-trail \
         seed-ddcpq-239-quoting-agent-trail seed-ddcpq-242-plugin-trail seed-lifecycle-acme seed-lifecycle-newbusiness; do
  echo "== $f"; sf apex run -o $NEW -f $A/$f.apex | grep -E "USER_DEBUG|Error|Exception" | tail -4
done
```

- **The deal-blocks seed** does not always finish in one run, and a rerun
  picks up where it stopped. Repeat it until it prints `BLOCKS TRAIL: done`.
- **The 256 datasets** run each step as its own Apex run. If the network
  drops, one quote is left empty; check the `report` lines. To redo a single
  step:
  `sed "s/{{WHO}}/Venkat/; s/{{STEP}}/priceBulk/" $A/seed-ddcpq-246-rep-certification.apex > /tmp/rc.apex && sf apex run -o $NEW -f /tmp/rc.apex`
- **attributes.apex** creates the Color, Deployment and Warranty attributes
  that the attribute-rules and field-map seeds expect.

## 4. Users and test settings

```bash
sf apex run -o $NEW -f scripts/rotation/test-users.apex            # Riya Rep + Nora NoCPQ
RIYA=$(sf data query -o $NEW -q "SELECT Username FROM User WHERE Alias='riyarep'" --json | python -c "import sys,json;print(json.load(sys.stdin)['result']['records'][0]['Username'])")
sed -i "s/[0-9_]*test-[a-z0-9]*@example.com/$RIYA/" $F
sf project deploy start -o $NEW -d $F -l NoTestRun --ignore-conflicts --wait 10
git checkout -- org-config
sf project deploy start -o $NEW --metadata-dir scripts/rotation/apex-settings --wait 10   # parallel Apex testing OFF
```

Riya Rep is a rep with `DD_CPQ_Engine_User` and the Quote approver. Nora NoCPQ
has no DD CPQ access, for the "no access" cases. Parallel testing is turned
off to match the old org, which is where RunLocalTests passes in full.

## 5. Copy what was hand-built in the old org

Ask Venkat first: **(A) copy the hand-built catalogs, settings and pre-check
rules** (chosen 2026-11-01), or (B) seeds only. Quotes are never copied. The
GOLD quotes are rebuilt in step 7.

```bash
python scripts/rotation/copy_org_data.py catalog   $OLD $NEW --dry   # counts only
python -u scripts/rotation/copy_org_data.py catalog $OLD $NEW        # 54 products last time
python scripts/rotation/copy_org_data.py settings  $OLD $NEW
python scripts/rotation/copy_org_data.py prechecks $OLD $NEW
```

- **catalog** copies every product the new org lacks, with what hangs off it.
- **settings** keeps `Long_Stop_Default_Days` at the new org's default.
- **prechecks** drops reference fields (OwnerId can't cross orgs).
- All three refuse unless both orgs carry the same namespace, and a second run
  copies nothing new.

**If the catalog copy dies partway**, delete what it created since it started,
then rerun. Its retries cover dropped connections, but not a laptop going to
sleep. Then prove the copy: price the same 10–12 copied products in both orgs
inside a rolled-back savepoint (`CpqEngine.previewDrafts`), and the net prices
must match.

## 6. Prove it

```bash
sf apex run test -o $NEW -l RunLocalTests -w 60
```

Every test passes (1,941 on 2026-11-01), including `EngineGoldenTest`
($101.83). A failure here is a real defect or a missed step. Do not move on.

## 7. Rebuild GOLD

`mcp-server/tests/goldReplay.test.ts` replays the GOLD rep-certification
missions against fixed record ids, and those ids now point at the dead org.

1. Run the GOLD missions on the new org through the MCP tools (Build, West,
   Riverside, Partner, Nordhavn and the rest). Use the GOLD dataset from step 3.
2. Check every number against the mission answer sheet.
3. Put the new ids into `goldReplay.test.ts` and run `npx vitest run tests/goldReplay.test.ts`
   from `mcp-server/`. It must pass in full.
4. Open a PR (DDCPQ-254 style).

## 8. Retest records for open stories

For each story In Review whose test steps name records in the old org, build
the same records in the new org. Post a Jira comment with the new quote numbers
and links. Don't edit the story text.

## 9. Cut over (Venkat chooses when; default: once steps 1–7 pass)

**Worker** (`mcp-server/worker/wrangler.jsonc`), on a branch from `feat/ddcpq-api-parity`:

- `SF_LOGIN_URL` = the new My Domain.
- `ALLOWED_ORG_IDS` = new org id **plus** the old one until it expires.

Then `npm run typecheck && npm test` in `mcp-server/worker`, `npx wrangler deploy`,
and open a PR. Write down the old Worker version for rollback
(`npx wrangler deployments list`).

**Trail page** (the rep-certification trail artifact):

- Update `ORG`, `MYDOMAIN`, the org notice text, and the `fc-datasets` JSON
  (quote numbers and ids per tester, read from the new org).
- Republish it to the same URL.

**Jira:**

- Swap old quote numbers and the org name for the new ones **in place** in
  the rep-certification stories and sub-tasks (272–278 and their sub-tasks).
  Never regenerate them, because some carry hand-added rows.
- Afterwards, search the stories for the old org alias and old quote numbers.
  Nothing should be left.

**Skills and memory:**

- Every skill that names the dev org changes to the new alias
  (`grep -rn "cpq-dev" .claude/skills`). Leave lines that describe history.
- Update the workspace memory line and the rotation memory.
- Skills are republished only after Venkat merges.

**After the old org expires:**

- Remove its id from `ALLOWED_ORG_IDS` with the next Worker change.

## Package install check org (separate, when a new beta is built)

The installed-package org (`cpq-pkgN`) uses a **single-currency, no-namespace**
definition, so it proves the package installs on a plain customer org:

```json
{ "orgName": "DD CPQ Package Install Check", "edition": "Developer",
  "settings": { "lightningExperienceSettings": { "enableS1DesktopEnabled": true },
                "quoteSettings": { "enableQuote": true } } }
```

- **A beta does not upgrade to the next beta.** Delete the
  `DDCPQ__DD_CPQ_Engine_Admin` permission-set assignments, uninstall the old
  version, install the new one, then reassign.
- **Package build tests run as a user with no DD CPQ permission set.** Wrap
  them in `System.runAs(DdCpqTestAccess.sharedEngineUser())`.
- Then run the packaged golden test on the installed org.

## Gotchas, in the order they bit

1. **The org-config approver is a username from a dead org.** Swap it in the
   working copy, deploy, and `git checkout` (step 2). Never commit the swap.
2. **Seed before you copy.** The copy skips products that already exist by
   name, but seeds don't check for copied products.
3. **Anonymous Apex from git-bash:** heredocs mangle backslashes and quotes.
   Generate Apex from Python (as `copy_org_data.py settings` does), or write
   the file with an editor.
4. **Apex reserved words** (`trigger`, `on`, `group`) break hand-written
   anonymous Apex, and the error doesn't point at the cause.
5. **Network drops mid-copy** leave half a catalog. Delete what was created
   after the start time, then rerun. Don't patch around it.
6. **`dd_cpq_start_renewal` `exclude` takes asset ids**, not product ids, when
   you rebuild the GOLD renewal mission.
7. **The constitution step in old copies of this skill is a cut feature**
   (CLAUDE.md rule 11). The engine has no constitution. Never insert one.
