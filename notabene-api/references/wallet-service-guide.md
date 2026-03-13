# Wallet Service Integration Guide

This guide explains how a wallet service (Infrastructure Provider / IP) participates in Notabene transactions — both **Flow** (payment) and **Transact** (travel rule transfer) products. 

---

## What is an Infrastructure Provider (IP)?

An IP is a wallet or custody service that handles on-chain operations **on behalf of** another institution. When an originator-side or beneficiary-side institution adds you as their agent in a transaction, you:

1. Provision a wallet for the client
2. Determine your role (originator side or beneficiary side) based on the agent chain
3. Select settlement assets and/or provide settlement addresses
4. Execute on-chain transfers when settlement is required
5. Report settlement back to Notabene

> **Flow terminology:** In Flow transactions, the originator side is called the **PRA** (Payment Responding Agent / payer side) and the beneficiary side is called the **PIA** (Payment Initiating Agent / merchant side). The responsibilities are the same — only the naming differs.

---

## Offering Travel Rule Compliance to Your Customers

As a wallet service, you can offer **travel rule integration as a service** to your customers (exchanges, VASPs, neobanks) by wrapping Notabene's Transfers API within your existing wallet transaction API. Your customers get compliant crypto transfers without building a direct Notabene integration.

### How it works

When your customer initiates a withdrawal or send through your wallet API:

1. **Create the transfer in Notabene** on behalf of your customer using `POST /entities/{entityDID}/tx`. Add your customer as the primary originating agent and yourself as an agent acting on their behalf:

```json
{
  "ref": "withdrawal-001",
  "originator": { "@id": "did:email:end-user@example.com" },
  "beneficiary": { "@id": "did:email:recipient@example.com" },
  "asset": "eip155:1/erc20:0xA0b86991c6218b36c1d19D4a2e9Eb0cE3606eB48",
  "amount": "100.00",
  "agents": [
    {
      "@id": "did:web:your-customer-vasp.com",
      "role": "VASP",
      "for": "did:email:end-user@example.com",
      "policies": [{ "@type": "REQUIRE_AUTHORIZATION" }]
    },
    {
      "@id": "did:web:your-wallet-service.com",
      "role": "IP",
      "for": "did:web:your-customer-vasp.com"
    }
  ]
}
```

2. **Wait for authorization** — your customer's compliance team reviews the transfer through their Notabene dashboard and authorizes or rejects it. You receive `notification.transferStatusChanged` with `toStatus: "AUTHORIZED"` once both your customer and any counterparty compliance checks pass.

3. **Execute on-chain settlement** — once authorized, send the funds on-chain and report settlement via `POST /entities/{entityDID}/tx/{transferId}/settle`.

### Key details

- **Customer DID:** Your customer's entity DID (typically `did:web:customer-domain.com`), obtained during onboarding. You need this to add them as the primary agent.
- **Agent chain:** Your customer is the agent `for` the end-user (originator/beneficiary). You are the agent `for` your customer. This chain determines who handles compliance (your customer) vs. who handles settlement (you).
- **Authorization wait:** This is the critical addition to your existing wallet flow. Do **not** broadcast the on-chain transaction until you receive the `AUTHORIZED` webhook. Your customer's compliance team must approve the transfer first.
- **Incoming transfers:** For deposits, pre-register your customer's addresses via the Relationships API and handle address ownership confirmation on their behalf. See the [Beneficiary Side](#beneficiary-side-you-receive-funds) section.

---

## Offering Billing / Pay-in APIs to Your Customers

You can also wrap Notabene's **Flow Pay-ins API** to offer billing and invoicing capabilities to your customers. Your customers (merchants, SaaS platforms) get compliant payment collection without building a direct Flow integration.

### How it works

When your customer wants to create an invoice or payment request:

1. **Register your customer** as a Flow customer:

```
POST /entities/{entityDID}/flow/customers
```

2. **Create pay-ins on their behalf** using `POST /entities/{entityDID}/flow/customers/{customerDID}/payins`, adding yourself as an agent acting on behalf of the PIA (your customer):

```json
{
  "ref": "INV-2025-001",
  "amount": "1000.00",
  "currency": "USD",
  "supportedAssets": [
    "eip155:1/erc20:0xA0b86991c6218b36c1d19D4a2e9Eb0cE3606eB48"
  ],
  "customer": {
    "@id": "did:email:payer@example.com"
  },
  "agents": [
    {
      "@id": "did:web:your-wallet-service.com",
      "for": "did:web:your-customer-merchant.com",
      "role": "IP"
    }
  ]
}
```

3. **Distribute the payment link** — return the `paymentLink` from the response to your customer for distribution to their payers.

