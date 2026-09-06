---
name: diebold-atm-cash-withdrawal
description: Run a self-service cash withdrawal through the DN Open Backend API's two-phase authorize/finalize protocol, and reverse it when the dispense fails.
api: DN Open Backend API
operations:
  - getToken
  - consumerIdentify
  - accountList
  - accountInquiry
  - pinCheck
  - withdrawalAuthorize
  - withdrawalFinalize
  - reversal
  - getAccountWarnings
generated: '2026-09-06'
method: generated
source: openapi/diebold-dn-open-backend-api-openapi.yml
---

# Withdraw cash at a self-service terminal

The Open Backend API is how a Diebold Nixdorf terminal asks a core banking system whether it may
dispense. The protocol is deliberately two-phase: the core authorizes, the terminal dispenses, the
terminal confirms. If you collapse the two, you will hand out money the core never committed.

## Steps

1. **Get a token.** See `diebold-authenticate-transaction-middleware`.

2. **Identify the consumer.** `POST /consumerIdentify` (`consumerIdentify`). Card data, including
   EMV tag data via the `EMVTagData` / `EMVTags` schemas, is carried here.

3. **List and inspect accounts.** `POST /accountList` (`accountList`) then `POST /accountInquiry`
   (`accountInquiry`) for balances. `POST /getAccountWarnings` (`getAccountWarnings`) and
   `POST /getConsumerWarnings` surface holds, dormancy and other conditions before you offer an
   amount the core will refuse.

4. **Verify the PIN.** `POST /pinCheck` (`pinCheck`). A failure returns
   `extendedResponseCode: PIN_VALIDATION_FAILED`, and repeated failures return
   `PIN_TRY_LIMIT_EXCEEDED` — these are distinct outcomes and the terminal should treat them
   differently.

5. **Authorize.** `POST /withdrawalAuthorize` (`withdrawalAuthorize`). The core either approves or
   returns a business outcome in `extendedResponseCode`. Several of those are not declines but
   requests for consent — `RESPONSE_FEE_CONFIRM_REQUIRED`, `OVERDRAFT_CONFIRM_REQUIRED`,
   `FCC_CONFIRM_REQUIRED`, `DIRECT_CURRENCY_CONVERSION_CONFIRM_REQUIRED`, `MULTI_CONFIRM_REQUIRED`.
   Show the consumer the fee or the conversion and re-authorize with their confirmation.

6. **Dispense, then finalize.** Only after the cash leaves the machine, call
   `POST /withdrawalFinalize` (`withdrawalFinalize`).

7. **Reverse a failed dispense.** If the dispense fails or the finalize never lands, call
   `POST /reversal` (`reversal`). This is the operation that stops a customer being debited for
   money they never received.

## What the contract does not tell you

- **No idempotency.** Neither `withdrawalAuthorize` nor `withdrawalFinalize` declares a replay key.
  If you retry a finalize after a timeout you have no contractual guarantee it will not double-post.
  Agree the deduplication behaviour with the institution before going live, and use the `referenceId`
  field consistently so a reconciliation is possible either way.
- **No reversal window.** The contract does not say how long after a withdrawal a `reversal` is
  accepted, or what happens if you send it twice. Ask.
- **No rate limits and no 429.** Sizing is the institution's, not the vendor's.
