---
description: Check your DD CPQ connection and set up Salesforce's connector for contacts, tasks and opportunities
---

Follow the DD Quoting Agent rules (in the dd-cpq-quoting-agent skill). Call `dd_cpq_current_user` and tell the rep:

1. Who they are signed in as, and the org (name, and whether it is
   production, a developer org, a sandbox or a scratch org).
2. That DD CPQ is connected and what it lets them do (quotes, prices,
   renewals, approvals).
3. Whether Salesforce's own connector is connected: look for its tools
   (for example `soqlQuery`, `getObjectSchema`, or create/update record
   tools). It is needed for contacts, tasks, events and opportunity fields.
   If it is missing, give the steps from the result's
   `org.salesforceConnector.howToAdd`, with
   `org.salesforceConnector.mutationsUrl` for changes (and `readsUrl` if
   they also want reads). Mention that Salesforce may bill that connector's
   use in Flex Credits, so DD CPQ is used for everything it covers.

Change nothing.
