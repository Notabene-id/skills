# TAIP Catalog — All 19 TAIPs

Complete reference for all Transaction Authorization Improvement Proposals.

## Table of Contents
- [TAIP-1: Framework & Meta](#taip-1-framework--meta)
- [TAIP-2: Messaging](#taip-2-messaging)
- [TAIP-3: Asset Transfer](#taip-3-asset-transfer)
- [TAIP-4: Authorization Flow](#taip-4-authorization-flow)
- [TAIP-5: Transaction Agents](#taip-5-transaction-agents)
- [TAIP-6: Transaction Parties](#taip-6-transaction-parties)
- [TAIP-7: Agent Policies](#taip-7-agent-policies)
- [TAIP-8: Selective Disclosure](#taip-8-selective-disclosure)
- [TAIP-9: Proof of Relationship](#taip-9-proof-of-relationship)
- [TAIP-10: IVMS101 / Travel Rule](#taip-10-ivms101--travel-rule)
- [TAIP-11: Legal Entity Identifier](#taip-11-legal-entity-identifier)
- [TAIP-12: Hashed Name](#taip-12-hashed-name)
- [TAIP-13: Purpose Codes](#taip-13-purpose-codes)
- [TAIP-14: Payments](#taip-14-payments)
- [TAIP-15: Agent Connection Protocol](#taip-15-agent-connection-protocol)
- [TAIP-16: Invoices](#taip-16-invoices)
- [TAIP-17: Composable Escrow](#taip-17-composable-escrow)
- [TAIP-18: Asset Exchange](#taip-18-asset-exchange)
- [TAIP-19: ISO 20022 Mapping](#taip-19-iso-20022-mapping)

---

## TAIP-1: Framework & Meta
**Status:** Final | **Type:** Meta

Establishes the TAIP framework itself. Defines what a TAIP is, TAIP header format (front matter fields: taip, title, author, discussions-to, status, type, created, updated, requires, replaces, superseded-by), and editorial workflow.

**Key design principles introduced:**
- Robustness Principle: be conservative in what you send, liberal in what you accept
- Chain-agnostic, message-based, peer-to-peer messaging
- Support self-hosted wallets under direct party control
- No centralized gateways — use DIDs for all parties
- Any agent can initiate transactions
- No strict message flow (flexibility by design)
- Many-to-many relationship between TAP transactions and blockchain transactions
- PII must be end-to-end encrypted via DIDComm
- Built on open standards (JSON-LD, CAIP, W3C DIDs, JWS, JWE, DIDComm v2)
- DID types: `did:pkh` for wallets/smart contracts, `did:web` for centralized services

---

## TAIP-2: Messaging
**Status:** Last Call | **Requires:** TAIP-1

Defines the DIDComm v2 transport layer for all TAP messages.

**DIDComm v2 message structure:**
```json
{
  "id": "uuid",
  "type": "https://tap.rsvp/schema/1.0#Transfer",
  "from": "did:web:sender.example.com",
  "to": ["did:web:receiver.example.com"],
  "thid": "parent-thread-id",
  "pthid": "parent-of-parent-thread-id",
  "body": { /* TAP message body */ },
  "created_time": 1700000000,
  "expires_time": 1700003600
}
```

**Transport:** HTTPS with TLS 1.3+. Endpoint discovered from DID Document, with `serviceUrl` agent field as fallback.

**OOB (Out-Of-Band) discovery:** Agents can share connection invitations via URLs using `_oob=<base64url-encoded-message>` or `_oobid=<id>` query parameters. `goal_code: "tap.connect"` for TAIP-15 connections.

**Signing:** All messages must be signed with JWS. Supported algorithms: EdDSA (Ed25519) or ES256K (Secp256k1), referencing DID Document verification method.

**Encryption:** Optional JWE (ECDH-1PU or ECDH-ES + AES-256-GCM) for sensitive messages. Required for TAIP-8 presentations containing PII.

---

## TAIP-3: Asset Transfer
**Status:** Last Call | **Requires:** TAIP-2, TAIP-4, TAIP-5, TAIP-6

The foundational TAP message for initiating a blockchain asset transfer.

**Message type:** `https://tap.rsvp/schema/1.0#Transfer`

**Fields:**
| Field | Required | Type | Description |
|-------|----------|------|-------------|
| `asset` | Yes | CAIP-19 string | The asset being transferred |
| `amount` | Yes | Decimal string | Amount in base unit (e.g., "100.00") |
| `originator` | Yes | TAIP-6 Party | Sender party object |
| `beneficiary` | Yes | TAIP-6 Party | Receiver party object |
| `agents` | Yes | TAIP-5 Agent[] | Participating agents |
| `settlementId` | No | CAIP-220 | Blockchain tx ID if already settled |
| `memo` | No | String | Free-text note |
| `expiry` | No | ISO 8601 datetime | When the transfer request expires |

**Pattern:** The Transfer message's DIDComm `id` becomes the `thid` for all subsequent messages in the same transaction thread.

---

## TAIP-4: Authorization Flow
**Status:** Last Call | **Requires:** TAIP-2, TAIP-3

Defines the state machine messages for authorizing, settling, or rejecting transfers. All messages reference the Transfer via `thid`.

**Messages:**

**`Authorize`** — Agent approves the transfer, optionally providing settlement details:
- `settlementAddress` (CAIP-10, optional) — the address to send funds to
- `settlementAsset` (CAIP-19, optional) — if different from requested
- `amount` (decimal string, optional) — if different from requested
- `expiry` (ISO 8601, optional)

**`Settle`** — Confirms blockchain transaction was sent:
- `settlementAddress` (CAIP-10) — address funds were sent to
- `settlementId` (CAIP-220) — blockchain transaction ID
- `amount` (decimal string, optional)

**`Reject`** — Agent refuses the transfer:
- `reason` (string) — human-readable explanation

**`Cancel`** — Any party cancels the pending transfer:
- `by` (DID) — who is cancelling
- `reason` (string, optional)

**`Revert`** — Request reversal of an already-settled transaction:
- `settlementAddress` (CAIP-10) — address to return funds to
- `reason` (string)

**`AuthorizationRequired`** — Indicates out-of-band authorization is needed:
- `authorizationUrl` (URL) — where to complete authorization
- `expires` (ISO 8601)

---

## TAIP-5: Transaction Agents
**Status:** Last Call | **Requires:** TAIP-2

Defines agent objects that participate in a transaction and messages for managing agent membership.

**Agent object fields:**
| Field | Required | Description |
|-------|----------|-------------|
| `@id` | Yes | DID of the agent |
| `@type` | Yes | Agent type (VASP, Custodian, etc.) |
| `role` | No | Role in transaction (OriginatorVASP, BeneficiaryVASP, etc.) |
| `for` | No | DID of the party this agent acts for |
| `policies` | No | Array of TAIP-7 policy objects |
| `name` | No | Human-readable name |
| `url` | No | Website URL |
| `logo` | No | Logo image URL |
| `serviceUrl` | No | Fallback messaging endpoint URL |

**Endpoint resolution priority:** DID Document service endpoint → `serviceUrl` field

**Management messages:**
- `AddAgents` — Add new agents to the transaction; body: `{ agents: Agent[] }`
- `ReplaceAgent` — Replace an agent with another; body: `{ original: DID, replacement: Agent }`
- `RemoveAgent` — Remove an agent; body: `{ agent: DID }`

---

## TAIP-6: Transaction Parties
**Status:** Last Call | **Requires:** TAIP-2

Defines party objects (originator/beneficiary) and the message for updating party information.

**Party object fields:**
| Field | Required | Description |
|-------|----------|-------------|
| `@id` | Yes | DID of the party |
| `@type` | No | `Individual` or `Organization` |
| `name` | No | Full name |
| `url` | No | Website |
| `logo` | No | Logo URL |
| `description` | No | Description |
| `email` | No | Email address |
| `telephone` | No | Phone number |

**`UpdateParty` message fields:**
- `partyType` — `"originator"` or `"beneficiary"`
- `party` — The updated Party object

---

## TAIP-7: Agent Policies
**Status:** Last Call | **Requires:** TAIP-2, TAIP-5

Defines policy types that agents can declare in their `policies` array, and the `UpdatePolicies` message.

**Policy types:**

**`RequireAuthorization`** — Requires explicit authorization from a specific party:
- `from` (DID, optional) — specific DID that must authorize
- `fromRole` (string, optional) — role that must authorize
- `fromAgent` (DID, optional) — specific agent that must authorize

**`RequirePresentation`** — Requires a W3C Verifiable Presentation:
- `presentationDefinition` (URL) — link to the presentation definition

**`RequireRelationshipConfirmation`** — Requires proof of relationship (TAIP-9):
- `nonce` (string) — challenge value for the proof

**`RequirePurpose`** — Requires purpose/category codes (TAIP-13):
- `fields` (string array) — `["purpose"]`, `["categoryPurpose"]`, or both

**`UpdatePolicies` message:**
- `policies` (object) — map of agent DID → updated policies array
- All messages in the thread are linked via `thid`

---

## TAIP-8: Selective Disclosure
**Status:** Last Call | **Requires:** TAIP-2, TAIP-7

Enables privacy-preserving identity data exchange using W3C Verifiable Credentials and the WACI Present Proof protocol.

**Flow:**
1. Agent requests a presentation via `RequirePresentation` policy (TAIP-7) with a `presentationDefinition` URL
2. Requester agent fetches the presentation definition
3. Responding agent sends a DIDComm Present Proof message: `https://didcomm.org/present-proof/3.0/presentation`
4. The Present Proof message **must** be a DIDComm Encrypted Message (JWE) to protect PII

**Credential types:**
- `TravelRuleCredential` for IVMS-101 data (TAIP-10)
- Any W3C VC type per the presentation definition

---

## TAIP-9: Proof of Relationship
**Status:** Last Call | **Requires:** TAIP-2, TAIP-7, TAIP-8

Allows agents (especially self-hosted wallets) to prove they control an address and have a relationship with a party.

**`ConfirmRelationship` message:**
- Can include a CACAO (Chain-Agnostic CAPabilities Object, CAIP-74) as an attachment
- CACAO is a message signed by the blockchain wallet proving ownership and consent

**Certainty levels for relationship status:**
1. **Unconfirmed** — address provided but not verified
2. **Confirmed** — ConfirmRelationship message received (DID-level proof)
3. **Proven** — CACAO attachment with cryptographic wallet signature

---

## TAIP-10: IVMS101 / Travel Rule
**Status:** Last Call | **Requires:** TAIP-2, TAIP-7, TAIP-8, TAIP-9

Enables Travel Rule compliance by mapping IVMS-101 (InterVASP Messaging Standard) data exchange onto TAP's selective disclosure flow.

**No new message types** — uses TAIP-7 `RequirePresentation` + TAIP-8 `Present Proof`.

**Two flows:**
1. **Proactive:** Originator VASP sends IVMS-101 data alongside or after the Transfer (before beneficiary VASP replies)
2. **Reactive:** Beneficiary VASP requests IVMS-101 data via `RequirePresentation` in `UpdatePolicies`

**`TravelRuleCredential` VC type** includes:
- `originator` — IVMS-101 originator person object (name, address, national ID, etc.)
- `beneficiary` — IVMS-101 beneficiary person object

**Threshold guidance:** Data sharing requirements depend on jurisdiction and transaction amount (typically $1,000 USD / €1,000 EUR threshold).

---

## TAIP-11: Legal Entity Identifier
**Status:** Review | **Requires:** TAIP-2, TAIP-6

Adds LEI (Legal Entity Identifier) to party objects for institutional identification.

**`leiCode` field:** ISO 17442 20-character alphanumeric code added to party objects.

```json
{
  "@type": "https://schema.org/Organization",
  "@id": "did:web:bank.example.com",
  "name": "Example Bank",
  "leiCode": "5493001KJTIIGC8Y1R12"
}
```

**Rule:** Institutions that have an LEI MUST include it in their party object. Individual customers do not have LEIs.

**Planned extension:** vLEI (verifiable LEI) support for cryptographically verified LEIs.

---

## TAIP-12: Hashed Name
**Status:** Review | **Requires:** TAIP-2, TAIP-6

Enables privacy-preserving name verification without sharing the full name, for use cases like Travel Rule name screening without full PII disclosure.

**`nameHash` field:** SHA-256 hash added to party objects.

**Algorithm:**
1. Convert name to uppercase
2. Remove all whitespace
3. Compute SHA-256 hex digest

**Example:** "Alice Lee" → "ALICELEE" → `b117f44426c9670da91b563db728cd0bc8bafa7d1a6bb5e764d1aad2ca25032e`

**Compatibility:** Designed to be compatible with VerifyVASP and GTR name matching systems.

---

## TAIP-13: Purpose Codes
**Status:** Review | **Requires:** TAIP-2, TAIP-3

Adds ISO 20022 purpose codes to Transfer messages for payment categorization and regulatory reporting.

**Fields added to Transfer body:**
- `purpose` (string, optional) — ISO 20022 `ExternalPurpose1Code` (e.g., "SALA" for salary)
- `categoryPurpose` (string, optional) — ISO 20022 `ExternalCategoryPurpose1Code` (e.g., "SUPP" for supplier)

**Common purpose codes:**
| Code | Meaning |
|------|---------|
| SALA | Salary/payroll |
| TAXS | Tax payment |
| GDDS | Goods & merchandise |
| BEXP | Business expenses |
| CHAR | Charitable donation |
| SUBS | Subscription fee |
| SUPP | Supplier payment |
| DIVI | Dividend payment |

**`RequirePurpose` policy** (TAIP-7 extension):
```json
{
  "@type": "RequirePurpose",
  "fields": ["purpose", "categoryPurpose"]
}
```

---

## TAIP-14: Payments
**Status:** Review | **Requires:** TAIP-2, TAIP-3, TAIP-4, TAIP-5, TAIP-6

Defines a merchant-initiated payment request flow, as an alternative to TAIP-3's originator-initiated transfer.

**Message type:** `https://tap.rsvp/schema/1.0#Payment`

**Key difference from Transfer:** Uses `customer` and `merchant` roles instead of `originator`/`beneficiary`. The merchant creates the Payment and the customer completes it.

**Fields:**
| Field | Required | Description |
|-------|----------|-------------|
| `asset` | One of | CAIP-19 asset the merchant wants to receive |
| `currency` | One of | ISO 4217 currency (for fiat-denominated amounts) |
| `amount` | No | Requested amount |
| `supportedAssets` | No | Array of CAIP-19 strings OR pricing objects `{ asset, amount }` |
| `fallbackSettlementAddresses` | No | Map of CAIP-19 → CAIP-10 (default addresses if no Authorize) |
| `expiry` | No | ISO 8601 expiry for the payment request |
| `invoice` | No | TAIP-16 Invoice object |
| `policies` | No | Required policies for completion |
| `agents` | No | Participating agents |

**Flow:** Merchant creates Payment → Customer's wallet sends Authorize (with `settlementAddress`) → Merchant sends Settle after blockchain tx.

---

## TAIP-15: Agent Connection Protocol
**Status:** Review | **Requires:** TAIP-2, TAIP-3, TAIP-4, TAIP-5, TAIP-6

Establishes persistent, pre-authorized connections between agents with defined spending limits and constraints. Enables recurring billing, subscription payments, and pre-authorized transaction flows similar to OAuth scopes.

**Message type:** `https://tap.rsvp/schema/1.0#Connect`

**Fields:**
| Field | Required | Description |
|-------|----------|-------------|
| `requester` | Yes | TAIP-6 Party requesting the connection |
| `principal` | Yes | TAIP-6 Party being connected to |
| `agents` | No | TAIP-5 agents in the connection |
| `constraints` | No | Spending and permission limits |
| `agreement` | No | URL to legal agreement / terms |
| `expiry` | No | ISO 8601 expiry of the connection |
| `attachments` | No | DIDComm attachment objects (can include Transfer/Payment for immediate auth) |

**Constraints object:**
```json
{
  "purposes": ["SALA", "GDDS"],
  "categoryPurposes": ["SUPP"],
  "limits": {
    "per_transaction": "500.00",
    "day": "2000.00",
    "week": "5000.00",
    "month": "10000.00",
    "year": "100000.00",
    "currency": "USD"
  },
  "allowedBeneficiaries": ["did:pkh:eip155:1:0x..."],
  "allowedSettlementAddresses": ["eip155:1:0x..."],
  "allowedAssets": ["eip155:1/erc20:0xA0b8..."]
}
```

**Connection flow:**
1. Requester sends `Connect` message (OOB via `goal_code: "tap.connect"`)
2. Recipient responds with `AuthorizationRequired` (redirect to user consent UI) or directly `Authorize`
3. Connection is established; the `Connect` message's `id` becomes the connection `thid`
4. Subsequent transactions reference the connection via `pthid`

**Self-hosted wallet onboarding:** `requester` and `principal` can be the same DID for wallet self-registration.

---

## TAIP-16: Invoices
**Status:** Review | **Requires:** TAIP-14

Adds a structured invoice object to TAIP-14 Payment messages for detailed billing information.

**`invoice` object** (embedded in Payment body):

**Required fields:**
| Field | Description |
|-------|-------------|
| `id` | Unique invoice identifier |
| `issueDate` | ISO 8601 date |
| `currencyCode` | ISO 4217 3-letter code |
| `lineItems` | Array of line item objects |
| `total` | Total amount (decimal string) |

**Line item fields:** `id`, `description`, `quantity`, `unitPrice`, `lineTotal`

**Optional fields:** `taxTotal`, `subTotal`, `dueDate`, `note`, `paymentTerms`, `accountingCost`, `orderReference`, `additionalDocumentReference`

**Validation rule:** Invoice `total` MUST match Payment `amount`.

**Standards compatibility:**
- Maps to/from UBL (Universal Business Language) JSON format
- Maps to/from W3C Payment Request API

---

## TAIP-17: Composable Escrow
**Status:** Draft | **Requires:** TAIP-2, TAIP-3, TAIP-4, TAIP-5, TAIP-6

Adds escrow functionality to TAP transactions, enabling conditional or time-locked payments.

**New message types:**

**`Escrow`** — `https://tap.rsvp/schema/1.0#Escrow`
- Initiates an escrow arrangement
- `expiry` is **required** (escrow always has a deadline)
- Must have exactly one agent with `role: "EscrowAgent"`
- Fields: `asset` or `currency`, `amount`, `originator`, `beneficiary`, `expiry`, `agreement` (URL), `agents`

**`Capture`** — `https://tap.rsvp/schema/1.0#Capture`
- Releases escrowed funds to the beneficiary
- Sent by the beneficiary's agent (any agent with `for` = beneficiary DID)
- Fields: `amount` (optional, must be ≤ original escrow amount), `settlementAddress` (CAIP-10)

**State machine:**
```
Requested → (EscrowAgent accepts) → Active → (Capture sent) → Captured
                                           → (Cancel sent) → Cancelled
                                           → (expiry reached) → Expired
```

**Use cases:** Payment guarantees, cross-asset atomic swaps (combined with TAIP-18), delivery-on-payment, professional service milestones, platform escrow.

---

## TAIP-18: Asset Exchange
**Status:** Draft | **Requires:** TAIP-2, TAIP-3, TAIP-4, TAIP-17

Enables cross-asset exchange (cryptocurrency swaps, FX conversion, on/off ramps) within the TAP protocol.

**New message types:**

**`Exchange`** — `https://tap.rsvp/schema/1.0#Exchange`
- Initiates an exchange request; can be broadcast to multiple providers for best price
- Fields: `fromAssets` (array of CAIP-19/DTI/ISO-4217), `toAssets` (array), `fromAmount` OR `toAmount`, `requester` (TAIP-6 Party), `provider` (optional DID), `agents`, `policies`

**`Quote`** — `https://tap.rsvp/schema/1.0#Quote`
- Provider responds with a price quote
- Fields: `fromAsset`, `toAsset`, `fromAmount`, `toAmount`, `provider` (DID), `agents`, `expires`

**Flow using TAIP-4:**
1. Requester broadcasts `Exchange` to multiple providers
2. Providers respond with `Quote` messages
3. Requester accepts best quote via `Authorize` (with `settlementAddress`, `settlementAsset`, `amount`)
4. Provider settles via `Settle` (with `settlementId`)

**Combined with TAIP-17 escrow:**
- Reference the Escrow thread via `pthid` for atomic settlement guarantees

**Asset identifier types supported in Exchange:**
- CAIP-19 (crypto assets)
- DTI (Digital Token Identifier, ISO 24165)
- ISO 4217 (fiat currencies)

**Use cases:** USDC↔EURC stablecoin swaps, crypto on-ramps, crypto off-ramps, FX for cross-border payments, bridging between chains.

---

## TAIP-19: ISO 20022 Mapping
**Status:** Draft | **Requires:** TAIP-2, TAIP-3, TAIP-4, TAIP-14, TAIP-15

Defines bidirectional field mappings between ISO 20022 banking messages and TAP messages, enabling interoperability with traditional payment systems.

**No new message types** — all mappings use existing TAP messages.

**Core mapping principles:**
| ISO 20022 family | TAP equivalent | Purpose |
|------------------|----------------|---------|
| PACS (clearing/settlement) | TAIP-3 Transfer | Bank-to-bank backend settlement |
| PAIN (payment initiation) | TAIP-14 Payment or TAIP-15 Connect | Customer payment request |
| CAMT (cash management) | TAIP-4 Cancel / Revert | Status reporting and reversals |

**Key mappings:**
| ISO 20022 | TAP |
|-----------|-----|
| pacs.008 Credit Transfer | TAIP-3 Transfer |
| pain.001 Credit Transfer Initiation | TAIP-14 Payment |
| pain.009 Mandate Initiation | TAIP-15 Connect |
| camt.056 Payment Cancellation | TAIP-4 Revert/Cancel |

**Status code mapping:**
| ISO 20022 status | TAP equivalent |
|------------------|----------------|
| ACCP/ACTC | Authorize (accepted) |
| ACSP | Settle pending (tx broadcast) |
| ACSC | Settle (confirmed settlementId) |
| RJCT | Reject |
| CANC | Cancel |
| PDNG | AuthorizationRequired |

**payto:// URIs (RFC 8905)** for traditional accounts in TAP `settlementAddress` fields:
- IBAN: `payto://iban/DE89370400440532013000`
- ACH: `payto://ach/021000021/123456789`
- BIC/SWIFT: `payto://bic/DEUTDEDB`
- Sort code: `payto://sortcode/040004/01001234`
