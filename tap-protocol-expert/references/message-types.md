# TAP Message Types — Complete Field Reference

All TAP message types with full field specifications. Every message body is wrapped in a DIDComm v2 envelope (see TAIP-2).

## DIDComm v2 Envelope (all messages)

```json
{
  "id": "uuid",
  "type": "https://tap.rsvp/schema/1.0#<MessageType>",
  "from": "did:web:sender.example.com",
  "to": ["did:web:recipient.example.com"],
  "thid": "thread-id (= original Transfer.id for all replies)",
  "pthid": "parent-thread-id (e.g. connection id from TAIP-15)",
  "created_time": 1700000000,
  "expires_time": 1700003600,
  "body": { /* TAP message body */ }
}
```

`thid` is omitted from the first message in a thread. For all subsequent messages, `thid` = the original initiating message's `id`.

---

## TAIP-3: Transfer

**Type:** `https://tap.rsvp/schema/1.0#Transfer`

```typescript
{
  "@context": "https://tap.rsvp/schema/1.0",
  "@type": "https://tap.rsvp/schema/1.0#Transfer",

  // Asset and amount
  asset: string,        // Required. CAIP-19 asset identifier
  amount?: string,      // Required for fungible tokens; optional for NFTs. Decimal string (e.g., "100.00")

  // Parties (both optional in spec; convention is to include them)
  originator?: Party,   // TAIP-6 Party object (sender)
  beneficiary?: Party,  // TAIP-6 Party object (receiver)

  // Agents
  agents: Agent[],      // Required. TAIP-5 Agent objects. At least one MUST match
                        // the DIDComm `from` and have `for` set to the originator
                        // or beneficiary DID

  // Optional
  settlementId?: string,  // CAIP-220 blockchain tx ID (if already settled)
  memo?: string,          // Free-text note
  expiry?: string,        // ISO 8601 datetime

  // Fiat-equivalent value (added 2025-08-21) — useful for Travel Rule threshold
  // determination when the asset is not widely traded and receiving agents
  // cannot easily resolve fiat value
  transactionValue?: {
    amount: string,    // Required. Decimal string of fiat amount
    currency: string,  // Required. ISO 4217 3-letter code ("USD", "EUR", ...)
  },

  // TAIP-13 additions
  purpose?: string,       // ISO 20022 ExternalPurpose1Code (e.g., "SALA")
  categoryPurpose?: string, // ISO 20022 ExternalCategoryPurpose1Code
}
```

---

## TAIP-4: Authorization Messages

### Authorize

**Type:** `https://tap.rsvp/schema/1.0#Authorize`

```typescript
{
  "@context": "https://tap.rsvp/schema/1.0",
  "@type": "https://tap.rsvp/schema/1.0#Authorize",

  settlementAddress?: string, // CAIP-10 address to send funds to
  settlementAsset?: string,   // CAIP-19 if accepting different asset
  amount?: string,            // Decimal string if accepting different amount
  expiry?: string,            // ISO 8601 — when this authorization expires
}
```

### Settle

**Type:** `https://tap.rsvp/schema/1.0#Settle`

```typescript
{
  "@context": "https://tap.rsvp/schema/1.0",
  "@type": "https://tap.rsvp/schema/1.0#Settle",

  settlementAddress: string,  // Required. CAIP-10 address funds were sent to
  settlementId: string,       // Required. CAIP-220 blockchain transaction ID
  amount?: string,            // Decimal string (actual settled amount)
}
```

### Reject

**Type:** `https://tap.rsvp/schema/1.0#Reject`

```typescript
{
  "@context": "https://tap.rsvp/schema/1.0",
  "@type": "https://tap.rsvp/schema/1.0#Reject",

  reason: string,  // Required. Human-readable rejection reason
}
```

### Cancel

**Type:** `https://tap.rsvp/schema/1.0#Cancel`

```typescript
{
  "@context": "https://tap.rsvp/schema/1.0",
  "@type": "https://tap.rsvp/schema/1.0#Cancel",

  by: string,     // Required. DID of the party cancelling
  reason?: string,
}
```

### Revert

**Type:** `https://tap.rsvp/schema/1.0#Revert`

