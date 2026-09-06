---
name: accountsiq-post-sales-invoice
description: Create and post a sales invoice into an AccountsIQ entity over the Integration 2.0 SOAP API, safely and without creating duplicates.
generated: '2026-09-06'
method: generated
source: wsdl/accountsiq-integration-2-0.wsdl; https://accountsiq.github.io/API-Wiki/troubleshooting.html
api: accountsiq:integration-2-0
operations:
- TokenGet
- TokenRefresh
- GetEntitiesByToken
- GetInvoicesByExternalReference
- GetNewSalesInvoice
- GetCustomer
- GetTaxCodeList
- SaveInvoiceGetBackInvoiceID
- PostInvoiceGetBackTransactionID
- GetInvoicePDFByTransactionID
- CancelInvoice
---

# Post a sales invoice to AccountsIQ

This posts real money into a real ledger. There is no idempotency mechanism on this API, so
the duplicate check in step 3 is not optional.

## Before you start

- Endpoint: `https://{region}.accountsiq.com/system/dashboard/integration/integration_2_0.asmx`
  where `{region}` is one of `eu1`, `eu2`, `uk1`, `us1`. The region comes from the customer's
  own AccountsIQ URL — never guess it.
- SOAP 1.1. Every response is a `WSResult2Of<T>`. Check `Status` and `ErrorCode` on **every**
  call: this API declares no SOAP faults, so a failure arrives as HTTP 200 with a null `Result`.

## 1. Authenticate

Call `TokenGet(clientId, clientSecret)`. Keep both `AccessToken` and `RefreshToken`.

Set the SOAP header on every subsequent call:

```
AiqSoapHeader:
  AccessToken: <AccessToken>
  Entity:      <the entity code you are posting into>
```

Each 2.0 operation also takes a token as its **first argument**. Leave it blank — it exists only
for 1.1 compatibility and the header is what is read.

If any response returns `HasExpired = true`, call `TokenRefresh(clientId, clientSecret,
refreshToken)` and retry the read. Do **not** blindly retry a write on expiry — see step 5.

## 2. Confirm the entity

Call `GetEntitiesByToken` and confirm the entity you were given is in the list. Posting to the
wrong entity of a group is the most expensive mistake available here and it will not error.

## 3. Check for a duplicate — do this before every post

Put your own stable key in `ExternalReference` (an order id, a source-system invoice id).
Then call `GetInvoicesByExternalReference` with that key.

If it returns an invoice, **stop**. The invoice already exists. AccountsIQ has no
`Idempotency-Key` header and does not enforce uniqueness on `ExternalReference`, so this
read-before-write is the only duplicate protection that exists. A retry without it posts a
second invoice.

## 4. Build the invoice from a server shell

Call `GetNewSalesInvoice` and populate the object it returns. Do not construct the object
yourself — the shell carries the entity's defaults and the fields the server expects.

Rules that cause most failures:

- `CustomerCode` must exist and must be **UPPER CASE**. Confirm with `GetCustomer` first.
- Server-assigned fields must be `xsi:nil="true"` — not `""`, and never the string `"N/A"`.
  That means `AuthorUserID`, `CreationDate`, `QuoteID` and `Status`.
- `CurrencyCode` is mandatory. Tax codes come from `GetTaxCodeList` — do not hardcode them.
- Do not set `RowVersionNumber` on a new object.

## 5. Save, then post

`SaveInvoiceGetBackInvoiceID` creates the invoice and returns its id.
`PostInvoiceGetBackTransactionID` commits it to the ledger and returns the transaction id.

Save is reversible in practice; **post is the point of no return**. Between the two, the
invoice exists but has not hit the ledger.

If a call times out or the connection drops after you sent it, do **not** retry. Go back to
step 3 and check `GetInvoicesByExternalReference` to find out whether it landed.

## 6. Confirm and retrieve

`GetInvoicePDFByTransactionID` returns the rendered document if you need to show or file it.

## Undoing it

- Not yet posted: `CancelInvoice`.
- Already posted: you cannot delete it. Raise a credit note — `GetNewSalesCreditNote`,
  `SaveCreditNoteGetBackCreditNoteID`, `PostCreditNoteGetBackTransactionID`. That is the correct
  accounting behaviour, not a workaround.

AccountsIQ publishes **no time window** for either. Do not tell a user they have N days.

## Volume

For more than a handful of invoices use the plural operations — `GetNewSalesInvoices`,
`CreateInvoicesGetBackInvoiceIDs`, `PostInvoicesGetBackTransactionIDs`. The provider states the
bulk methods support thousands of records a second, and they return per-row `DataErrorDTO`
entries so a partial failure names the offending record. There is no rate limit on this API.
