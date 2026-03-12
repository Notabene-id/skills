---
name: notabene-api
description: >
  Notabene API integration guide for developers and coding agents. Use this skill whenever
  anyone is building an integration with Notabene, writing code against Notabene APIs,
  debugging an integration, or asking how Notabene works technically. Triggers on: "integrate
  with Notabene", "Notabene API", "Travel Rule API", "Flow API", "pay-in", "VASP integration",
  "originator", "beneficiary VASP", "PIA", "PRA", "infrastructure provider", "notabene
  webhook", "authorize transfer", "settle transfer", "append PII", "IVMS101", "CAIP-19",
  "TAP", "DIDComm", "entityDID", "notabene auth token", "SafeConnect", "WithdrawalAssist",
  "DepositAssist", "DepositRequest", "ConnectWallet", "customer token", "javascript sdk",
  "embedded component", or any question about sending or receiving payments or travel rule
  data via Notabene. Use proactively whenever a developer writes code calling Notabene
  endpoints.
---

# Notabene API Integration Guide

Notabene provides two primary APIs. Read the relevant guide(s) before writing any code.

---

## Which guide do you need?

| Use Case | Guide |
|---|---|
| Travel Rule compliance (VASP-to-VASP PII exchange, outgoing/incoming crypto transfers) | [guide-travel-rule-api.md](./references/guide-travel-rule-api.md) |
| Flow pay-ins (stablecoin/fiat pull payments with merchant invoicing, payment links) | [guide-flow-payins-api.md](./references/guide-flow-payins-api.md) |
| Embedding Travel Rule UI in your frontend (withdrawal screen, deposit screen, wallet verification) | [guide-javascript-sdk.md](./references/guide-javascript-sdk.md) |

If you're unsure: **Travel Rule** = you are a VASP moving customer funds between blockchain addresses and need to comply with FATF regulations. **Flow** = you are building a payment product where a merchant requests money from a customer. **JavaScript SDK** = you need a UI component that collects Travel Rule data from your users inside your web app.

> **Flow and Travel Rule:** Flow transactions automatically feed into Notabene's Travel Rule compliance workflows. If your primary use case is payment flows via Flow, you do **not** need to separately integrate the Travel Rule API — compliance is handled for you as part of the Flow protocol.

---

## Where to integrate: Travel Rule

### Sending crypto (outgoing transfers)

Integrate at these points in your platform:

- **Withdrawal screen** on a crypto exchange
- **Crypto arm of a fiat onramp** — after converting fiat to crypto, before sending to the customer's wallet
- **Send transaction** functionality in a custodial wallet or server API
- **Post-trade settlement** — settling tokens after an OTC or exchange trade

**Critical:** Call `POST /entities/{entityDID}/tx` **before** broadcasting the on-chain transaction. Wait for the `notification.transferStatusChanged` webhook with `toStatus: "AUTHORIZED"` before sending funds. Never send on-chain until you receive this status.

If your compliance policy requires collecting beneficiary info from your customer before authorization, use the **Withdrawal Assist** component from the [JavaScript SDK](./references/guide-javascript-sdk.md) to embed a compliant data-collection UI at the point of withdrawal.

### Receiving crypto (incoming transfers)

Integrate at these points:

- **Deposit flow** on a crypto exchange
- **Crypto arm of a fiat offramp** — before converting received crypto to fiat
- **Receiving funds** in a custodial wallet or server API

In most cases you will receive the travel rule message **before** funds arrive on-chain. As a developer your primary responsibilities are: confirming address ownership, verifying beneficiary name against your KYC records, and processing the authorization decision.

If additional data needs to be collected from a depositing customer, use the **Deposit Assist** component. To improve deposit UX and reduce incoming data gaps, add the **Deposit Request** component to your deposit screen — it enriches your QR code so senders' VASPs can pre-fill transfer data.

---

## Core Concepts (both APIs)

**Authentication** — OAuth2 client credentials, token valid 24 hours:
```
POST https://auth.notabene.id/oauth/token
{ "client_id", "client_secret", "grant_type": "client_credentials",
  "audience": "https://api.eu1.notabene.id" }
```
Base URL for all REST calls: `https://api.eu1.notabene.id`
Dashboard: `https://app.eu1.notabene.id/`

> The JavaScript SDK uses a separate **customer token** (not the API access token) generated server-side per user. See [guide-javascript-sdk.md](./references/guide-javascript-sdk.md) for details.