```typescript
{
  "@context": "https://tap.rsvp/schema/1.0",
  "@type": "https://tap.rsvp/schema/1.0#Revert",

  settlementAddress: string,  // Required. CAIP-10 address for the return
  reason?: string,
}
```

### AuthorizationRequired

**Type:** `https://tap.rsvp/schema/1.0#AuthorizationRequired`

```typescript
{
  "@context": "https://tap.rsvp/schema/1.0",
  "@type": "https://tap.rsvp/schema/1.0#AuthorizationRequired",

  authorizationUrl: string,  // Required. URL to complete authorization
  expires: string,           // Required. ISO 8601 when URL expires
}
```

---

## TAIP-5: Agent Management Messages

### AddAgents

**Type:** `https://tap.rsvp/schema/1.0#AddAgents`

```typescript
{
  "@context": "https://tap.rsvp/schema/1.0",
  "@type": "https://tap.rsvp/schema/1.0#AddAgents",

  agents: Agent[],  // Required. Array of TAIP-5 Agent objects to add
}
```

### ReplaceAgent

**Type:** `https://tap.rsvp/schema/1.0#ReplaceAgent`

```typescript
{
  "@context": "https://tap.rsvp/schema/1.0",
  "@type": "https://tap.rsvp/schema/1.0#ReplaceAgent",

  original: string,     // Required. DID of agent being replaced
  replacement: Agent,   // Required. New TAIP-5 Agent object
}
```

### RemoveAgent

**Type:** `https://tap.rsvp/schema/1.0#RemoveAgent`

```typescript
{
  "@context": "https://tap.rsvp/schema/1.0",
  "@type": "https://tap.rsvp/schema/1.0#RemoveAgent",

  agent: string,  // Required. DID of agent to remove
}
```

---

## TAIP-6: UpdateParty

**Type:** `https://tap.rsvp/schema/1.0#UpdateParty`

```typescript
{
  "@context": "https://tap.rsvp/schema/1.0",
  "@type": "https://tap.rsvp/schema/1.0#UpdateParty",

  partyType: "originator" | "beneficiary",  // Required
  party: Party,  // Required. Updated TAIP-6 Party object
}
```

---

## TAIP-7: UpdatePolicies

**Type:** `https://tap.rsvp/schema/1.0#UpdatePolicies`

```typescript
{
  "@context": "https://tap.rsvp/schema/1.0",
  "@type": "https://tap.rsvp/schema/1.0#UpdatePolicies",

  policies: {
    [agentDID: string]: Policy[]  // Map of DID → policies array
  }
}
```

**Policy objects:**

```typescript
// RequireAuthorization
{ "@type": "RequireAuthorization", from?: string, fromRole?: string, fromAgent?: string }

// RequirePresentation
{ "@type": "RequirePresentation", presentationDefinition: string /* URL */ }

// RequireRelationshipConfirmation
{ "@type": "RequireRelationshipConfirmation", nonce: string }

// RequirePurpose (TAIP-13)
{ "@type": "RequirePurpose", fields: ("purpose" | "categoryPurpose")[] }
```

---

## TAIP-8: Present Proof (WACI)

**Type:** `https://didcomm.org/present-proof/3.0/presentation`

Note: This is a WACI/DIDComm standard message type, not a custom TAP type. Must be a JWE encrypted message.

```typescript
{
  // Standard DIDComm envelope (encrypted)
  "presentations~attach": [{
    "@id": "attachment-id",
    "mime-type": "application/ld+json",
    "data": {
      "json": {
        "@context": ["https://www.w3.org/2018/credentials/v1"],
        "type": ["VerifiablePresentation"],
        "verifiableCredential": [/* W3C VCs */]
      }
    }
  }]
}
```

---

## TAIP-9: ConfirmRelationship

**Type:** `https://tap.rsvp/schema/1.0#ConfirmRelationship`

```typescript
{
  "@context": "https://tap.rsvp/schema/1.0",
  "@type": "https://tap.rsvp/schema/1.0#ConfirmRelationship",

  // Optional CACAO attachment (CAIP-74) for cryptographic wallet proof
  "attachments"?: [{
    "@id": "cacao-id",
    "mime-type": "application/json",
    "data": { "json": { /* CACAO object */ } }
  }]
}
```

---

## TAIP-14: Payment

**Type:** `https://tap.rsvp/schema/1.0#Payment`

