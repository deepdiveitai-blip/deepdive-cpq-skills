---
name: dd-cpq-case-resolver
description: >-
  Work the bug Cases testers file against DD CPQ, one at a time. Reads the
  Case, downloads and looks at its screenshots, then REPRODUCES the problem
  in the org before touching code — because a large share of reports turn
  out to be duplicated or orphaned configuration data rather than engine
  defects, and several Cases often share one root cause. Produces a verdict
  per Case with evidence, fixes what is a real defect, and never closes a
  Case on a claim it has not checked.
---

# Working a DD CPQ bug Case

Testers file Cases. Most describe a symptom accurately and diagnose it
wrongly — that is not a criticism, it is what a symptom looks like from the
UI. The job is to get from symptom to cause with evidence, and the single
most valuable outcome is discovering that five Cases are one bug.

## 1. Pull the queue

```bash
sf data query -o cpq-dev -q "SELECT CaseNumber, Subject, Status, Priority, Type, CreatedDate, Description FROM Case ORDER BY CaseNumber"
```

**Discard the noise first.** Cases whose Subject and Description are both a
string like `Aemail00DB0000000KlV9Aemail` are Salesforce email-to-case
artefacts, not reports. Count them separately and say so — do not work them,
and do not include them in "open bug" totals.

## 2. Read the attachments — they usually carry the real content

Several Cases have **no Description at all** and everything is in the
screenshot. Never skip them, and never infer the bug from the Subject line.

```bash
# list what is attached
sf data query -o cpq-dev -q "SELECT CaseNumber, (SELECT ContentDocument.Title, ContentDocument.FileExtension FROM ContentDocumentLinks) FROM Case"

# find the ContentVersion ids
sf data query -o cpq-dev -q "SELECT Id, Title FROM ContentVersion WHERE IsLatest = true"

# download one (VersionData is binary — curl it with the org token)
INFO=$(sf org display -o cpq-dev --json)
TOKEN=$(...accessToken from INFO...); URL=$(...instanceUrl...)
curl -s -H "Authorization: Bearer $TOKEN" \
  "$URL/services/data/v62.0/sobjects/ContentVersion/<068...>/VersionData" -o shot.png
```

Then **Read the .png**. Testers annotate them — red boxes and a sentence like
_"Parent quantity is 4, but child remains stuck at 1"_ is often the whole
diagnosis, and it is invisible from SOQL.

## 3. Reproduce before you believe it

Open the quote the Case links to and look at the live records. Two things
happen constantly:

- **The record has moved on.** Quotes get reworked between filing and
  triage, so the QuoteLineItem Ids quoted in the Case may no longer exist.
  That does not mean the bug is fixed — say plainly that the original
  evidence is gone and reproduce from scratch.
- **The data is the bug.** Before reading any Apex, check the configuration
  the Case depends on.

## 4. Suspect the data first — this is the highest-yield check

Run these before opening a single class. They have each already found a
live problem in `cpq-dev`:

```bash
# the same option attached to the same bundle twice
SELECT ParentProduct__c, ChildProduct__c, COUNT(Id) n FROM ProductOption__c
GROUP BY ParentProduct__c, ChildProduct__c HAVING COUNT(Id) > 1

# options with no parent at all
SELECT COUNT() FROM ProductOption__c WHERE ParentProduct__c = null

# the same bundle defined twice
SELECT Name, COUNT(Id) n FROM BundleDefinition__c GROUP BY Name HAVING COUNT(Id) > 1

# two parent lines for one bundle on one quote
SELECT Product2.Name, COUNT(Id) FROM QuoteLineItem WHERE QuoteId = '<id>'
AND ParentLine__c = null GROUP BY Product2.Name HAVING COUNT(Id) > 1
```

A duplicated `ProductOption__c` where one copy says `Static` and the other
`Match with Parent` produces exactly the "child quantity stuck at 1" report —
and it is a data fix, not an engine fix. Shipping a code change for it would
be wrong and would not help the next org.

## 5. Group before you fix

Lay the Cases side by side and look for one cause behind several. Strong
signals: the same Quote Id in more than one Case, the same product, or a
"UI glitch" Case and a "duplicate record" Case filed minutes apart — the
first is almost always the second seen from the front end.

Say so explicitly: _"#1030 is a symptom of #1031, not a separate defect."_
That is worth more than two fixes.

## 6. Verdict per Case

| Verdict               | Meaning                                                                               |
| --------------------- | ------------------------------------------------------------------------------------- |
| **CONFIRMED — code**  | Reproduced, and the fault is in Apex/LWC. Fix it.                                     |
| **CONFIRMED — data**  | Reproduced, caused by bad records. Fix the data; consider a guard so it cannot recur. |
| **DUPLICATE OF #n**   | Same root cause as another Case.                                                      |
| **CANNOT REPRODUCE**  | Say what you tried and what the org shows now.                                        |
| **WORKS AS DESIGNED** | Quote the rule or the documented behaviour.                                           |

## 7. Fix, prove, then report

Follow the repo's normal rules: `-d force-app` on every deploy, run the
affected tests, and add a regression test for anything confirmed as a code
defect — a fix without a test invites the same Case again next month.

Then post the finding to the Case and set Status. **Never close a Case on
reasoning alone** — attach the evidence: the query output, the number before
and after, or the passing test name.

If the fix is a data correction, also answer: _how did this data get in?_
A bundle saved twice through Bundle Builder is a UI defect worth its own
Case.