**Webhooks** — Notabene uses [Svix](https://svix.com/) for delivery. Configure endpoints in the dashboard. Always verify the `svix-signature` header before processing. See the travel rule guide for the verification snippet.

**Webhook payload format** — All webhooks use the same envelope:
```json
{
  "message": "flow.payout.created",
  "payload": {
    "id": "423ae44a-...",
    "for": "did:web:notabene.id:pg",
    "amount": "1300",
    "currency": "USD"
  },
  "version": "1.0.0"
}
```
- Event type is in the `message` field (NOT `eventType`)
- Transfer ID is in `payload.id` (NOT a top-level `id`)
- All event-specific data is nested inside `payload`

**DIDs (Decentralized Identifiers)** — Notabene identifies every actor with a DID:
- Your entity: `did:web:your-domain.com` (configured in dashboard)
- Customers: `did:email:user@example.com` or `did:pkh:eip155:1:0xAddress`
- Blockchain addresses: `did:pkh:eip155:<chainId>:<address>`

**CAIP-19** — Asset identifiers used throughout both APIs:
- ETH mainnet USDC: `eip155:1/erc20:0xA0b86991c6218b36c1d19D4a2e9Eb0cE3606eB48`
- Polygon USDC: `eip155:137/erc20:0x2791Bca1f2de4661ED88A30C99A7a9449Aa84174`
- For fiat rails: PayTo URIs, e.g. `payto://iban/DE89370400440532013000`

**Entity DID** — almost every API path includes `{entityDID}`. This is your organization's DID, found in the Notabene dashboard under Settings.

---

## Travel Rule API — Quick orientation

Read [`references/guide-travel-rule-api.md`](./references/guide-travel-rule-api.md) for the complete guide.

**Two roles:**
- **Originator VASP** — creates outgoing transfers, provides PII when requested, settles
- **Beneficiary VASP** — receives incoming transfers, confirms address ownership, verifies beneficiary name, authorizes or rejects

**Core lifecycle (originator):**
1. `POST /entities/{entityDID}/tx` — create transfer
2. Listen for `tap.requirePresentationRequested` → `POST .../transfers/{id}/append` with IVMS101 PII
3. Wait for `notification.transferStatusChanged` with `toStatus: "AUTHORIZED"` before going on-chain
4. `POST .../tx/{id}/settle` — report on-chain tx hash

**Core lifecycle (beneficiary):**
1. Register deposit addresses via `POST /entities/{entityDID}/relationship` (recommended)
2. Listen for `tap.requirePresentationSatisfied` → GET transfer with `?decrypt=true`, run name match
3. Complete checks: `/checks/beneficiary-name-matching` and `/checks/internal-checks`
4. Authorize automatically via configured policies, or reject if checks fail

---

## Flow Pay-ins API — Quick orientation

Read [`references/guide-flow-payins-api.md`](./references/guide-flow-payins-api.md) for the complete guide.

> **Travel Rule compliance is automatic.** Flow transactions are built on TAP and automatically feed into Notabene's Travel Rule compliance workflows. If you are primarily focused on payment flows, you do not need to separately integrate the Travel Rule API.

**Three roles:**
- **PIA (Payment Initiating Agent)** — merchant side; creates pay-ins, distributes payment links, reconciles
- **PRA (Payment Responding Agent)** — payer side; receives requests, surfaces to customer, authorizes, settles
- **IP (Infrastructure Provider)** — optional; custody/wallet/liquidity layer that handles on-chain settlement

**Core lifecycle (PIA):**
1. Register merchants: `POST /entity/{entityDID}/flow/customers`
2. Create pay-in: `POST /entity/{entityDID}/flow/customers/{merchantDID}/pay-ins`
3. Send `paymentLink` to customer
4. Authorize when ready: `POST .../tx/{id}/authorize`
5. Reconcile on `notification.transferStatusChanged` → `SETTLED`

**Core lifecycle (PRA):**
1. Receive `flow.payin.created` webhook
2. Optionally post `authorization_required` URL for customer approval
3. Authorize with asset selection: `POST .../tx/{id}/authorize` + `{ "settlement_address": "eip155:1:0x..." }`
4. Execute settlement and report: `POST .../tx/{id}/settle`

---

## JavaScript SDK — Quick orientation

Read [`references/guide-javascript-sdk.md`](./references/guide-javascript-sdk.md) for the complete guide.

The SDK provides embeddable **SafeConnect** components that handle Travel Rule data collection in your frontend. Install via `npm install @notabene/javascript-sdk`.

**Key components:**

| Component | Where to use |
|---|---|
| `createWithdrawalAssist` | Withdrawal screen — collects beneficiary info before transfer creation |
| `createDepositAssist` | Post-deposit — collects missing originator info from depositing customer |
| `createDepositRequest` | Deposit screen — generates enriched QR code for Travel Rule-compliant deposits |
| `createConnectWallet` | When wallet ownership proof is required for an unhosted wallet |

**Critical pattern:** Generate a `customerToken` server-side (never expose your API access token to the frontend), initialize `new Notabene({ nodeUrl, authToken: customerToken })`, render the component, then use the `txCreate` field from the response as the body for `POST /entities/{entityDID}/tx`.

---

## Developer checklist before writing code

- [ ] API credentials generated in the Notabene dashboard
- [ ] Webhook endpoint registered and signature verification implemented (Svix)
- [ ] Entity DID noted (used in every API path)
- [ ] Correct guide read end-to-end before implementation
- [ ] If using the JS SDK: customer token generation endpoint built on your backend
- [ ] "Where to integrate" integration points identified and sequenced correctly

---

## References

- **Full API reference**: [devx.notabene.id/reference](https://devx.notabene.id/reference)
- **SafeConnect components docs**: [devx.notabene.id/docs/components-overview](https://devx.notabene.id/docs/components-overview)
- **Travel Rule guide**: [`references/guide-travel-rule-api.md`](./references/guide-travel-rule-api.md)
- **Flow Pay-ins guide**: [`references/guide-flow-payins-api.md`](./references/guide-flow-payins-api.md)
- **JavaScript SDK guide**: [`references/guide-javascript-sdk.md`](./references/guide-javascript-sdk.md)