```typescript
{
  "@context": "https://tap.rsvp/schema/1.0",
  "@type": "https://tap.rsvp/schema/1.0#Payment",

  // One of these is required:
  asset?: string,     // CAIP-19 (preferred payment asset)
  currency?: string,  // ISO 4217 (for fiat-denominated requests)

  amount?: string,    // Decimal string

  // Multi-asset support
  supportedAssets?: string[] | { asset: string, amount: string }[],

  // Fallback addresses if customer doesn't explicitly Authorize
  fallbackSettlementAddresses?: {
    [caip19Asset: string]: string  // CAIP-19 → CAIP-10
  },

  expiry?: string,     // ISO 8601
  invoice?: Invoice,   // TAIP-16 Invoice object
  policies?: Policy[], // Required policies
  agents?: Agent[],    // TAIP-5 agents

  // Parties use merchant/customer roles
  merchant?: Party,    // TAIP-6 Party (the payee)
  customer?: Party,    // TAIP-6 Party (the payer)
}
```

---

## TAIP-15: Connect

**Type:** `https://tap.rsvp/schema/1.0#Connect`

```typescript
{
  "@context": "https://tap.rsvp/schema/1.0",
  "@type": "https://tap.rsvp/schema/1.0#Connect",

  requester: Party,   // Required. TAIP-6 Party requesting the connection
  principal: Party,   // Required. TAIP-6 Party being connected to

  agents?: Agent[],   // TAIP-5 agents

  constraints?: {
    purposes?: string[],          // ISO 20022 purpose codes allowed
    categoryPurposes?: string[],  // ISO 20022 category purpose codes
    limits?: {
      per_transaction?: string,   // Max per single tx (decimal string)
      day?: string,               // Daily limit
      week?: string,              // Weekly limit
      month?: string,             // Monthly limit
      year?: string,              // Annual limit
      currency: string,           // Required if any limits set (ISO 4217)
    },
    allowedBeneficiaries?: string[],         // Allowed recipient DIDs
    allowedSettlementAddresses?: string[],   // CAIP-10 allowed addresses
    allowedAssets?: string[],                // CAIP-19 allowed assets
  },

  agreement?: string,   // URL to legal terms/agreement
  expiry?: string,      // ISO 8601 when connection expires

  attachments?: object[], // DIDComm attachments (e.g., Transfer to auto-authorize)
}
```

---

## TAIP-16: Invoice (embedded in Payment)

Not a standalone message — embedded as `invoice` field in TAIP-14 Payment.

```typescript
{
  id: string,           // Required. Unique invoice ID
  issueDate: string,    // Required. ISO 8601 date
  currencyCode: string, // Required. ISO 4217 (e.g., "USD")
  total: string,        // Required. Total amount (must match Payment.amount)

  lineItems: [{
    id: string,          // Required. Line item ID
    description: string, // Required. Description
    quantity: string,    // Required. Quantity (decimal string)
    unitPrice: string,   // Required. Unit price (decimal string)
    lineTotal: string,   // Required. Line total (decimal string)
  }],

  // Optional
  subTotal?: string,     // Subtotal before tax
  taxTotal?: string,     // Total tax amount
  dueDate?: string,      // ISO 8601 payment due date
  note?: string,         // Free text note
  paymentTerms?: string, // Payment terms description
  accountingCost?: string,
  orderReference?: string,
  additionalDocumentReference?: string,
}
```

---

## TAIP-17: Composable Escrow Messages

> **Renamed 2026-05-01:** the message type formerly called `Escrow` is now `Lock`. The TAIP title remains "Composable Escrow" and the agent role remains `EscrowAgent`.

### Lock

**Type:** `https://tap.rsvp/schema/1.0#Lock`

```typescript
{
  "@context": "https://tap.rsvp/schema/1.0",
  "@type": "https://tap.rsvp/schema/1.0#Lock",

  // One of these is required:
  asset?: string,    // CAIP-19
  currency?: string, // ISO 4217

  amount: string,    // Required. Decimal string
  expiry: string,    // Required. ISO 8601 — escrow always expires

  originator: Party,  // Required. TAIP-6 Party (depositor — funds will come from here)
  beneficiary: Party, // Required. TAIP-6 Party (recipient on capture)

  agreement?: string, // URL to escrow terms

  agents: Agent[],    // Required. Must include exactly one with role: "EscrowAgent"
}
```

