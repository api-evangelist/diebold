---
name: diebold-initiate-and-reverse-a-payment
description: Initiate a cardless or card-based payment through the DN Payment Initiation API, confirm it, and cancel or refund it using the transactionId as the idempotence key.
api: DN Payment Initiation API
operations:
  - initiateSession
  - creditBalance
  - authorizeCardlessPayment
  - cardlessPayment
  - cardbasedPayment
  - confirmPayment
  - transactionDetails
  - transactions
  - cancel
  - refund
  - pki
generated: '2026-09-06'
method: generated
source: openapi/diebold-dn-payment-initiation-api-openapi.yml
---

# Initiate a payment, and take it back

The PI-API is the one Diebold Nixdorf surface with an explicit replay key, and the one place the
contract tells you how to undo a write. Use it as written.

## Required headers on every call

- `X-Request-ID` — **required**, and the specification says it must be unique to the call, chosen by
  the initiating party. This is a correlation identifier, not an idempotence key; the contract does
  not say the server deduplicates on it.
- `Timestamp`.
- `PSU-IP-Address`, `PSU-User-Agent`, `PSU-GEO-Location` — optional, and lifted verbatim from the
  Berlin Group NextGenPSD2 header set. Send them when you have them; they are what a bank's fraud
  and SCA tooling expects to see.

Authentication is OpenID Connect (`openId`) or a bearer token, per
`authentication/diebold-authentication.yml`.

## Steps

1. **Optional: open a session for a wallet payment.**
   `POST /pay/{merchantId}/initiateSession` (`initiateSession`) for token-based payments such as
   Google Pay.

2. **Optional: check funds.** `POST /pay/{merchantId}/creditBalance` (`creditBalance`).

3. **Pay.**
   - Cardless: `POST /pay/{merchantId}/authorize` (`authorizeCardlessPayment`) to authorize a later
     payment, then `POST /pay/{merchantId}/cardless` (`cardlessPayment`).
   - Card-based: `POST /pay/{merchantId}/cardbased/{cardBasedTransactionType}` (`cardbasedPayment`).

4. **Confirm.** `PUT /pay/{merchantId}/confirm/{transactionId}` (`confirmPayment`).

5. **Check state.** `GET /payments/{merchantId}/{transactionId}` (`transactionDetails`) for one
   transaction, `GET /{merchantId}/payments` (`transactions`) for a set filtered by
   `fromDateTime` / `toDateTime`, `status` and `paymentProvider`. There is no pagination — the date
   range is your only bound.

## Reversal

Two operations take a payment back, and **both are idempotent on `transactionId`** — the spec states
"Idempotence key: transactionId" on each:

- `DELETE /payments/{merchantId}/{transactionId}` (`cancel`) — cancel an initiated payment.
- `PUT /payments/{merchantId}/{transactionId}` (`refund`) — refund an initiated payment.

Safe to retry with the same `transactionId`. **The contract states no window**: it does not say how
long after initiation a cancel is accepted, whether a refund is permitted after settlement, or how
partial refunds behave. Do not assume; get it in writing from the institution.

## Reading failures

Card-based errors return `CardBasedError`, which extends the standard error object with
`detailedResponseCode[]` and `transactionState`. `detailedResponseCode` carries card-verification
outcomes — CVV2 (`CURRENT_CVV2_VALID`, `CURRENT_CVV2_INVALID`, `CURRENT_CVV2_NOT_PROCESSED`,
`CVV2_UNVERIFIED`), CAVV/3-D Secure (`CURRENT_CAVV_VALID`, `CURRENT_CAVV_INVALID`, `CAVV_UNVERIFIED`),
AVS (`AVS_ZIP_MATCH_ADDRESS_MATCH` through `AVS_NOTHING_MATCHES`, `AVS_ZIP_RETRY`) and name
verification (`NVS_NAME_MATCH`, `NVS_PARTIAL_NAME_MATCH`, `NVS_NO_NAME_MATCH`). See
`errors/diebold-decline-codes.yml`.

Amounts follow ISO 20022: `value` is `CurrencyAndAmount` and `currency` is `ActiveCurrencyCode`.

## Device management

The same API registers the terminals that take these payments:
`POST /device/{merchantId}/{terminalId}` (`initializeDevice`), `PUT` (`updateDevice`),
`GET` (`getDevice`), `DELETE` (`deleteDevice`), and `GET /{merchantId}/devices` (`getDevices`).
`GET /pkiKey` (`pki`) returns the server's public RSA key.