4. **Handle settlement** — when the payer's side authorizes and settles, you receive the settlement on behalf of your customer and credit their account.

This pattern lets you build a complete billing API on top of Notabene Flow, with compliance handled automatically.

---

## Authentication

All API calls require an OAuth2 access token:

```
POST https://auth.notabene.id/oauth/token
Content-Type: application/x-www-form-urlencoded

grant_type=client_credentials
&client_id=YOUR_CLIENT_ID
&client_secret=YOUR_CLIENT_SECRET
&audience=https://api.notabene.dev
```

The token is valid for 24 hours. Cache and refresh it before expiry.

**Base URL:** `https://api.notabene.dev` (or `https://api.eu1.notabene.id` for production)

---

## Webhook Events to Handle

Register a webhook endpoint with Notabene. You will receive events with this structure:

```json
{
  "message": "event.type.here",
  "payload": { "id": "transfer-uuid", ... },
  "version": "1.0.0"
}
```

The event type is in `message`, the transfer ID is in `payload.id`.

### Key events — Flow transactions

| Event | When it fires | What to do |
|-------|---------------|------------|
| `flow.payin.agentAdded` / `flow.payout.agentAdded` | You are added as an agent to a Flow transfer | Provision wallet, determine role, authorize |
| `flow.payin.created` / `flow.payout.created` | A Flow transfer is created involving you | Same as agentAdded (may arrive instead of or before it) |
| `flow.payin.assetSelected` / `flow.payout.assetSelected` | A settlement asset has been selected | Beneficiary side: provide your settlement address |
| `notification.transferStatusChanged` → `AUTHORIZED` | All parties have authorized | Originator side: settle if you have the settlement address |
| `flow.payin.settlementAddressSelected` / `flow.payout.settlementAddressSelected` | Settlement address selected | Originator side: settle if already authorized |
| `notification.transferStatusChanged` → `SETTLED` | Settlement is complete | Beneficiary side: verify funds received, mark complete |

### Key events — Transact (regular transfers)

| Event | When it fires | What to do |
|-------|---------------|------------|
| `notification.transferAgentAdded` | You are added as an agent to a transfer | Provision wallet, determine role |
| `notification.transferCreated` | A transfer is created involving you | Same as transferAgentAdded (may arrive instead of or before it) |
| `tap.requireRelationshipConfirmationRequested` | Address ownership confirmation needed | Confirm or deny that the address belongs to your client |
| `tap.requireAuthorizationRequested` | Authorization decision needed | Review and authorize or reject the transfer |
| `notification.transferStatusChanged` → `AUTHORIZED` | All parties have authorized | Originator side: execute on-chain settlement |
| `notification.transferStatusChanged` → `SETTLED` | Settlement is complete | Beneficiary side: verify funds received |

### Important: Duplicate webhooks

Notabene could send each webhook event more than once. Your handler must be idempotent. Check if you've already processed a transfer before creating records or executing settlements.

---

## Determining Your Role

Your role depends on which side of the transaction your client is on. The transfer has an `agents` array:

```json
{
  "agents": [
    { "agent": { "@id": "did:web:your-service" }, "for": "did:web:client-vasp" },
    { "agent": { "@id": "did:web:client-vasp" }, "for": "did:email:originator@example.com" }
  ],
  "originator": { "@id": "did:email:originator@example.com" },
  "beneficiary": { "@id": "did:email:beneficiary@example.com" }
}
```

To determine your role, trace the `for` chain starting from your entity DID:

1. Build a map: `agent["@id"]` → `for`
2. Starting from your DID, follow the chain: `you → for → for → ...`
3. If the chain reaches `originator["@id"]` → you are on the **originator side** (you send funds)
4. If the chain reaches `beneficiary["@id"]` → you are on the **beneficiary side** (you receive funds)

```
you (IP) → client VASP → originator = originator side (you pay)
you (IP) → client VASP → beneficiary = beneficiary side (you receive)
```

> In Flow transactions, originator side = **PRA**, beneficiary side = **PIA**.

---

## Originator Side (You Send Funds)

