# Travel Rule API Guide

This guide covers the essential API endpoints for Travel Rule compliance using Notabene Transact V2. It focuses on the strictly required APIs for creating outgoing transfers, handling incoming transfers, and managing the authorization lifecycle.

For the full API reference, see [devx.notabene.id/reference](https://devx.notabene.id/reference).

## Table of Contents

- [Authentication](#authentication)
- [Webhooks Setup](#webhooks-setup)
- [Where to Integrate](#where-to-integrate)
  - [When You Are Sending Crypto (Outgoing)](#when-you-are-sending-crypto-outgoing)
  - [When You Are Receiving Crypto (Incoming)](#when-you-are-receiving-crypto-incoming)
- [Outgoing Transfers (Originator)](#outgoing-transfers-originator)
  - [Create a Transfer](#1-create-a-transfer)
  - [Provide PII When Requested](#2-provide-pii-when-requested)
  - [Settle the Transfer](#3-settle-the-transfer)
- [Incoming Transfers (Beneficiary)](#incoming-transfers-beneficiary)
  - [Confirm Address Ownership](#1-confirm-address-ownership)
  - [Verify Beneficiary Name](#2-verify-beneficiary-name)
  - [Reject a Transfer](#3-reject-a-transfer)
- [Transfer Statuses](#transfer-statuses)
- [Webhook Event Reference](#webhook-event-reference)

---

## Authentication

### 1. Obtain API Credentials

Generate credentials from the Notabene dashboard at `https://app.eu1.notabene.id/` under Settings > API Credentials.

### 2. Generate an Access Token

```
POST https://auth.notabene.id/oauth/token
Content-Type: application/json
```

```json
{
  "client_id": "<your_client_id>",
  "client_secret": "<your_client_secret>",
  "grant_type": "client_credentials",
  "audience": "https://api.eu1.notabene.id"
}
```

**Response:**

```json
{
  "access_token": "eyJhbGc...",
  "expires_in": 86400,
  "token_type": "Bearer"
}
```

The token is valid for **24 hours**. Refresh on expiry.

### 3. Use the Token

Include it in all API requests:

```
Authorization: Bearer <access_token>
```

**Base URL:** `https://api.eu1.notabene.id`

---

## Webhooks Setup

Notabene uses [Svix](https://svix.com/) for webhook delivery with automatic retries and signature verification.

### Configure Endpoints

Set up webhook endpoints in the Notabene dashboard under Settings. Subscribe to the events relevant to your integration (see [Webhook Event Reference](#webhook-event-reference)).

### Verify Signatures

```javascript
import { Webhook } from "svix";

const webhook = new Webhook(process.env.SVIX_WEBHOOK_SECRET);

app.post("/webhook", express.raw({ type: "application/json" }), async (req, res) => {
  try {
    const event = webhook.verify(req.body, req.headers);
    await processWebhook(event);
    res.status(200).send("OK");
  } catch (err) {
    res.status(400).send("Invalid signature");
  }
});
```

### Webhook Payload Structure

All Notabene webhooks use the same envelope format:

```json
{
  "message": "flow.payout.created",
  "payload": {
    "id": "423ae44a-...",
    "for": "did:web:notabene.id:pg",
    "amount": "1300",
    "currency": "USD",
    ...
  },
  "version": "1.0.0"
}
```

| Field | Description |
|---|---|
| `message` | The event type (e.g., `notification.transferStatusChanged`, `tap.requirePresentationRequested`, `flow.payin.created`) |
| `payload` | All event-specific data, including the transfer `id` and other fields |
| `version` | Always `"1.0.0"` |

> **Important:** The event type is in the `message` field (not `eventType`). The transfer ID is in `payload.id` (not a top-level `id`). All event-specific data is nested inside `payload`.

---

## Where to Integrate

### When You Are Sending Crypto (Outgoing)

Use the [Outgoing Transfers](#outgoing-transfers-originator) API whenever your platform is sending crypto tokens. Common integration points:

- **Withdrawal screen** on a crypto exchange
- **The crypto arm of a fiat onramp** — after converting fiat to crypto, the outgoing transfer to the customer's wallet
- **Send transaction** functionality in a custodial wallet or server API
- **Post-trade settlement** — settling tokens after an OTC or exchange trade

**Critical requirement:** You must call the create transfer API **before** broadcasting the on-chain transaction. Wait for the `notification.transferStatusChanged` webhook with `toStatus: "AUTHORIZED"` before sending funds. This status means your internal compliance policies (configured in the Notabene dashboard) have been satisfied. Do not send funds until authorization is confirmed.

Your compliance policies may require collecting additional information from your customer (e.g., beneficiary details) before the transfer can be authorized. Use the **Withdrawal Assist** component from the [Notabene JavaScript SDK](https://gitlab.com/notabene/open-source/javascript-sdk) to embed a data collection UI in your withdrawal flow that gathers the required information before submitting the transfer.

### When You Are Receiving Crypto (Incoming)

Use the [Incoming Transfers](#incoming-transfers-beneficiary) API whenever your platform is receiving crypto tokens. Common integration points:

- **Deposit flow** on a crypto exchange
- **The crypto arm of a fiat offramp** — receiving crypto from a customer before converting to fiat
- **Receiving funds** in a custodial wallet or server API

In most cases, you will receive the travel rule message **before** the funds arrive on-chain. Your compliance team handles authorization through their configured policies in the Notabene dashboard. As a developer, your primary responsibility is integrating the incoming transfer flow: confirming address ownership, verifying beneficiary name, and processing the authorization decision.

If your compliance team needs additional data from the customer (e.g., to clarify a deposit source), you can use the **Deposit Assist** component from the [Notabene JavaScript SDK](https://gitlab.com/notabene/open-source/javascript-sdk) to collect it.

You can also use the **Deposit Request** component to improve the deposit experience on your deposit screen. This component adds additional details to your deposit QR code, enabling the sender's VASP to pre-fill transfer information and resulting in smoother, faster deposits.

### Delegating Settlement to a Wallet Service (IP)

If you don't want to handle blockchain settlement yourself, you can add a supported **wallet service (Infrastructure Provider / IP)** as an agent to your transfers. The IP handles all address management, on-chain settlement, and transaction reporting on your behalf. You focus solely on compliance — PII exchange, authorization decisions, and beneficiary verification.

**How it works:**

When creating a transfer, add the wallet service as an agent acting on your behalf:

```json
{
  "ref": "withdrawal-001",
  "originator": { "@id": "did:email:customer@example.com" },
  "beneficiary": { "@id": "did:email:recipient@example.com" },
  "asset": "eip155:1/erc20:0xA0b86991c6218b36c1d19D4a2e9Eb0cE3606eB48",
  "amount": "100.00",
  "agents": [
    {
      "@id": "did:web:your-vasp.com",
      "role": "VASP",
      "for": "did:email:customer@example.com",
      "policies": [{ "@type": "REQUIRE_AUTHORIZATION" }]
    },
    {
      "@id": "did:web:wallet-service.com",
      "role": "IP",
      "for": "did:web:your-vasp.com"
    }
  ]
}
```

The IP receives a `notification.transferAgentAdded` webhook, provisions a wallet, and waits for the transfer to reach `AUTHORIZED` status before executing settlement on-chain. Your responsibilities remain:

- Creating the transfer
- Responding to PII requests (`tap.requirePresentationRequested`)
- Authorizing or rejecting based on compliance checks

For incoming transfers, the IP handles address ownership confirmation and settlement reconciliation. See the [Wallet Service Integration Guide](./wallet-service-guide.md) for the full IP perspective.

---

## Outgoing Transfers (Originator)

As an originator VASP, you create transfers and respond to counterparty requirements.

### 1. Create a Transfer

```
POST /entities/{entityDID}/tx
Authorization: Bearer <token>
Content-Type: application/json
```

**To a hosted wallet (known counterparty):**

```json
{
  "originator": {
    "@id": "did:email:sender@example.com"
  },
  "beneficiary": {
    "@id": "did:email:receiver@example.com"
  },
  "asset": "eip155:1/erc20:0xA0b86991c6218b36c1d19D4a2e9Eb0cE3606eB48",
  "amount": "1000",
  "agents": [
    {
      "@id": "did:pkh:eip155:1:0xSourceAddress",
      "for": "did:web:your-vasp.com",
      "role": "SourceAddress"
    },
    {
      "@id": "did:web:your-vasp.com",
      "for": "did:email:sender@example.com",
      "role": "VASP"
    },
    {
      "@id": "did:web:counterparty-vasp.com",
      "for": "did:email:receiver@example.com",
      "role": "VASP"
    },
    {
      "@id": "did:pkh:eip155:1:0xSettlementAddress",
      "for": "did:web:counterparty-vasp.com",
      "role": "SettlementAddress"
    }
  ],
  "ref": "unique-idempotency-key"
}
```

**To an unhosted wallet:** Omit the counterparty VASP agent and point the SettlementAddress `for` field at the beneficiary directly:

```json
{
  "originator": { "@id": "did:email:sender@example.com" },
  "beneficiary": { "@id": "did:email:receiver@example.com" },
  "asset": "eip155:1/erc20:0xA0b86991c6218b36c1d19D4a2e9Eb0cE3606eB48",
  "amount": "1000",
  "agents": [
    {
      "@id": "did:pkh:eip155:1:0xSourceAddress",
      "for": "did:web:your-vasp.com",
      "role": "SourceAddress"
    },
    {
      "@id": "did:web:your-vasp.com",
      "for": "did:email:sender@example.com",
      "role": "VASP"
    },
    {
      "@id": "did:pkh:eip155:1:0xSettlementAddress",
      "for": "did:email:receiver@example.com",
      "role": "SettlementAddress"
    }
  ],
  "ref": "unique-idempotency-key"
}
```

After creation, the system automatically:

- Runs agent discovery (if counterparty not specified)
- Checks jurisdictions and travel rule thresholds
- Applies your default Transaction Authorization Policies
- Sends the transfer request via DIDComm to the counterparty

**Key payload fields:**

| Field         | Description                                           |
| ------------- | ----------------------------------------------------- |
| `originator`  | Your customer's DID                                   |
| `beneficiary` | The recipient's DID                                   |
| `asset`       | CAIP-19 format or Notabene abbreviation (e.g., `BTC`) |
| `amount`      | Transfer amount as a string                           |
| `agents`      | Chain of agents with roles and `for` relationships    |
| `ref`         | Idempotency key for deduplication                     |

> **Note on agent roles:** The Notabene API uses simplified role values (`VASP`, `SourceAddress`, `SettlementAddress`) that differ from the underlying TAP protocol roles (`OriginatorVASP`, `BeneficiaryVASP`, etc. — see TAIP-5). The API maps these internally. Always use the Notabene role values shown in this guide when calling Notabene endpoints.

### 2. Provide PII When Requested

Listen for the **`tap.requirePresentationRequested`** webhook. This fires when the counterparty's jurisdiction requires Travel Rule data.

**Webhook payload:**

```json
{
  "message": "tap.requirePresentationRequested",
  "payload": {
    "for": "did:web:your-vasp.com",
    "id": "<transfer-id>",
    "originatorAgentID": "did:web:your-vasp.com",
    "beneficiaryAgentID": "did:web:counterparty-vasp.com",
    "originatorPresentationDefinition": "https://pd.notabene.id/ivms101/v2/US-3000.json",
    "beneficiaryPresentationDefinition": "https://pd.notabene.id/ivms101/v2/JP-0.json",
    "callbackUrl": "/entity/<entityDID>/tx/<tx>/append"
  }
}
```

**Append PII (Notabene-managed encryption):**

```
POST /entities/{entityDID}/transfers/{transferId}/append
Authorization: Bearer <token>
Content-Type: application/json
```

```json
{
  "originator": {
    "@id": "did:email:sender@example.com"
  },
  "beneficiary": {
    "@id": "did:email:receiver@example.com"
  },
  "ivms101": {
    "originator": {
      "originatorPersons": [
        {
          "naturalPerson": {
            "name": {
              "nameIdentifier": [
                {
                  "primaryIdentifier": "Smith",
                  "secondaryIdentifier": "John",
                  "nameIdentifierType": "LEGL"
                }
              ]
            },
            "dateAndPlaceOfBirth": {
              "dateOfBirth": "1990-01-15"
            },
            "geographicAddress": [
              {
                "country": "US",
                "addressType": "HOME"
              }
            ]
          }
        }
      ],
      "accountNumber": ["did:pkh:eip155:1:0xSourceAddress"]
    }
  }
}
```

**Response:** `HTTP 202` on success. PII requirements vary by jurisdiction. Use the `presentationDefinition` URLs from the webhook to check which IVMS101 fields are required.

### 3. Settle the Transfer

After executing the on-chain transaction, report the settlement:

```
POST /entities/{entityDID}/tx/{transferId}/settle
Authorization: Bearer <token>
Content-Type: application/json
```

```json
{
  "settlementId": "eip155:1:tx/0x3edb98c24d46d148eb926c714f4fbaa117c47b0c0821f38bfce9763604457c33"
}
```

---

## Incoming Transfers (Beneficiary)

As a beneficiary VASP, you receive transfers and must confirm ownership, verify identity, and make authorization decisions.

### 1. Confirm Address Ownership

When a transfer arrives targeting one of your addresses, you must confirm ownership.

**Option A: Pre-register addresses via the Relationships API (recommended)**

```
POST /entities/{entityDID}/relationship
Authorization: Bearer <token>
Content-Type: application/json
```

```json
{
  "from": "did:pkh:eip155:1:0xYourDepositAddress",
  "to": "did:web:your-vasp.com"
}
```

Pre-registered addresses are automatically confirmed when transfers arrive.

**Option B: Respond to the webhook**

Listen for **`tap.requireRelationshipConfirmationRequested`** and POST to the `confirmCallbackUrl` in the payload to confirm ownership.

### 2. Verify Beneficiary Name

After PII arrives, listen for the **`tap.requirePresentationSatisfied`** webhook. Then:

1. Retrieve the transfer with PII:

```
GET /entities/{entityDID}/tx/{transferId}?decrypt=true
Authorization: Bearer <token>
```

2. Extract the beneficiary name from the response at:
   `beneficiary.beneficiaryPerson.naturalPerson.name.nameIdentifier`

3. Compare against your KYC records (fuzzy matching recommended, ~90% threshold).

4. Confirm the checks:

```
POST /entities/{entityDID}/tx/{transferId}/checks/beneficiary-name-matching
Authorization: Bearer <token>
```

```
POST /entities/{entityDID}/tx/{transferId}/checks/internal-checks
Authorization: Bearer <token>
```

Both checks must pass before the transfer can proceed to authorization.

If the name does not match, reject the transfer with reason `BENEFICIARY_NOT_FOUND`.

### 3. Reject a Transfer

If your compliance checks fail, reject the transfer:

```
POST /entities/{entityDID}/tx/{transferId}/reject
Authorization: Bearer <token>
Content-Type: application/json
```

```json
{
  "reason": "SANCTION_SCREENING",
  "comment": "Failed sanctions check"
}
```

**Valid rejection reasons:** `COUNTERPARTY_RISK`, `COUNTERPARTY_DUE_DILIGENCE`, `BLOCKCHAIN_RISK_SCORE`, `SANCTION_SCREENING`, `ASSET_TYPE`, `SUSPICIOUS_TRANSACTION`, `COUNTERPARTY_POLICIES`, `COUNTERPARTY_REJECTED`, `COUNTERPARTY_NO_RESPONSE`, `CANCELED_BY_INITIATOR`, `REMOVED_FROM_TRANSFER`, `TRANSFER_PARTICIPANT`, `SOURCE_ADDRESS`, `BENEFICIARY_ADDRESS`, `BENEFICIARY_NOT_FOUND`, `ORIGINATOR_REJECT_OUTGOING`, `BENEFICIARY_REJECT_INCOMING`, `COMPLIANCE_POLICIES`, `OTHER`

### 4. Reconcile Received Transfers

In most cases, the travel rule message arrives **before** the on-chain transaction. However, sometimes funds arrive first — or you need to match an on-chain deposit to its corresponding Notabene transfer after the fact. Use the **match endpoint** to reconcile on-chain transactions with Notabene transfers.

```
GET /entities/{entityDID}/tx/match?settlement_id={txHash}&settlement_address={destinationAddress}&direction=INCOMING
Authorization: Bearer <token>
```

| Parameter | Required | Description |
|-----------|----------|-------------|
| `settlement_id` | yes | On-chain transaction hash (e.g., `0xabc123...`) |
| `settlement_address` | yes | Destination address that received funds |
| `settlement_id_index` | no | Vout or log index (for UTXO chains or multi-transfer txs) |
| `memo_tag` | no | Memo/tag for chains that use them (e.g., XRP, XLM) |
| `direction` | no | `INCOMING` or `OUTGOING` |

The response includes a `meta.match_strategy` field:
- `strict` — exact match on all provided parameters
- `broadened` — matched after relaxing the `settlement_id_index` constraint
- `none` — no matching transfer found

**When no match is found:** If a deposit arrives with no corresponding Notabene transfer, create the transfer in Notabene yourself so it can be tracked for compliance:

```
POST /entities/{entityDID}/tx
Authorization: Bearer <token>
Content-Type: application/json
```

```json
{
  "ref": "your-unique-deposit-reference",
  "originator": { "@id": "did:pkh:eip155:1:0xSenderAddress" },
  "beneficiary": { "@id": "did:email:your-customer@example.com" },
  "asset": "eip155:1/erc20:0xA0b86991c6218b36c1d19D4a2e9Eb0cE3606eB48",
  "amount": "100.00",
  "agents": [
    {
      "@id": "did:web:your-vasp.com",
      "role": "VASP",
      "for": "did:email:your-customer@example.com",
      "policies": [{ "@type": "REQUIRE_AUTHORIZATION" }]
    }
  ],
  "settlementId": "0xTransactionHash"
}
```

This ensures the deposit enters Notabene's compliance workflow even when the originator VASP did not initiate the travel rule message. Notabene will attempt to identify the originator VASP and request travel rule data from them.

**Recommended pattern:** When you detect an on-chain deposit, call the match endpoint. If a transfer is found, link it to your internal deposit record. If no match is found, create the transfer in Notabene so it can be reconciled and compliance-tracked.

---

## Transfer Statuses

| Status               | Description                                      |
| -------------------- | ------------------------------------------------ |
| `OUTGOING`           | New outgoing transfer created                    |
| `INCOMING`           | New incoming transfer received                   |
| `AUTHORIZED`         | Transfer authorized by all required parties      |
| `REJECTED`           | Transfer rejected                                |
| `SETTLED`            | Transaction settled on-chain                     |
| `REVERTED`           | Transaction reverted (terminal)                  |

---

## Webhook Event Reference

### Required for Outgoing Transfers

| Event                                  | When                                      | Action Required |
| -------------------------------------- | ----------------------------------------- | --------------- |
| `tap.requirePresentationRequested`     | Counterparty requests Travel Rule PII     | POST PII to `callbackUrl` |
| `notification.transferStatusChanged`   | Transfer status changes                   | Wait for `AUTHORIZED` before sending on-chain |

### Required for Incoming Transfers

| Event                                           | When                                    | Action Required |
| ----------------------------------------------- | --------------------------------------- | --------------- |
| `tap.requireRelationshipConfirmationRequested`  | Address ownership confirmation needed   | POST to `confirmCallbackUrl` |
| `tap.requirePresentationSatisfied`              | PII has been received                   | Verify beneficiary name, complete checks |
| `notification.transferStatusChanged`            | Transfer status changes                 | Update internal records |

### Informational Events

| Event                                              | Description                              |
| -------------------------------------------------- | ---------------------------------------- |
| `notification.transferCreated`                     | Transfer created                         |
| `notification.transferAgentAdded`                  | Agent added to transfer                  |
| `notification.transferAgentReplaced`               | Agent replaced in transfer               |
| `notification.transferAgentStatusChanged`          | Agent status updated                     |
| `tap.requireAuthorizationSatisfied`                | All authorizations complete              |
| `tap.requireRelationshipConfirmationSatisfied`     | Address ownership confirmed              |
| `tap.policySatisfied`                              | Policy fully satisfied                   |
| `tap.requirePresentationPartiallySatisfied`        | Partial PII received, more needed        |
