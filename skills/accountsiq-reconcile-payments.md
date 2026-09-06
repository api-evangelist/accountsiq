---
name: accountsiq-reconcile-payments
description: Record payments and receipts in AccountsIQ and allocate them against open invoices, including how to reverse an allocation.
generated: '2026-09-06'
method: generated
source: wsdl/accountsiq-integration-2-0.wsdl; https://accountsiq.github.io/API-Wiki/troubleshooting.html
api: accountsiq:integration-2-0
operations:
- TokenGet
- GetBankList
- GetBankAccountBalance
- GetCustomer
- GetSupplier
- GetInvoicesByCustomerCode
- GetInvoicesBySupplierCode
- GetAccountBalanceInformation
- SaveSalesReceiptGetBackTransactionID
- SavePurchasePaymentGetBackTransactionID
- SaveSundryReceiptPaymentsGetBackTransactionIDs
- AllocateTransactions
- AllocateTransactionsWithDiscount
- GetAllocationsByTransactionId
- UnallocateTransactions
- PostPayAndAllocateSalesInvoice
- DisputeTransactions
- PostBankTransfer
---

# Reconcile payments in AccountsIQ

Recording money and matching it to an invoice are **two separate operations**. A receipt that is
saved but never allocated leaves the invoice showing as unpaid and the customer's balance wrong.

## 1. Set up

`TokenGet`, then `AiqSoapHeader.AccessToken` + `Entity`.

`GetBankList` gives you the bank accounts. Two different codes are easy to confuse and the
provider calls this out explicitly:

- **BankCode** — the General Ledger code representing the bank in AccountsIQ.
- **BankAccountCode** — the physical account number at the branch.

Using one where the other belongs is a common failure.

## 2. Find what is open

- Customers: `GetInvoicesByCustomerCode`, `GetCustomer`, `GetCustomerBalanceInformation`.
- Suppliers: `GetInvoicesBySupplierCode`, `GetSupplier`, `GetSupplierBalanceInformation`.
- Either side: `GetAccountBalanceInformation`.

## 3. Record the money

- Money in: `SaveSalesReceiptGetBackTransactionID`.
  Mandatory: `BankAccountCode`, `CheckReference`, `CustomerCode`. Omitting any of the three is
  the documented cause of sales-receipt errors.
- Money out: `SavePurchasePaymentGetBackTransactionID`.
- Several at once: `SaveSundryReceiptPaymentsGetBackTransactionIDs`.
- Between your own accounts: `PostBankTransfer` — a transfer, not a receipt. Do not allocate it.

Keep the returned transaction id. You need it to allocate.

## 4. Allocate

`AllocateTransactions` matches the payment transaction to one or more invoice transactions.

Use `AllocateTransactionsWithDiscount` when a settlement discount is being taken, so the
discount posts to the right account instead of leaving a residue on the invoice.

`PostPayAndAllocateSalesInvoice` (and its plural `PostPayAndAllocateSalesInvoices`) does post,
pay and allocate in one call. Prefer it for paid-at-point-of-sale flows: fewer round trips and
no window in which the invoice sits unallocated.

## 5. Verify

`GetAllocationsByTransactionId` shows what the transaction is matched against. Check it —
allocation can partially succeed across multiple invoices.

## Reversing

`UnallocateTransactions` undoes a match. This is a genuine undo and it is the reversal for this
flow. AccountsIQ publishes **no time window** for it; do not tell a user one exists.

Unallocating does not remove the payment — the money stays recorded and becomes unmatched again.
To reverse the money itself you need a compensating entry, not a delete.

`DisputeTransactions` flags a transaction as disputed without moving any money. Use it when a
customer contests an invoice: it is a state change, not a reversal.

## Retry safety

There is no idempotency on this API. A retried `SaveSalesReceiptGetBackTransactionID` records a
second receipt. If a call fails without a clear answer, read back with
`GetInvoicesByCustomerCode` or `GetAllocationsByTransactionId` before trying again — never retry
blind.
