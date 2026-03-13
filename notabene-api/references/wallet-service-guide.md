# Wallet Service Integration Guide for Notabene Flow

This guide explains how a wallet service (Infrastructure Provider / IP) participates in Notabene Flow transactions. It is based on real implementation experience building a custodial wallet agent on the Tempo testnet.

---

## What is an Infrastructure Provider (IP)?

An IP is a wallet or custody service that handles on-chain operations on behalf of a PIA (Payment Initiating Agent / merchant side) or PRA (Payment Responding Agent / payer side). When a PIA or PRA adds you as their agent in a Flow transaction, you:

1. Provision a wallet for the client
2. Determine your role (PRA or PIA) based on the agent chain
3. Select settlement assets and/or provide settlement addresses
4. Execute on-chain transfers when settlement is required
5. Report settlement back to Notabene

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

### Key events

| Event | When it fires | What to do |
|-------|---------------|------------|
| `flow.payin.agentAdded` / `flow.payout.agentAdded` | You are added as an agent to a transfer | Provision wallet, determine role, authorize |
| `flow.payin.created` / `flow.payout.created` | A transfer is created involving you | Same as agentAdded (may arrive instead of or before it) |
| `flow.payin.assetSelected` / `flow.payout.assetSelected` | A settlement asset has been selected | PIA: provide your settlement address |
| `notification.transferStatusChanged` → `AUTHORIZED` | All parties have authorized | PRA: settle if you have the settlement address |
| `flow.payin.settlementAddressSelected` / `flow.payout.settlementAddressSelected` | Settlement address selected | PRA: settle if already authorized |
| `notification.transferStatusChanged` → `SETTLED` | Settlement is complete | PIA: verify funds received, mark complete |

### Important: Duplicate webhooks

Notabene could sends each webhook event more than once. Your handler must be idempotent. Check if you've already processed a transfer before creating records or executing settlements.

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
3. If the chain reaches `originator["@id"]` → you are the **PRA** (payer side, you send funds)
4. If the chain reaches `beneficiary["@id"]` → you are the **PIA** (payee side, you receive funds)

```
you (IP) → client VASP → originator = PRA (you pay)
you (IP) → client VASP → beneficiary = PIA (you receive)
```

---

## PRA Flow (Payer Side — You Send Funds)

When you are the PRA, you are responsible for selecting the settlement asset, authorizing the transfer, and executing the on-chain payment.

### Step 1: Provision wallet

When you receive `agentAdded` or `created`:
- Create a wallet for the client agent DID if one doesn't exist
- Fund it from a faucet (testnet) or ensure it has sufficient balance

### Step 2: Select settlement asset

Check which of the transfer's `supportedAssets` you can actually settle with (i.e., tokens you hold). Pick the one with the highest balance.

Call the **Flow payout** endpoint to declare your chosen asset:

```
POST /entities/{entityDID}/flow/payouts/{transferId}/settlement_asset
Content-Type: application/json
Authorization: Bearer {token}

{ "asset": "eip155:42431/erc20:0x20c0000000000000000000000000000000000002" }
```

**Important:** As PRA, you send only `{ asset }` — no settlement address. The PIA will provide their address after seeing your asset selection.

### Step 3: Authorize

Authorize the transfer with an empty body:

```
POST /entities/{entityDID}/tx/{transferId}/authorize
Content-Type: application/json
Authorization: Bearer {token}

{}
```

**Important:** Authorization does NOT include asset or address information. Those are communicated via the `settlement_asset` endpoints.

### Step 4: Wait for settlement conditions

You must wait for **both** conditions before settling on-chain:

1. **All parties authorized** — you receive `notification.transferStatusChanged` with `toStatus: "AUTHORIZED"`
2. **Settlement address available** — you receive `flow.payout.settlementAddressSelected` (the PIA has provided their wallet address)

Either event may arrive first. Only execute settlement when both conditions are met.

### Step 5: Execute on-chain settlement

Once both conditions are satisfied:

1. Parse the settlement address (CAIP-10 format: `eip155:42431:0xRecipient`)
2. Parse the asset (CAIP-19 format: `eip155:42431/erc20:0xTokenAddress`)
3. Execute the ERC-20 `transfer(to, amount)` on-chain
4. Wait for transaction confirmation