### Capture

**Type:** `https://tap.rsvp/schema/1.0#Capture`

```typescript
{
  "@context": "https://tap.rsvp/schema/1.0",
  "@type": "https://tap.rsvp/schema/1.0#Capture",
  // thid links to the original Lock message

  amount?: string,           // Decimal string (≤ original lock; omit = full amount)
  settlementAddress?: string, // CAIP-10 address for funds release (omit = use earlier Authorize address)
}
```

State transitions reuse `Authorize`, `Settle`, `Cancel`, `Reject`, `Revert` from TAIP-4. See `taip-catalog.md` for the full lifecycle.

---

## TAIP-18: Asset Exchange Messages

> **Renamed 2026-05-01:** the message type formerly called `Exchange` is now `RFQ` (Request for Quote). The TAIP title remains "Asset Exchange".

### RFQ

**Type:** `https://tap.rsvp/schema/1.0#RFQ`

```typescript
{
  "@context": "https://tap.rsvp/schema/1.0",
  "@type": "https://tap.rsvp/schema/1.0#RFQ",

  fromAssets: string[],  // Required. Array of CAIP-19 / DTI / ISO-4217 (available source assets)
  toAssets: string[],    // Required. Array of CAIP-19 / DTI / ISO-4217 (desired target assets)

  // One of these is required:
  fromAmount?: string,   // Amount to send (decimal string)
  toAmount?: string,     // Amount to receive (decimal string)

  requester: Party,      // Required. TAIP-6 Party seeking the exchange
  provider?: Party,      // Optional. TAIP-6 Party of preferred liquidity provider.
                         // Omit to broadcast to multiple providers.

  agents: Agent[],       // Required. TAIP-5 agents (per RFQ definition)
  policies?: Policy[],   // Optional TAIP-7 policies
}
```

### Quote

**Type:** `https://tap.rsvp/schema/1.0#Quote`

```typescript
{
  "@context": "https://tap.rsvp/schema/1.0",
  "@type": "https://tap.rsvp/schema/1.0#Quote",
  // thid links to the original RFQ message

  fromAsset: string,    // Required. CAIP-19 / DTI / ISO-4217 (copied from RFQ)
  toAsset: string,      // Required. CAIP-19 / DTI / ISO-4217 (copied from RFQ)
  fromAmount: string,   // Required. Decimal string
  toAmount: string,     // Required. Decimal string

  provider: Party,      // Required. TAIP-6 Party of the liquidity provider

  agents: Agent[],      // Required. Must include all agents from the original RFQ
                        // plus any additional provider agents
  expires: string,      // Required. ISO 8601 when quote expires
}
```

After a Quote, acceptance and settlement reuse TAIP-4:
- **Accept:** `Authorize` with `thid` = Quote's `id`, plus `settlementAddress`, `settlementAsset` (= `toAsset`), and `amount` (= `toAmount`).
- **Settle:** `Settle` with `settlementId` (CAIP-220 tx hash) once the swap executes on-chain.

---

## TAIP-19: No New Message Types

TAIP-19 only defines ISO 20022 field mappings onto existing TAP messages. See `taip-catalog.md` for mapping tables.

**payto:// URI scheme** (RFC 8905) used in `settlementAddress` for traditional accounts:
```
payto://iban/DE89370400440532013000
payto://ach/021000021/123456789
payto://bic/DEUTDEDB
payto://sortcode/040004/01001234
```

---

## TAIP-20: No New Message Types

TAIP-20 is a settlement-layer convention, not a TAP message. It defines a deterministic on-chain memo derived from the TAP transfer ID:

```
tap_hash = SHA-256(UTF8(tap_transfer_id))
```

`tap_transfer_id` is one of:
- `Transfer.id` (TAIP-3)
- `Payment.id` (TAIP-14)
- the active settlement-thread `thid`

**Profile A — text memo:** `tap:1:<64-lowercase-hex-of-tap_hash>`
**Profile B — binary/hash memo:** raw 32-byte `tap_hash`

The memo lives in the chain's native reference field (Stellar memos, Cosmos `TxBody.memo`, Solana memo extensions, Tempo, ARC-style payment APIs, etc.). See `taip-catalog.md` for full encoding rules and verification flow.
