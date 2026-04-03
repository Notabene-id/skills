# Notabene Flow Invoice Payments API Guide

This guide covers the API usage for Notabene Flow invoice payments, from both the initiator (PIA) and responder (PRA) perspectives. Flow is a B2B pull-payment network that coordinates identity, compliance, authorization, and settlement using the Transaction Authorization Protocol (TAP).

What are Pull Payments and why are they needed for Invoice Payments? Payments come in two flavors. The most common ones for B2B payments and in stablecoins are known as push payments, where the sender pushes the funds to the beneficiary. This is what SWIFT and blockchains do natively. Pull payments are mostly known from the Card networks, where the transaction is initiated by the beneficiary (the merchant) to pull (or request) funds from their customers bank account. Pull payments are particularly useful for invoice payments as they can automatically reconcile with the billing or accounts receivable system and trigger amongst other things fulfillment.

> **Travel Rule compliance is automatic.** Flow transactions automatically feed into Notabene's Travel Rule compliance workflows. If you are primarily focused on payment flows, you do not need to separately integrate the Travel Rule API — compliance is handled as part of the Flow protocol.

For general authentication and webhook setup, see [guide-travel-rule-api.md](./guide-travel-rule-api.md).

## Table of Contents

- [Concepts](#concepts)
  - [What is an Invoice Payment?](#what-is-an-invoice-payment)
  - [Roles](#roles)
- [Prerequisites](#prerequisites)
  - [Authentication](#authentication)
  - [Model Your Customers](#model-your-customers)
  - [Supported Assets & Fallback Settlement Addresses](#supported-assets--fallback-settlement-addresses)
  - [Webhook Setup](#webhook-setup)
- [Initiator Flow (PIA)](#initiator-flow-pia)
  - [Step 1: Create an Invoice Payment](#step-1-create-an-invoice-payment)
  - [Step 2: Distribute Payment Link](#step-2-distribute-payment-link)
  - [Step 3: Monitor Progress via Webhooks](#step-3-monitor-progress-via-webhooks)
  - [Step 4: Authorize or Reject](#step-4-authorize-or-reject)
  - [Step 5: Reconcile Settlement](#step-5-reconcile-settlement)
- [Responder Flow (PRA)](#responder-flow-pra)
  - [Step 1: Receive the Invoice Payment Request](#step-1-receive-the-invoice-payment-request)
  - [Step 2: Request Customer Authorization](#step-2-request-customer-authorization)
  - [Step 3: Select Asset](#step-3-select-asset)
  - [Step 4: Authorize or Reject](#step-4-authorize-or-reject)
  - [Step 5: Execute Settlement](#step-5-execute-settlement)
  - [Step 6: Report Settlement](#step-6-report-settlement)
- [Infrastructure Provider Flow (IP)](#infrastructure-provider-flow-ip)
  - [How IPs Are Added](#how-ips-are-added)
  - [Handling Settlement as an IP](#handling-settlement-as-an-ip)
- [Complete Lifecycle Example](#complete-lifecycle-example)
- [Webhook Event Reference](#webhook-event-reference)

---

## Concepts

### What is an Invoice Payment?

An invoice payment is a **pull payment**: a merchant (or their PIA) requests payment, and the customer pays using their wallet or account through their PRA. Flow coordinates discovery, identity exchange, compliance checks, authorization, and settlement across all parties.

### Roles

| Role | Full Name               | Represents          | Examples                                      |
| ---- | ----------------------- | -------------------- | --------------------------------------------- |
| PIA  | Payment Initiating Agent | Merchant / payee    | Payment processors, billing platforms, SaaS    |
| PRA  | Payment Responding Agent | Payer / customer    | Payout PSPs, wallets, neobanks                 |
| IP   | Infrastructure Provider  | Supporting services  | Custody platforms, MPC wallets, liquidity providers |

**PIA responsibilities:**
- Create invoice payment requests on behalf of merchants
- Specify accepted assets and settlement addresses
- Distribute payment links
- Reconcile incoming funds

**PRA responsibilities:**
- Receive invoice payment requests and surface them to payers
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

Before creating invoice payments, register your customers (merchants for PIAs, payers for PRAs):

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
            "naturalPersonNameIdentifierType": "LEGL"
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

**IVMS101 Validation (strict — unknown fields are rejected):**
- For `natural_person`: `profileData.naturalPerson.name` is **required**
- For `legal_person`: `profileData.legalPerson.name` **and** `profileData.legalPerson.nationalIdentification` are both **required**
- When `nameIdentifier` array is provided, `primaryIdentifier` and `naturalPersonNameIdentifierType` are **required** within each entry
- The API rejects any unrecognized fields in `profileData`

### Supported Assets & Fallback Settlement Addresses

**Supported assets** define which payment methods a PIA accepts for an invoice payment. Each entry is a [CAIP-19](https://github.com/ChainAgnostic/CAIPs/blob/main/CAIPs/caip-19.md) token identifier. We recommend listing stablecoins denominated in the same currency as the invoice amount (e.g., USDC and USDT for a USD invoice) across the chains you support.

**Fallback settlement addresses** are the PIA's receiving addresses — one per blockchain for each supported asset. These are specified as CAIP-10 account identifiers. We recommend that PIAs provide a `fallbackSettlementAddresses` entry for each chain they list in `supportedAssets`, so the system can automatically match the settlement address when the PRA selects an asset. You or your customer can also add bank account details as PayTo URIs (e.g., `payto://iban/DE89370400440532013000`) as a backup for customers who prefer fiat settlement. Customers paying through fiat fallback addresses are not charged through the Flow system — these are provided as a convenience mechanism.

Pre-registering wallet addresses in the Wallet Service is **not required** for Flow. The `fallbackSettlementAddresses` on the invoice payment (or at entity level) are sufficient.

The **PRA** selects one of the `supportedAssets` on behalf of the payer, depending on what the customer has available or by helping them onramp to one of the supported tokens.

### Webhook Setup

Subscribe to Flow webhook events in the Notabene dashboard. See [Webhook Event Reference](#webhook-event-reference) for the full list.

### Delegating Settlement to a Wallet Service (IP)

Both PIAs and PRAs can add a supported **wallet service (Infrastructure Provider / IP)** as an agent to handle all blockchain settlement on their behalf. This lets you focus on the payment and compliance aspects without managing wallets, on-chain transfers, or settlement addresses.

The IP handles: wallet provisioning, asset selection, settlement address management, on-chain execution, and settlement reporting. You handle: creating invoice payments, distributing payment links, authorization decisions, and reconciliation.

See [Infrastructure Provider Flow](#infrastructure-provider-flow-ip) for how IPs are added, and the [Wallet Service Integration Guide](./wallet-service-guide.md) for the full IP perspective.

---

## Initiator Flow (PIA)

### Step 1: Create an Invoice Payment

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
    "eip155:137/erc20:0x2791Bca1f2de4661ED88A30C99A7a9449Aa84174"
  ],
  "fallbackSettlementAddresses": [
    "eip155:1:0xYourEthereumAddress",
    "eip155:137:0xYourPolygonAddress",
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
| `ref`             | string   | Yes      | Unique reference for reconciliation. 1–64 chars, pattern: `^[a-zA-Z0-9_-]{1,64}$` |
| `amount`          | string   | No       | Payment amount. Pattern: `^\d+(\.\d+)?$`          |
| `currency`        | string   | No       | Fiat currency code (e.g., `USD`). 1–3 chars       |
| `asset`                      | string   | No       | Default asset in CAIP-19 format                   |
| `supportedAssets`            | string[] | No       | Up to 10 accepted CAIP-19 token identifiers (see [Supported Assets](#supported-assets--fallback-settlement-addresses)) |
| `fallbackSettlementAddresses`| string[] | No       | PIA's receiving addresses — one CAIP-10 address per chain, plus optional PayTo URIs for fiat (see [Supported Assets](#supported-assets--fallback-settlement-addresses)) |
| `customer`        | object   | Yes      | Payer identity (`@id` required, 3–255 chars; optional `name`, `email`)  |
| `memo`            | string   | No       | Payment description                               |

> **Entity-level invoice payment** (`POST /entities/{entityDID}/flow/payins`): When creating an invoice payment at entity level (without a customer path param), the `merchant` object, `supportedAssets`, and `fallbackSettlementAddresses` fields are all **required** in addition to the fields above.

**Supported assets** list the CAIP-19 tokens you accept. We recommend stablecoins denominated in the invoice currency (e.g., USDC on Ethereum and Polygon for a USD invoice). The PRA selects one of these on behalf of the payer — depending on what the customer has available or by helping them onramp to a supported token.

**Fallback settlement addresses** provide a receiving address for each chain. When the PRA selects an asset, the system matches it to the corresponding `fallbackSettlementAddresses` entry by chain ID. You can also include PayTo URIs (e.g., `payto://iban/DE89370400440532013000`) as a fiat backup — customers paying via fiat fallback are not charged through Flow.

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

Track the invoice payment lifecycle through webhook events:

1. **`flow.payin.created`** - Invoice payment created (confirmation)
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

As a PRA, you receive invoice payment requests and manage the payer's side of the transaction.

### Step 1: Receive the Invoice Payment Request

When a PIA creates an invoice payment that involves your customer, you receive the **`flow.payin.created`** webhook (or `flow.payout.created` depending on direction):

```json
{
  "message": "flow.payin.created",
  "payload": {
    "id": "<transfer-id>",
    "for": "did:web:your-pra.com",
    "initiatedBy": "did:web:pia-payments.com",
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

You can also poll for invoice payments:

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

1. Receive the invoice payment request via webhook or API
2. Provide an authorization URL so the customer can approve the payment (or skip this to auto-authorize)
3. Select the payment asset (asset only, no address)
4. Wait for the PIA's settlement address, then authorize or reject the transfer
5. Send funds on-chain to the PIA's settlement address
6. Report settlement by calling `/settle` with the transaction hash

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

### Step 3: Select Asset

The PRA selects one of the `supportedAssets` from the invoice payment request on behalf of the payer. This is done via a separate asset selection endpoint — **not** as part of the authorize call.

```
POST /entities/{entityDID}/flow/payouts/{transferId}/settlement_asset
Authorization: Bearer <token>
Content-Type: application/json
```

```json
{
  "asset": "eip155:1/erc20:0xA0b86991c6218b36c1d19D4a2e9Eb0cE3606eB48"
}
```

Send **only** the `asset` field. Do **not** include a `settlementAddress` — that refers to the PIA's receiving address (from their `fallbackSettlementAddresses`), not the PRA's sending address. The PIA will confirm which of their addresses to use.

After asset selection, the PIA responds with their settlement address. You receive the **`flow.payin.settlementAddressSelected`** webhook containing the address where funds should be sent.

### Step 4: Authorize or Reject

Once you receive the PIA's settlement address (via the `flow.payin.settlementAddressSelected` webhook), authorize the transfer. The authorize call takes an **empty body** — asset selection was already handled in Step 3.

**Authorize:**

```
POST /entities/{entityDID}/tx/{transferId}/authorize
Authorization: Bearer <token>
Content-Type: application/json
```

```json
{}
```

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

### Step 5: Execute Settlement

Once both parties have authorized and the PIA has provided a settlement address (via the `flow.payin.settlementAddressSelected` webhook), send the funds on-chain to the PIA's settlement address.

### Step 6: Report Settlement

After the on-chain transfer confirms, report the settlement to Notabene with the transaction hash:

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

Infrastructure Providers such as wallet custody platforms, MPC wallet services, or liquidity providers handle the blockchain settlement aspect of a Flow invoice payment on behalf of a PIA or PRA. The IP does not own the customer relationship — it acts as a settlement agent under instruction from its client.

### How IPs Are Added

**By the PIA at creation time:** The PIA typically includes the IP as an agent when creating the invoice payment. This is common when the merchant's custody provider will receive the settled funds:

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
      "role": "Custodian"
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
    "role": "Custodian"
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

                                          6. POST /flow/payouts/{id}/settlement_asset
                                             (selects asset only, no address)
   ← flow.payin.assetSelected

   PIA confirms settlement address
   ← flow.payin.settlementAddressSelected              ← flow.payin.settlementAddressSelected

                                          7. POST /tx/{id}/authorize (empty body)
   8. POST /tx/{id}/authorize
   ← transferStatusChanged(AUTHORIZED)    ← transferStatusChanged(AUTHORIZED)

                                          9. Send funds on-chain (PRA or IP)
                                          10. POST /tx/{id}/settle
   ← transferStatusChanged(SETTLED)       ← transferStatusChanged(SETTLED)

11. Reconcile and mark invoice paid
```

---

## Webhook Event Reference

### For PIAs (Initiators)

| Event                                  | When                                          | What to Do                                      |
| -------------------------------------- | --------------------------------------------- | ----------------------------------------------- |
| `flow.payin.created`                   | Invoice payment request created               | Confirm creation in your system                 |
| `flow.payin.opened`                    | Customer opened payment link                  | Track customer engagement                       |
| `flow.payin.assetSelected`             | Customer selected payment asset               | Track payment progress                          |
| `flow.payin.authorizationRequired`     | PRA provided an authorization URL             | Direct customer to the PRA's authorization page |
| `flow.payin.fundingAdded`             | Funding address details finalized             | Record funding details                          |
| `notification.transferStatusChanged`   | Transfer status changed (e.g., `SETTLED`)     | Authorize, then reconcile when settled          |

### For PRAs (Responders)

| Event                                  | When                                          | What to Do                                      |
| -------------------------------------- | --------------------------------------------- | ----------------------------------------------- |
| `flow.payin.created`                   | Invoice payment request received from PIA     | Surface to payer for approval                   |
| `flow.payin.agentAdded`               | Your agent was added to the transfer          | Confirm agent assignment                        |
| `flow.payin.settlementAddressSelected`             | PIA provided settlement address for funds     | Call `/authorize`, then send funds to this address |
| `notification.transferStatusChanged`   | Transfer status changed (e.g., `AUTHORIZED`)  | Call `/settle` when `toStatus: "AUTHORIZED"`    |

### For IPs (Infrastructure Providers)

| Event                                  | When                                          | What to Do                                      |
| -------------------------------------- | --------------------------------------------- | ----------------------------------------------- |
| `flow.payin.agentAdded`               | You were added to the transfer                | Acknowledge and prepare for settlement          |
| `flow.payin.fundingAdded`             | Settlement address finalized                  | Prepare to send or receive funds                |
| `notification.transferStatusChanged`   | Transfer status changed (e.g., `AUTHORIZED`)  | Execute on-chain transfer, then call `/settle`  |