### Step 6: Report settlement

```
POST /entities/{entityDID}/tx/{transferId}/settle
Content-Type: application/json
Authorization: Bearer {token}

{ "settlementId": "eip155:42431:tx/0xTransactionHash" }
```

The `settlementId` format is `eip155:{chainId}:tx/{txHash}`.

---

## PIA Flow (Payee Side — You Receive Funds)

When you are the PIA, you authorize the transfer and provide your wallet address so the PRA can send you funds.

### Step 1: Provision wallet

Same as PRA — create and fund a wallet for the client agent DID.

### Step 2: Authorize

Authorize immediately with an empty body:

```
POST /entities/{entityDID}/tx/{transferId}/authorize
Content-Type: application/json
Authorization: Bearer {token}

{}
```

You don't select an asset — the PRA does that.

### Step 3: Provide settlement address

When you receive `assetSelected` (meaning the PRA has selected an asset), provide your wallet address via the **Flow payin** endpoint:

```
POST /entities/{entityDID}/flow/payins/{transferId}/settlement_asset
Content-Type: application/json
Authorization: Bearer {token}

{
  "asset": "eip155:42431/erc20:0x20c0000000000000000000000000000000000002",
  "settlementAddress": "eip155:42431:0xYourWalletAddress"
}
```

**Important:** As PIA, you send both `asset` and `settlementAddress`. The asset should match what the PRA selected. The address must be in your entity's `fallbackSettlementAddresses` list (configured in the Notabene dashboard).

### Step 4: Verify settlement

When you receive `notification.transferStatusChanged` with `toStatus: "SETTLED"`, verify that you received the funds on-chain (optional but recommended).

---

## API Endpoints Summary

| Action | Method | Endpoint | Body |
|--------|--------|----------|------|
| Get transfer details | GET | `/entities/{entityDID}/tx/{transferId}` | — |
| Authorize | POST | `/entities/{entityDID}/tx/{transferId}/authorize` | `{}` |
| Select asset (PRA) | POST | `/entities/{entityDID}/flow/payouts/{transferId}/settlement_asset` | `{ "asset": "..." }` |
| Select asset + address (PIA) | POST | `/entities/{entityDID}/flow/payins/{transferId}/settlement_asset` | `{ "asset": "...", "settlementAddress": "..." }` |
| Report settlement | POST | `/entities/{entityDID}/tx/{transferId}/settle` | `{ "settlementId": "..." }` |
| Reject | POST | `/entities/{entityDID}/tx/{transferId}/reject` | `{ "reason": "...", "comment": "..." }` |

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
The `authorize` endpoint takes an **empty body** `{}`. Asset selection and address provisioning happen via the separate `settlement_asset` endpoints.

### 2. PRA providing a settlement address
The PRA selects the **asset only** (no address). The PIA provides the settlement address. If the PRA includes an address that isn't in the PIA's `fallbackSettlementAddresses`, the API returns a 400 error.

### 3. Settling before all parties authorize
You must wait for `notification.transferStatusChanged` with `toStatus: "AUTHORIZED"` (meaning ALL parties have authorized) before executing on-chain settlement. The per-agent `notification.transferAgentStatusChanged` events fire when individual agents authorize — this is not sufficient.

### 4. Settling before the settlement address is available
The PRA selects the asset, then the PIA provides their settlement address. These happen asynchronously. You need both the `AUTHORIZED` status and a settlement address before you can settle.

### 5. Not handling duplicate webhooks
Every webhook event is delivered at least twice. Without deduplication, you may create duplicate records, execute duplicate on-chain transfers, or hit nonce collisions.

### 6. Not handling event ordering
`AUTHORIZED`, `settlementAddressSelected`, and `agentAdded` events can arrive in any order. Design your handlers to work regardless of which event arrives first — check current state and act accordingly.

---

## Identifier Formats

| Format | Example | Description |
|--------|---------|-------------|
| CAIP-19 (asset) | `eip155:42431/erc20:0x20c0...0002` | Chain + token contract |
| CAIP-10 (account) | `eip155:42431:0x742d35Cc...` | Chain + wallet address |
| Settlement ID | `eip155:42431:tx/0xabc123...` | Chain + transaction hash |
| DID (entity) | `did:web:your-service.example.com` | Decentralized identifier |

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
