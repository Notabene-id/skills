# Notabene Flow Pay-ins API Guide

This guide covers the API usage for Notabene Flow pay-in transactions, from both the initiator (PIA) and responder (PRA) perspectives. Flow is a compliant stablecoin payment network that coordinates identity, compliance, authorization, and settlement using the Transaction Authorization Protocol (TAP).

> **Travel Rule compliance is automatic.** Flow transactions automatically feed into Notabene's Travel Rule compliance workflows. If you are primarily focused on payment flows, you do not need to separately integrate the Travel Rule API — compliance is handled as part of the Flow protocol.

For general authentication and webhook setup, see [guide-travel-rule-api.md](./guide-travel-rule-api.md).

## Table of Contents

- [Concepts](#concepts)
  - [What is a Pay-in?](#what-is-a-pay-in)
  - [Roles](#roles)
- [Prerequisites](#prerequisites)
  - [Authentication](#authentication)
  - [Model Your Customers](#model-your-customers)
  - [Webhook Setup](#webhook-setup)
- [Initiator Flow (PIA)](#initiator-flow-pia)
  - [Step 1: Create a Pay-in](#step-1-create-a-pay-in)
  - [Step 2: Distribute Payment Link](#step-2-distribute-payment-link)
  - [Step 3: Monitor Progress via Webhooks](#step-3-monitor-progress-via-webhooks)
  - [Step 4: Authorize or Reject](#step-4-authorize-or-reject)
  - [Step 5: Reconcile Settlement](#step-5-reconcile-settlement)
- [Responder Flow (PRA)](#responder-flow-pra)
  - [Step 1: Receive the Pay-in Request](#step-1-receive-the-pay-in-request)
  - [Step 2: Request Customer Authorization](#step-2-request-customer-authorization)
  - [Step 3: Select Asset, Authorize, or Reject](#step-3-select-asset-authorize-or-reject)
  - [Step 4: Execute Settlement](#step-4-execute-settlement)
- [Infrastructure Provider Flow (IP)](#infrastructure-provider-flow-ip)
  - [How IPs Are Added](#how-ips-are-added)
  - [Handling Settlement as an IP](#handling-settlement-as-an-ip)
- [Complete Lifecycle Example](#complete-lifecycle-example)
- [Webhook Event Reference](#webhook-event-reference)

---

## Concepts

### What is a Pay-in?

A pay-in is a **pull payment**: a merchant (or their PIA) requests payment, and the customer pays using their wallet or account through their PRA. Flow coordinates discovery, identity exchange, compliance checks, authorization, and settlement across all parties.

### Roles

| Role | Full Name               | Represents          | Examples                                      |
| ---- | ----------------------- | -------------------- | --------------------------------------------- |
| PIA  | Payment Initiating Agent | Merchant / payee    | Payment processors, billing platforms, SaaS    |
| PRA  | Payment Responding Agent | Payer / customer    | Payout PSPs, wallets, neobanks                 |
| IP   | Infrastructure Provider  | Supporting services  | Custody platforms, MPC wallets, liquidity providers |

**PIA responsibilities:**
- Create pay-in requests on behalf of merchants
- Specify accepted assets and settlement addresses
- Distribute payment links
- Reconcile incoming funds

**PRA responsibilities:**
- Receive pay-in requests and surface them to payers
- Provide an authorization URL for customer approval (or auto-authorize)
- Authorize or reject transfers based on compliance checks
- Execute or coordinate settlement via IPs once authorized

**IPs** are optional supporting agents that provide custody, wallets, FX, or fiat rail capabilities.

---

## Prerequisites

### Authentication

Use the same OAuth flow as Transact V2:

```
POST https://auth.notabene.id/oauth/token
```

```json
{
  "client_id": "<your_client_id>",
  "client_secret": "<your_client_secret>",
  "grant_type": "client_credentials",
  "audience": "https://api.eu1.notabene.id"
}
```

**Base URL:** `https://api.eu1.notabene.id`

### Model Your Customers

Before creating pay-ins, register your customers (merchants for PIAs, payers for PRAs):

```
POST /entities/{entityDID}/flow/customers
Authorization: Bearer <token>
Content-Type: application/json
```

```json
{
  "customerDid": "did:web:merchant.example.com",
  "customerType": "natural_person",
  "profileData": {
    "naturalPerson": {
      "name": {
        "nameIdentifier": [
          {
            "primaryIdentifier": "Doe",
            "secondaryIdentifier": "Jane",
            "nameIdentifierType": "LEGL"
          }
        ]
      },
      "geographicAddress": [
        {
          "country": "US",
          "addressType": "BIZZ"
        }
      ]
    }
  },
  "verificationStatus": "verified",
  "verificationLevel": "enhanced"
}
```

**Update a customer:**

```
PUT /entities/{entityDID}/flow/customers/{customerDID}
```

**Delete a customer:**

```
DELETE /entities/{entityDID}/flow/customers/{customerDID}
```

**Verification status values:** `pending`, `verified`, `rejected`, `expired`
**Verification level values:** `basic`, `enhanced`, `premium`

### Webhook Setup

Subscribe to Flow webhook events in the Notabene dashboard. See [Webhook Event Reference](#webhook-event-reference) for the full list.

### Delegating Settlement to a Wallet Service (IP)

Both PIAs and PRAs can add a supported **wallet service (Infrastructure Provider / IP)** as an agent to handle all blockchain settlement on their behalf. This lets you focus on the payment and compliance aspects without managing wallets, on-chain transfers, or settlement addresses.

The IP handles: wallet provisioning, asset selection, settlement address management, on-chain execution, and settlement reporting. You handle: creating pay-ins, distributing payment links, authorization decisions, and reconciliation.

See [Infrastructure Provider Flow](#infrastructure-provider-flow-ip) for how IPs are added, and the [Wallet Service Integration Guide](./wallet-service-guide.md) for the full IP perspective.

---

## Initiator Flow (PIA)

### Step 1: Create a Pay-in

```
POST /entities/{entityDID}/flow/customers/{merchantCustomerDID}/payins
Authorization: Bearer <token>
Content-Type: application/json
```

```json
{
  "ref": "INV-2025-001",
  "amount": "1000.00",
  "currency": "USD",
  "asset": "eip155:1/erc20:0xA0b86991c6218b36c1d19D4a2e9Eb0cE3606eB48",
  "supportedAssets": [
    "eip155:1/erc20:0xA0b86991c6218b36c1d19D4a2e9Eb0cE3606eB48",
    "eip155:137/erc20:0x2791Bca1f2de4661ED88A30C99A7a9449Aa84174",
    "payto://iban/DE89370400440532013000"
  ],
  "customer": {
    "@id": "did:email:customer@example.com",
    "name": "Test Customer",
    "email": "customer@example.com"
  },
  "memo": "Payment for order #12345"
}
```

**Request fields:**

| Field             | Type     | Required | Description                                       |
| ----------------- | -------- | -------- | ------------------------------------------------- |
| `ref`             | string   | Yes      | Unique reference for reconciliation               |
| `amount`          | string   | No       | Payment amount                                    |
| `currency`        | string   | No       | Fiat currency code (e.g., `USD`)                  |
| `asset`           | string   | No       | Default asset in CAIP-19 format                   |
| `supportedAssets` | string[] | No       | Up to 10 accepted assets — CAIP-19 tokens or PayTo URIs (see below) |
| `customer`        | object   | Yes      | Payer identity (`@id`, optional `name`, `email`)  |
| `memo`            | string   | No       | Payment description                               |

**Supported assets:** The `supportedAssets` array accepts up to 10 entries. Each entry can be either:

- **CAIP-19 token identifier** — e.g., `eip155:1/erc20:0xA0b86991c6218b36c1d19D4a2e9Eb0cE3606eB48` (USDC on Ethereum)
- **PayTo URI** — e.g., `payto://iban/DE89370400440532013000` (for fiat settlement)

The PRA selects one of these assets on behalf of the payer (see [Responder Flow](#responder-flow-pra)), after which the PIA returns a settlement address for that asset.

**Response:**

```json
{
  "id": "<transfer-id>",
  "status": "INCOMING",
  "flowState": "CUSTOMER_REQUESTED",
  "transactionType": "PAYIN",
  "paymentLink": "https://connect.notabene.id/payin/<token>"
}
```

### Step 2: Distribute Payment Link

Send the `paymentLink` from the response to the payer via your preferred channel (email, in-app, invoice embed). When the payer opens it, you receive the **`flow.payin.opened`** webhook.

### Step 3: Monitor Progress via Webhooks

Track the pay-in lifecycle through webhook events:

1. **`flow.payin.created`** - Pay-in created (confirmation)
2. **`flow.payin.opened`** - Customer opened the payment link
3. **`flow.payin.assetSelected`** - Customer selected a payment asset
4. **`flow.payin.agentAdded`** - PRA or IP agent joined the transfer
5. **`flow.payin.settlementAddressSelected`** - Settlement address determined
6. **`flow.payin.fundingAdded`** - Funding address details finalized
7. **`notification.transferStatusChanged`** - Transfer status updated (authorized, settled, etc.)

### Step 4: Authorize or Reject

When the transfer is ready for your authorization decision (e.g., after Travel Rule data has been exchanged), authorize or reject it:

**Authorize:**

```
POST /entities/{entityDID}/tx/{transferId}/authorize
Authorization: Bearer <token>
```

**Reject:**

```
POST /entities/{entityDID}/tx/{transferId}/reject
Authorization: Bearer <token>
Content-Type: application/json
```

```json
{
  "reason": "COMPLIANCE_POLICIES",
  "comment": "Failed risk assessment"
}
```

### Step 5: Reconcile Settlement

When the PRA settles, you receive **`notification.transferStatusChanged`** with `toStatus: "SETTLED"`. Use the transfer ID and your `ref` to reconcile against invoices and mark them as paid.

Retrieve the final transfer state:

```
GET /entities/{entityDID}/tx/{transferId}
Authorization: Bearer <token>
```

---

## Responder Flow (PRA)

As a PRA, you receive pay-in requests and manage the payer's side of the transaction.

### Step 1: Receive the Pay-in Request

When a PIA creates a pay-in that involves your customer, you receive the **`flow.payin.created`** webhook (or `flow.payout.created` depending on direction):

```json
{
  "message": "flow.payin.created",
  "payload": {
    "id": "<transfer-id>",
    "for": "did:web:your-pra.com",
    "transactionType": "PAYIN",
    "customerDid": "did:email:customer@example.com",
    "merchantDid": "did:web:merchant.com",
    "paymentLink": "https://connect.notabene.id/payin/<token>",
    "amount": "1000.00",
    "currency": "USD",
    "supportedAssets": [
      "eip155:1/erc20:0xA0b86991c6218b36c1d19D4a2e9Eb0cE3606eB48"
    ]
  },
  "version": "1.0.0"
}
```

> **Webhook format:** All Notabene webhooks use the same envelope: `message` contains the event type, `payload` contains all event-specific data (including the transfer `id`), and `version` is always `"1.0.0"`. See [Webhook Payload Structure](./guide-travel-rule-api.md#webhook-payload-structure) for details.

You can also poll for pay-ins:

```
GET /entities/{entityDID}/flow/payins
Authorization: Bearer <token>
```

Retrieve details for a specific transfer:

```
GET /entities/{entityDID}/tx/{transferId}
Authorization: Bearer <token>
```

**As a PRA, your workflow is:**

1. Receive the pay-in request via webhook or API
2. Provide an authorization URL so the customer can approve the payment (or skip this to auto-authorize)
3. Run compliance checks and authorize or reject the transfer
4. Execute settlement and report it

### Step 2: Request Customer Authorization

Before authorizing, you typically need the customer (payer) to approve the payment. Call the authorization required endpoint to provide a URL where the customer can review and approve:

```
POST /entities/{entityDID}/flow/payins/{payinId}/authorization_required
Authorization: Bearer <token>
Content-Type: application/json
```

```json
{
  "authorizationUrl": "https://your-pra.com/approve/abc123"
}
```

**Response:**

```json
{
  "message": "Authorization link added",
  "transferId": "<payin-id>"
}
```

This sends a **`flow.payin.authorizationRequired`** webhook to the PIA, which includes your `authorizationUrl`. The PIA can then direct the customer to your authorization page.

**URL requirements:**
- Must be a valid HTTPS URL
- Cannot be localhost or loopback addresses

If your system auto-authorizes payments (e.g., for pre-approved connections or low-value transfers), you can skip this step and proceed directly to authorize.

### Step 3: Select Asset, Authorize, or Reject

The PRA selects one of the `supportedAssets` from the pay-in request on behalf of the payer. This is done as part of authorization by including a `settlement_address` for the chosen asset.

**Authorize (with asset selection):**

```
POST /entities/{entityDID}/tx/{transferId}/authorize
Authorization: Bearer <token>
Content-Type: application/json
```

```json
{
  "settlement_address": "eip155:1:0xPayerWalletAddress"
}
```

The `settlement_address` indicates which asset the payer will use and from which address funds will be sent. The PIA receives this via the `flow.payin.settlementAddressSelected` webhook and responds with their own settlement address where funds should be sent.

**Reject:**

```
POST /entities/{entityDID}/tx/{transferId}/reject
Authorization: Bearer <token>
Content-Type: application/json
```

```json
{
  "reason": "SANCTION_SCREENING",
  "comment": "Customer failed compliance check"
}
```

### Step 4: Execute Settlement

Once both parties have authorized and the PIA has provided a settlement address (via the `flow.payin.fundingAdded` webhook), send the funds on-chain and report the settlement:

```
POST /entities/{entityDID}/tx/{transferId}/settle
Authorization: Bearer <token>
Content-Type: application/json
```

```json
{
  "settlementId": "eip155:1:tx/0xabc123..."
}
```

The `settlementId` is the blockchain transaction hash or bank reference confirming the value transfer. If you are using an Infrastructure Provider for settlement, the IP handles this step instead (see [Infrastructure Provider Flow](#infrastructure-provider-flow-ip)).

---

## Infrastructure Provider Flow (IP)

Infrastructure Providers such as wallet custody platforms, MPC wallet services, or liquidity providers handle the blockchain settlement aspect of a Flow pay-in on behalf of a PIA or PRA. The IP does not own the customer relationship — it acts as a settlement agent under instruction from its client.

### How IPs Are Added

**By the PIA at creation time:** The PIA typically includes the IP as an agent when creating the pay-in. This is common when the merchant's custody provider will receive the settled funds:

```json
{
  "ref": "INV-2025-001",
  "amount": "1000.00",
  "currency": "USD",
  "supportedAssets": ["eip155:1/erc20:0xA0b86991c6218b36c1d19D4a2e9Eb0cE3606eB48"],
  "customer": {
    "@id": "did:email:customer@example.com"
  },
  "agents": [
    {
      "@id": "did:web:custody-provider.com",
      "for": "did:web:pia-entity.com",
      "role": "IP"
    }
  ]
}
```

**By the PRA after joining:** A PRA can add their own IP as an agent once they have been added to the transfer. This is common when the payer's wallet provider will execute the on-chain transfer:

```
POST /entities/{entityDID}/tx/{transferId}/agents
Authorization: Bearer <token>
Content-Type: application/json
```

```json
{
  "agent": {
    "@id": "did:web:wallet-provider.com",
    "for": "did:web:pra-entity.com",
    "role": "IP"
  }
}
```

When an IP is added, all parties receive the **`flow.payin.agentAdded`** webhook.

### Handling Settlement as an IP

As an IP, you receive instructions through the authorization flow from your client (the PIA or PRA). Your workflow:

1. **Receive the transfer** — Listen for the **`flow.payin.agentAdded`** webhook to know you have been added to a transfer
2. **Receive authorization** — When your client authorizes the transfer, you receive a **`notification.transferStatusChanged`** webhook with `toStatus: "AUTHORIZED"` and the settlement details
3. **Execute the on-chain transfer** — Move funds to the settlement address specified in the transfer
4. **Report settlement** — Call settle with the transaction hash:

```
POST /entities/{entityDID}/tx/{transferId}/settle
Authorization: Bearer <token>
Content-Type: application/json
```

```json
{
  "settlementId": "eip155:1:tx/0xabc123..."
}
```

---

## Complete Lifecycle Example

```
PIA (Merchant side)                       PRA (Payer side)
─────────────────                         ────────────────
1. POST /flow/customers/.../payins
   (with supportedAssets, optional IP agent)
   → paymentLink created
                                          ← flow.payin.created webhook
2. Send paymentLink to payer              2. Surface in payer approval UI

   ← flow.payin.opened webhook            3. Customer opens payment link

   ← flow.payin.agentAdded webhook        4. PRA agent added to transfer
                                             (optionally adds own IP agent)

                                          5. POST /flow/payins/{id}/authorization_required
   ← flow.payin.authorizationRequired        (with authorizationUrl)
   → Direct customer to PRA auth page
                                             Customer approves on PRA auth page

                                          6. POST /tx/{id}/authorize
                                             (selects asset + settlement_address)
   ← flow.payin.settlementAddressSelected

   PIA returns settlement address
   ← flow.payin.fundingAdded              ← flow.payin.fundingAdded

   7. POST /tx/{id}/authorize
   ← transferStatusChanged(AUTHORIZED)    ← transferStatusChanged(AUTHORIZED)

                                          8. Send funds on-chain (PRA or IP)
                                          9. POST /tx/{id}/settle
   ← transferStatusChanged(SETTLED)       ← transferStatusChanged(SETTLED)

10. Reconcile and mark invoice paid
```

---

## Webhook Event Reference

### For PIAs (Initiators)

| Event                                  | When                                          | What to Do                                      |
| -------------------------------------- | --------------------------------------------- | ----------------------------------------------- |
| `flow.payin.created`                   | Pay-in request created                        | Confirm creation in your system                 |
| `flow.payin.opened`                    | Customer opened payment link                  | Track customer engagement                       |
| `flow.payin.assetSelected`             | Customer selected payment asset               | Track payment progress                          |
| `flow.payin.authorizationRequired`     | PRA provided an authorization URL             | Direct customer to the PRA's authorization page |
| `flow.payin.fundingAdded`             | Funding address details finalized             | Record funding details                          |
| `notification.transferStatusChanged`   | Transfer status changed (e.g., `SETTLED`)     | Authorize, then reconcile when settled          |

### For PRAs (Responders)

| Event                                  | When                                          | What to Do                                      |
| -------------------------------------- | --------------------------------------------- | ----------------------------------------------- |
| `flow.payin.created`                   | Pay-in request received from PIA              | Surface to payer for approval                   |
| `flow.payin.agentAdded`               | Your agent was added to the transfer          | Confirm agent assignment                        |
| `flow.payin.fundingAdded`             | PIA provided settlement address for funds     | Send funds to this address                      |
| `notification.transferStatusChanged`   | Transfer status changed (e.g., `AUTHORIZED`)  | Call `/settle` when `toStatus: "AUTHORIZED"`    |

### For IPs (Infrastructure Providers)

| Event                                  | When                                          | What to Do                                      |
| -------------------------------------- | --------------------------------------------- | ----------------------------------------------- |
| `flow.payin.agentAdded`               | You were added to the transfer                | Acknowledge and prepare for settlement          |
| `flow.payin.fundingAdded`             | Settlement address finalized                  | Prepare to send or receive funds                |
| `notification.transferStatusChanged`   | Transfer status changed (e.g., `AUTHORIZED`)  | Execute on-chain transfer, then call `/settle`  |
