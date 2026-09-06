---
name: accountsiq-incremental-sync
description: Keep an external system in step with an AccountsIQ entity by polling for created and modified transactions, since AccountsIQ publishes no webhooks.
generated: '2026-09-06'
method: generated
source: wsdl/accountsiq-integration-2-0.wsdl
api: accountsiq:integration-2-0
operations:
- TokenGet
- TokenRefresh
- GetEntitiesByToken
- GetTransactionChangesBetween
- GetAccountsWithTransactionsCreatedBetween
- GetTransactionsCreatedByAccountBetween
- GetTransactionsModifiedBy
- GetCountTransactionsBy
- GetTransactionsBy
- GetAllocationsForTransactionsCreatedBetween
- GetAllocationsBetweenWithCreationDatePaged
- GetPeriodList
---

# Incrementally sync from AccountsIQ

**AccountsIQ has no webhooks and no event stream.** There is no AsyncAPI, no callback
registration, and nothing in the contract that pushes. Polling is the only option, and the
contract is built for it: creation date and modification date are exposed as separate filter
axes on the read operations.

## The axis that matters

Do not sync on transaction date. A transaction dated last month can be entered today.

- **Creation date** — when the record was entered. This is your sync watermark.
- **Transaction date** — the accounting date. This is business data, not a change signal.

`WSGetTransactionsByQuery` carries both `FromDate`/`ToDate` (transaction date) and
`FromCreationDate`/`ToCreationDate` (entry date). Drive your loop off the creation pair.

## 1. Authenticate and pick the entity

`TokenGet`, then set `AiqSoapHeader.AccessToken` and `AiqSoapHeader.Entity`.
`GetEntitiesByToken` lists what the token can reach — sync each entity separately, as there is
no cross-entity change feed.

## 2. Poll for changes

`GetTransactionChangesBetween` is the primary change feed: give it your last watermark and now.

Supporting reads when you need a narrower sweep:

- `GetAccountsWithTransactionsCreatedBetween` — which accounts moved, so you can skip the rest.
- `GetTransactionsCreatedByAccountBetween` — new transactions for one account.
- `GetTransactionsModifiedBy` — records changed by a given user.
- `GetAllocationsForTransactionsCreatedBetween` — payment matching that changed. Allocation is a
  separate change axis: an invoice's balance moves when a payment is allocated to it, and that
  is not a change to the invoice record.

## 3. Overlap the window

Advance the watermark to the **start** of the window you just fetched, not the end, and re-scan
a safety overlap on the next pass. Deduplicate on transaction id.

Records can be committed with a creation timestamp slightly behind the clock, so a
non-overlapping window silently drops rows. There is no cursor and no sequence number to detect
the gap with — this is why the overlap is necessary rather than merely careful.

## 4. Page the large sweeps

Most reads return everything. Three take explicit paging, and the `WSGetXxxByQuery` request
objects carry a `Skip` field:

- `GetAllocationsBetweenPaged`, `GetAllocationsBetweenWithCreationDatePaged` — `skip` + `limit`.
- `GetOrdersByPaged`.

Call `GetCountTransactionsBy` first to size the loop. For a first-run backfill, walk periods
from `GetPeriodList` rather than one enormous date range.

## 5. Check every response

`WSResult2Of<T>` reports errors in-band on HTTP 200. A sync loop that only checks HTTP status
will treat every failure as an empty page and advance its watermark over data it never read.
Check `Status` and `ErrorCode` before you commit the watermark, and only commit it after the
page is durably stored.

On `HasExpired = true`, call `TokenRefresh` and retry the same window. Reads are safe to retry —
this whole flow is read-only.

## Rate

There is no rate limit and no `Retry-After` signalling. The contractual constraint is fair use:
the API Terms of Use require that your application not degrade AccountsIQ's performance or
stability. Poll on a sane interval and keep the windows tight.
