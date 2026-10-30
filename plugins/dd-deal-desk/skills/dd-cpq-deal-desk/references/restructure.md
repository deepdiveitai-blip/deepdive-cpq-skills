# Rewrite, merge, split, transfer (`/rewrite`)

Contracts that have to be redone: two merged into one, a contract moved to
the parent company after an acquisition, part of a contract split off, or a
contract cancelled and replaced by new paper on new terms. Needs the
"Administer Pricing" permission (the Deal Desk connector has the tools; a
rep's connector answers 403 and the rep agent hands the job here).

## 1. Ground

- `dd_cpq_find_accounts`, `dd_cpq_list_contracts({accountId})`: the source
  contracts, with numbers, ends and lines. Several candidates: a numbered
  list.
- `dd_cpq_contract_lineage` on each source: has it been restructured
  before?
- The preset from the person's words:

  | They say                                        | Preset   | Key inputs                                                                                                    |
  | ----------------------------------------------- | -------- | ------------------------------------------------------------------------------------------------------------- |
  | "merge A and B", "one contract"                 | Merge    | `contractIds` (2+), `anchor` (end date: latest by default), `target.contractId` to keep one                   |
  | "move X to the parent", "after the acquisition" | Transfer | `contractIds`, `assetIds` (omit for all), `target.accountId`, `priceBasis` Hold or Reprice                    |
  | "split the clinics off"                         | Split    | `contractIds`, `parts[{assetIds, endDate}]`                                                                   |
  | "cancel and replace", "new paper from March"    | Rewrite  | `contractIds`, `effectiveDate`, `termMonths`, `holdLines`, `drop`, `quantities`, `add`, `earlyTerminationFee` |

  Anything the words leave open (which contract to keep, the effective date,
  hold or reprice, whether to waive the fee): ask, never pick.

## 2. Preview (always first)

`dd_cpq_restructure_preview` with those inputs. It writes nothing and returns
every move, credits, charges, any early-termination fee, net new ARR, churn,
bookings and whether approval is needed.

- `REWRITE_IS_RENEWAL`: the date is past the contract's end. Say so and send
  the person to `/renew`.
- A source contract with a change not yet in effect: the preview is refused
  with the first date that works. Report it as it is; never try another
  date or preset to get round it.
- `TRANSFER_NEEDS_RATE_PLAN`: the target lacks a plan in its currency. Name
  the products; an admin adds the plans.

## 3. The plan — "Restructure: <preset> <contracts>"

1. **Before**: each source contract, its end date and MRR.
2. **Moves**: one row per line: product, from (contract, quantity, price),
   to (contract, quantity, price), what happens (moved, cancelled, added,
   dropped).
3. **Settlement**: credits for the unused part, charges, the
   early-termination fee and whether it is waived (a waiver needs approval).
4. **After**: the new or kept contract(s), ARR, net new ARR, churn,
   bookings.
5. **Approvals** the preview names.

Then "Go?". Never start without the preview in this conversation.

## 4. One yes, then

`dd_cpq_restructure_start` with exactly the previewed inputs. It returns the
restructure quote. When the preview said approval is needed, offer
`dd_cpq_submit_for_approval` and stop there. Verify with
`dd_cpq_quote_snapshot`; leave a note.

Accept only after approval and the customer's signature, with
`dd_cpq_accept_amendment` (two yeses). Undo: `dd_cpq_restructure_reverse`
with `dryRun: true` first; it is refused once anything later touched those
lines.
