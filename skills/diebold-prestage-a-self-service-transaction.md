---
name: diebold-prestage-a-self-service-transaction
description: Stage a cardless cash-out or deposit from a mobile banking app with the DN Online & Mobile API, execute it at the terminal, and cancel it if the customer never arrives.
api: DN Online & Mobile API
operations:
  - getAccounts
  - getAccountDetails
  - createCashout4Me
  - createCashout2You
  - createDeposit
  - createBranchDeposit
  - getPrestagedTransactions
  - executePrestagedTransaction
  - deletePrestagedTransaction
  - createPushPayment
  - cancelPushPayment
  - registerForRemoteNotifications
generated: '2026-09-06'
method: generated
source: openapi/diebold-dn-online-mobile-api-openapi.yml
---

# Prestage a transaction in the app, complete it at the machine

The Online & Mobile API is the digital half of a self-service journey: the customer sets a
transaction up on their phone, then finishes it at an ATM or in a branch.

## Steps

1. **Authenticate.** Bearer token, per `diebold-authenticate-transaction-middleware`. This API also
   supports an OpenID Connect arrangement where the access token goes in `Authorization` and an
   encrypted ID token goes in the `consumerIdParam` parameter. Keys come from `GET /getRsaKey`
   (`getRsaKey`) and `GET /getEcKey` (`getEcKey`).

2. **Show the customer their accounts.** `GET /getAccounts` (`getAccounts`),
   `GET /getAccountDetails` (`getAccountDetails`), `GET /getAccountTypes` (`getAccountTypes`).
   `GET /checkAccount` (`checkAccount`) validates a destination before you offer it.

3. **Stage the transaction.**
   - `POST /createCashout4Me` (`createCashout4Me`) — cash for the account holder themselves.
   - `POST /createCashout2You` (`createCashout2You`) — cash for someone else.
   - `POST /createDeposit` (`createDeposit`) or `POST /createBranchDeposit` (`createBranchDeposit`).

4. **Find a machine.** `POST /getLocations` (`getLocations`).

5. **List and execute.** `GET /getPrestagedTransactions` (`getPrestagedTransactions`) at the
   terminal, then `POST /executePrestagedTransaction` (`executePrestagedTransaction`).

6. **Cancel an unused stage.** `POST /deletePrestagedTransaction` (`deletePrestagedTransaction`).
   This is the reversal path for step 3 — the contract states no expiry for a staged transaction, so
   treat cleanup as your responsibility, not the platform's.

7. **Notify.** `POST /registerForRemoteNotifications` (`registerForRemoteNotifications`) and
   `POST /unregisterForRemoteNotifications` for push updates.

## The SEPA Instant surface

Separate from prestaging, this API carries a real-time payment surface:

- `POST /sepa-instant/pushPayment` (`createPushPayment`), `GET` (`getPushPayment`),
  `DELETE` (`cancelPushPayment`).
- `POST /sepa-instant/pullPayment` (`authorizePullPayment`), `GET` (`getPullPayment`),
  `DELETE` (`cancelPullPayment`).

Each create has a matching cancel. Neither declares an idempotence key and neither states a
cancellation window — see `conventions/diebold-conventions.yml`.

## Errors

This specification publishes a full numeric error table inside `components.schemas.Error.description`:
codes are allocated per operation in the `287xxxx` range, where each operation owns a 100-code block
and the last two digits identify the family (`00` SERVER_FAILURE, `01` SYSTEM_ERROR, `02`
REQUEST_DATA_INVALID, `03` NO_RESULT, `04` HOST_OFFLINE, `05` HOST_COMMUNICATION_ERROR, `15`
VERIFICATION_FAILED). All 162 parsed codes are in `errors/diebold-problem-types.yml`. Match on the
code, not on the message string.