When you are on the originator side, you are responsible for authorizing the transfer and executing the on-chain payment once all parties (including your client's compliance team) have authorized.

### Step 1: Provision wallet

When you are added as an agent:
- Create a wallet for the client agent DID if one doesn't exist
- Fund it from a faucet (testnet) or ensure it has sufficient balance

### Step 2: Pre-register addresses (Transact — recommended)

For regular transfers, pre-register your client's deposit addresses so Notabene can automatically route incoming transfers to you:

```
POST /entities/{entityDID}/relationships
Content-Type: application/json
Authorization: Bearer {token}

{
  "from": "did:pkh:eip155:1:0xClientWalletAddress",
  "to": "did:web:client-vasp-did",
  "custodian": "did:web:your-wallet-service.com"
}
```

- `from` — the blockchain address (as a `did:pkh`) that your wallet service controls on behalf of the client
- `to` — your customer's entity DID (the VASP you custody for)
- `custodian` — **your own entity DID** (the wallet service / IP)

The `custodian` field is critical: it tells Notabene that you manage this address on behalf of the client. Without it, Notabene can identify the client VASP but won't know to add you as an agent when transfers arrive at this address.

### Step 3: Select settlement asset (Flow only)

In Flow transactions, check which of the transfer's `supportedAssets` you can actually settle with (i.e., tokens you hold). Pick the one with the highest balance.

Call the **Flow payout** endpoint to declare your chosen asset:

```
POST /entities/{entityDID}/flow/payouts/{transferId}/settlement_asset
Content-Type: application/json
Authorization: Bearer {token}

{ "asset": "eip155:42431/erc20:0x20c0000000000000000000000000000000000002" }
```

**Important:** As originator side, you send only `{ asset }` — no settlement address. The beneficiary side will provide their address after seeing your asset selection.

> In regular transfers, the asset and settlement details are determined by the transfer itself — you do not need to call this endpoint.

### Step 4: Authorize

Authorize the transfer with an empty body:

```
POST /entities/{entityDID}/tx/{transferId}/authorize
Content-Type: application/json
Authorization: Bearer {token}

{}
```

**Important:** Authorization does NOT include asset or address information. In Flow, those are communicated via the `settlement_asset` endpoints.

### Step 5: Wait for settlement conditions

You must wait for **all** conditions before settling on-chain:

1. **All parties authorized** — you receive `notification.transferStatusChanged` with `toStatus: "AUTHORIZED"`. This means both your client and their compliance team have approved the transfer.

**Flow only (additional condition):**

2. **Settlement address available** — you receive `flow.payout.settlementAddressSelected` (the beneficiary side has provided their wallet address)

Either event may arrive first. Only execute settlement when all conditions are met.

### Step 6: Execute on-chain settlement

Once all conditions are satisfied:

1. Parse the settlement address (CAIP-10 format: `eip155:42431:0xRecipient`)
2. Parse the asset (CAIP-19 format: `eip155:42431/erc20:0xTokenAddress`)
3. Execute the ERC-20 `transfer(to, amount)` on-chain
4. Wait for transaction confirmation

### Step 7: Report settlement

```
POST /entities/{entityDID}/tx/{transferId}/settle
Content-Type: application/json
Authorization: Bearer {token}

{ "settlementId": "eip155:42431:tx/0xTransactionHash" }
```

The `settlementId` format is `eip155:{chainId}:tx/{txHash}`.

---

## Beneficiary Side (You Receive Funds)

When you are on the beneficiary side, you confirm address ownership, authorize the transfer, and provide your wallet address so the originator side can send you funds.

### Step 1: Provision wallet

Same as originator side — create and fund a wallet for the client agent DID.

### Step 2: Pre-register addresses (Transact — recommended)

Pre-register your client's receiving addresses so incoming transfers are automatically routed to you:

```
POST /entities/{entityDID}/relationships
Content-Type: application/json
Authorization: Bearer {token}

{
  "from": "did:pkh:eip155:1:0xClientWalletAddress",
  "to": "did:web:client-vasp-did",
  "custodian": "did:web:your-wallet-service.com"
}
```

- `from` — the blockchain address (as a `did:pkh`) that your wallet service controls on behalf of the client
- `to` — your customer's entity DID (the VASP you custody for)
- `custodian` — **your own entity DID** (the wallet service / IP)

The `custodian` field is critical: it tells Notabene that you manage this address on behalf of the client. Without it, incoming transfers to this address will identify the client VASP but Notabene won't automatically add you as an agent — you'd need to be added manually to each transfer.

### Step 3: Confirm address ownership (Transact)

When you receive `tap.requireRelationshipConfirmationRequested`, Notabene is asking you to confirm that an address belongs to your client. The payload includes a `confirmCallbackUrl`:

```
PATCH /entities/{entityDID}/relationships?from={addressDID}&to={customerDID}
Content-Type: application/json
Authorization: Bearer {token}

{}
```

If the address does not belong to your client, do not confirm — the transfer will be handled through other channels.

> **Tip:** If you pre-registered addresses in Step 2, many relationship confirmations will be satisfied automatically.

### Step 4: Authorize

Authorize with an empty body:

```
POST /entities/{entityDID}/tx/{transferId}/authorize
Content-Type: application/json
Authorization: Bearer {token}

{}
```

You don't select an asset — the originator side does that (in Flow).

### Step 5: Provide settlement address (Flow only)

When you receive `assetSelected` (meaning the originator side has selected an asset), provide your wallet address via the **Flow payin** endpoint:

```
POST /entities/{entityDID}/flow/payins/{transferId}/settlement_address
Content-Type: application/json
Authorization: Bearer {token}

{
  "asset": "eip155:42431/erc20:0x20c0000000000000000000000000000000000002",
  "settlementAddress": "eip155:42431:0xYourWalletAddress"
}
```

**Important:** As beneficiary side, you send both `asset` and `settlementAddress`. The asset should match what the originator side selected. The address must be in your entity's `fallbackSettlementAddresses` list (configured in the Notabene dashboard).

### Step 6: Reconcile received transfers (Transact)

When you detect an on-chain deposit to one of your client's addresses, use the **match endpoint** to find the corresponding Notabene transfer:

```
GET /entities/{entityDID}/tx/match?settlement_id={txHash}&settlement_address={destinationAddress}&direction=INCOMING
Authorization: Bearer {token}
```

| Parameter | Required | Description |
|-----------|----------|-------------|
| `settlement_id` | yes | On-chain transaction hash |
| `settlement_address` | yes | Destination address that received funds |
| `settlement_id_index` | no | Vout or log index (for UTXO chains or multi-transfer txs) |
| `memo_tag` | no | Memo/tag for chains that use them (e.g., XRP, XLM) |
| `direction` | no | `INCOMING` or `OUTGOING` |

The response includes `meta.match_strategy`: `strict` (exact match), `broadened` (relaxed index matching), or `none` (no match found).

If no match is found, create the transfer in Notabene so it can be tracked for compliance:

```
POST /entities/{entityDID}/tx
Content-Type: application/json
Authorization: Bearer {token}

{
  "ref": "your-unique-deposit-reference",
  "originator": { "@id": "did:pkh:eip155:1:0xSenderAddress" },
  "beneficiary": { "@id": "did:web:client-vasp-did" },
  "asset": "eip155:1/erc20:0xTokenAddress",
  "amount": "100.00",
  "agents": [
    {
      "@id": "did:web:your-service.example.com",
      "role": "VASP",
      "for": "did:web:client-vasp-did",
      "policies": [{ "@type": "REQUIRE_AUTHORIZATION" }]
    }
  ],
  "settlementId": "0xTransactionHash"
}
```

This ensures the deposit enters Notabene's compliance workflow. Notabene will attempt to identify the originator VASP and request travel rule data from them.

**Recommended pattern:** When you detect an on-chain deposit to a client address, call the match endpoint. If a transfer is found, link it to your internal record. If no match is found, create the transfer in Notabene so it can be reconciled and compliance-tracked.

### Step 7: Verify settlement

When you receive `notification.transferStatusChanged` with `toStatus: "SETTLED"`, verify that you received the funds on-chain (optional but recommended).

---

## API Endpoints Summary

### Common (both Flow and Transact)

| Action | Method | Endpoint | Body |
|--------|--------|----------|------|
| Get transfer details | GET | `/entities/{entityDID}/tx/{transferId}` | — |
| Authorize | POST | `/entities/{entityDID}/tx/{transferId}/authorize` | `{}` |
| Report settlement | POST | `/entities/{entityDID}/tx/{transferId}/settle` | `{ "settlementId": "..." }` |
| Reject | POST | `/entities/{entityDID}/tx/{transferId}/reject` | `{ "reason": "...", "comment": "..." }` |

### Flow only

| Action | Method | Endpoint | Body |
|--------|--------|----------|------|
| Select asset (originator side) | POST | `/entities/{entityDID}/flow/payouts/{transferId}/settlement_asset` | `{ "asset": "..." }` |
| Select asset + address (beneficiary side) | POST | `/entities/{entityDID}/flow/payins/{transferId}/settlement_asset` | `{ "asset": "...", "settlementAddress": "..." }` |

### Transact only

| Action | Method | Endpoint | Body |
|--------|--------|----------|------|
| Pre-register address | POST | `/entities/{entityDID}/relationships` | `{ "from": "did:pkh:...", "to": "did:web:...", "custodian": "did:web:..." }` |
| Confirm relationship | PATCH | `/entities/{entityDID}/relationships?from=...&to=...` | `{}` |
| Match on-chain deposit | GET | `/entities/{entityDID}/tx/match?settlement_id=...&settlement_address=...` | — |

---

## Transfer Details (GET response)

Key fields on the transfer object:

```json
{
  "id": "transfer-uuid",
  "amount": "5000",
  "supportedAssets": ["eip155:42431/erc20:0x...", "USD"],
  "fallbackSettlementAddresses": ["eip155:42431:0x..."],
  "agents": [
    { "agent": { "@id": "did:web:..." }, "for": "did:web:..." }
  ],
  "originator": { "@id": "did:email:..." },
  "beneficiary": { "@id": "did:email:..." }
}
```

- `amount` — the transfer amount (human-readable, e.g., `"5000"` for $50.00 with 6-decimal tokens)
- `supportedAssets` — CAIP-19 identifiers the transfer supports (may include fiat like `"USD"`)
- `fallbackSettlementAddresses` — CAIP-10 addresses for settlement (updated as parties provide them)
- `agents` — the agent chain (use this for role determination)
- `originator["@id"]` / `beneficiary["@id"]` — the end-party DIDs

---

## Preventing Double Settlement

Multiple webhook events can trigger settlement logic concurrently (e.g., `AUTHORIZED` and `settlementAddressSelected` arriving at the same time). You must use an **atomic claim mechanism** to ensure only one handler executes the on-chain transfer.

Pattern:
1. Store a `status` field on your transfer record
2. Before settling, atomically transition `authorized` → `settling`
3. If the transition fails (status was already `settling` or `settled`), skip
4. Only the handler that successfully claims the transition executes the on-chain transfer

```
// Pseudocode
function claimForSettlement(transferId):
  transfer = db.get(transferId)
  if transfer.status != "authorized": return false
  db.update(transferId, { status: "settling" })  // atomic
  return true
```

---

## Common Pitfalls

### 1. Authorizing with asset/address in the body
The `authorize` endpoint takes an **empty body** `{}`. In Flow, asset selection and address provisioning happen via the separate `settlement_asset` endpoints.

### 2. Originator side providing a settlement address (Flow)
The originator side selects the **asset only** (no address). The beneficiary side provides the settlement address. If the originator side includes an address that isn't in the beneficiary's `fallbackSettlementAddresses`, the API returns a 400 error.

### 3. Settling before all parties authorize
You must wait for `notification.transferStatusChanged` with `toStatus: "AUTHORIZED"` (meaning ALL parties have authorized — including your client and their compliance team) before executing on-chain settlement. The per-agent `notification.transferAgentStatusChanged` events fire when individual agents authorize — this is not sufficient.

### 4. Settling before the settlement address is available (Flow)
The originator side selects the asset, then the beneficiary side provides their settlement address. These happen asynchronously. You need both the `AUTHORIZED` status and a settlement address before you can settle.

### 5. Not handling duplicate webhooks
Every webhook event is delivered at least twice. Without deduplication, you may create duplicate records, execute duplicate on-chain transfers, or hit nonce collisions.

### 6. Not handling event ordering
`AUTHORIZED`, `settlementAddressSelected`, and `agentAdded` events can arrive in any order. Design your handlers to work regardless of which event arrives first — check current state and act accordingly.

### 7. Not pre-registering addresses (Transact)
Without pre-registered relationships, every incoming transfer will require manual relationship confirmation. Pre-register addresses via the relationship API to automate this.

### 8. Omitting the custodian field when registering addresses
When calling `POST /entities/{entityDID}/relationships`, wallet services must include `"custodian": "did:web:your-wallet-service.com"` alongside `from` and `to`. Without the `custodian` field, Notabene identifies the client VASP as the address owner but does not know that a wallet service manages the address — so it cannot automatically add you as an agent to incoming transfers.

---

## Identifier Formats

| Format | Example | Description |
|--------|---------|-------------|
| CAIP-19 (asset) | `eip155:42431/erc20:0x20c0...0002` | Chain + token contract |
| CAIP-10 (account) | `eip155:42431:0x742d35Cc...` | Chain + wallet address |
| Settlement ID | `eip155:42431:tx/0xabc123...` | Chain + transaction hash |
| DID (entity) | `did:web:your-service.example.com` | Decentralized identifier |
| DID (address) | `did:pkh:eip155:1:0x742d35Cc...` | Blockchain address as DID |

---

## Environment Variables

```
NB_CLIENT_ID=<from notabene dashboard>
NB_CLIENT_SECRET=<from notabene dashboard>
NB_ENTITY_DID=did:web:your-service.example.com
NB_API_URL=https://api.notabene.dev
NOTABENE_OAUTH_TOKEN_URL=https://auth.notabene.id/oauth/token
NB_OAUTH_AUDIENCE=https://api.notabene.dev
```
