---
name: dd-cpq-dev-ready
description: >-
  On-demand developer for DD CPQ Jira work. Queries project DDCPQ for issues
  in status "DEV Ready", lists them, then builds the ones Venkat picks, one at
  a time: reads the story, moves it to In Progress, branches off
  feat/ddcpq-api-parity, builds LWC + REST + MCP within governor limits and
  AppExchange security, tests, deploys to cpq-dev, commits, pushes, opens a
  PR, comments the evidence on the issue and moves it to In Review. Never
  merges, never closes an issue, never deploys anywhere but cpq-dev.

  Trigger on: "/dd-cpq-dev-ready", "tackle dev ready", "what is in DEV
  Ready", "pick up the dev ready stories", "work DDCPQ-nn".

  Skip for filing issues (dd-cpq-issue-reporter) and tester bug Cases in
  Salesforce (dd-cpq-case-resolver).
---

# DEV Ready: on demand, one issue at a time

Venkat moves a Jira issue to **DEV Ready** when it is written well enough to
build. Nothing picks it up by itself. This skill runs only when he asks, from
Claude Code, and works the issues he names.

## Fixed facts

|             |                                                                                     |
| ----------- | ----------------------------------------------------------------------------------- |
| Jira site   | `deepdiveitsolutions.atlassian.net`, cloudId `f6a84dff-7c5f-4a00-8cf5-14db37af6fec` |
| Project     | `DDCPQ`                                                                             |
| Transitions | To Do `11` · In Progress `21` · In Review `31` · Done `41` · DEV Ready `42`         |
| Repo        | `deepdive-cpq-engine` only (CLAUDE.md rule 11)                                      |
| Base branch | `feat/ddcpq-api-parity`. PRs go there, never to `main`                              |
| Org         | `cpq-dev`, and only after its namespace reads `DDCPQ`                               |

If a Jira call answers "The app is not installed on this instance", the
connector has lost access. Stop and ask Venkat to re-authorise the Atlassian
connector. Do not work from memory of the story.

## 1. Pull the queue

```
searchJiraIssuesUsingJql
  jql: project = DDCPQ AND status = "DEV Ready" ORDER BY priority DESC, created ASC
  fields: summary, issuetype, priority, labels, parent
```

Show a short numbered list: key, type, summary, priority. Then:

- If Venkat named keys ("work DDCPQ-52"), do those, in that order.
- If he said "all", do them in list order, one at a time.
- Otherwise ask which. **Never start work he has not picked.**

An empty queue is an answer: say "Nothing is in DEV Ready" and stop.

## 2. Read the issue properly

`getJiraIssue` with `responseContentFormat: markdown` and `fields: ["*all"]`,
including comments. Stories follow Background → Acceptance criteria → Tech
details. The **acceptance criteria are the contract**. The tech details are a
starting point, often written before the code moved. Check every file, line
number and method they name against the repo before you trust it.

Stop and ask, with the issue left in DEV Ready, when:

- An acceptance criterion contradicts CLAUDE.md, or two criteria contradict
  each other.
- It needs a product decision the issue does not make.
- It asks for something CLAUDE.md rule 11 lists as cut.

## 3. Claim it

Transition `21` (In Progress). Create the branch from an up-to-date base:

```bash
git checkout feat/ddcpq-api-parity && git pull --ff-only origin feat/ddcpq-api-parity
git checkout -b feat/ddcpq-<nn>-<short-slug>
```

Several issues in one run may share one branch and one PR, **one commit per
issue**. Keep them apart if they change the same files for different reasons.

## 4. Design, if the issue needs it

A real design decision goes to the `design` agent first: schema, pipeline
stage, a new object, or "how should this work". Implement what it returns.
Routine changes and anything the issue already decided do not need it.

## 5. Build

Follow CLAUDE.md. The rules that most often bite:

- **Agentic-first:** a user-facing capability ships **LWC + REST + MCP**
  together. For REST, add an action on `/dd/v1/admin` (`CpqAdminRestResource`)
  or its own resource. Read actions go in `READ_ACTIONS`. For MCP, add a tool
  in `mcp-server/src/tools/`, register it in `index.ts`, add it to `readOnly.ts`
  if it writes nothing, and update the tool count in `mcp-server/README.md`.
- **Governor limits:** no SOQL or DML in loops. Measure SOQL, DML and CPU
  before and after on a real quote with anonymous Apex, and report both.
  Cart preview stays at or below 35 SOQL and 0 DML.
- **AppExchange security:** `with sharing`, `WITH USER_MODE` /
  `AccessLevel.USER_MODE`, `DdCpqAccess.require*`. Dynamic SOQL uses binds,
  and a new Apex class goes into the right permission set.
- **Namespace:** never write `DDCPQ__` in source.
- **LWC:** prefix CSS classes with the component's initials, put state on
  `data-*`, and keep text at 14px.
- **Apex reserved words** trip builds: `trigger`, `group`, `last`, and
  `json` as a parameter name.

## 6. Test and deploy

1. Jest for every changed LWC. vitest and `tsc --noEmit` for MCP changes.
2. Check the org: `sf data query -o cpq-dev -q "SELECT NamespacePrefix FROM Organization"` must return `DDCPQ`.
3. While iterating: `sf project deploy start -o cpq-dev --ignore-conflicts -l RunSpecifiedTests -t <tests> -d <each changed path>`.
   Always name paths with `-d`. An unfiltered deploy computes deletions.
   Deploy only changed classes. Deploying the whole classes folder is refused
   while scheduled jobs hold it.
4. Once, before pushing: the same paths with `-l RunLocalTests`. It must
   pass in full, including `EngineGoldenTest` ($101.83).
5. Check the acceptance criteria in the org itself (anonymous Apex, REST or
   MCP), not only in tests.

## 7. Commit, push, PR

- Commit message from the `summarize` agent. Refine it, then commit. The
  subject starts with the issue key (`DDCPQ-52: …`). Stage only this issue's
  files. The message ends with the Co-Authored-By line.
- `git push -u origin <branch>`, then
  `gh pr create --base feat/ddcpq-api-parity`. The PR body lists each
  commit, the measured numbers, and any way the build departs from the issue.
- **Never merge.** Venkat merges. Tell him to use "Create a merge commit".

## 8. Hand back on the issue

`addCommentToJiraIssue` (markdown), written for a junior tester:

- **Built and deployed to cpq-dev:** commit, PR link, test count.
- **What changed**, in plain words.
- **Numbers:** SOQL, DML and CPU before and after.
- **Differences from the story**, if any, and why.
- **How to test:** click-by-click steps on real records with real Ids. Never
  ask a tester to run seed data.

Then transition `31` (In Review). Never move an issue to Done. That is the
tester's call after retest.

If you could not finish, comment what you found and what blocks it. Move the
issue back to DEV Ready (`42`) if it only needs an answer, or To Do (`11`) if
it needs rewriting. Say which in the reply.

## 9. Report

One short block per issue: key, status now, PR, the numbers, and anything
Venkat must decide. Line 1 is STATUS or ACTION NEEDED.
