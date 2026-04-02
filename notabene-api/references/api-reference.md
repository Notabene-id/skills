# Notabene API Reference

**OpenAPI**: 3.1.0 | **Version**: 1.0.0 | **Base URL**: `https://api.eu1.notabene.id`

---

## Authentication

All endpoints require a Bearer JWT token obtained via OAuth2 client credentials:

```
POST https://auth.notabene.id/oauth/token
{
  "client_id": "<your_client_id>",
  "client_secret": "<your_client_secret>",
  "grant_type": "client_credentials",
  "audience": "https://api.eu1.notabene.id"
}
```

All requests must include: `Authorization: Bearer <token>`

---

## Table of Contents

- [Address Ownership](#address-ownership)
- [Entities](#entities)
- [Network](#network)
- [Relationships](#relationships)
- [Transfers](#transfers)
- [Transfer Checks](#transfer-checks)
- [PII (Personal Identifiable Information)](#pii)
- [Flow Customers](#flow-customers)
- [Flow Pay-ins](#flow-pay-ins)
- [Flow Payouts](#flow-payouts)
- [Flow Internal](#flow-internal)
- [Schemas](#schemas)

---

## Common Path Parameter

Almost every endpoint includes `{entityDID}` in the path:

| Name | Type | Description | Example |
|---|---|---|---|
| `entityDID` | `string (did)` | Your organization's DID | `did:web:vasps.id:jxnl0411:at` |

Pattern: `^did:([a-z0-9]+):((?:(?:[a-zA-Z0-9._-]|(?:%[0-9a-fA-F]{2}))*:)*(?:[a-zA-Z0-9._-]|(?:%[0-9a-fA-F]{2}))+)((;[a-zA-Z0-9_.:%-]+=[a-zA-Z0-9_.:%-]*)*)(/[^#?]*)?([?][^#]*)?(#.*)?$`

## Common Error Response

All error responses (400, 401, 403, 404, 409, 500) share this schema:

```json
{
  "error": "string (required)",
  "details": "string | string[]"
}
```

---

## Address Ownership

### `POST /entity/{entityDID}/address-ownership/discover`

Discover address ownership information for a blockchain address using relationships, hashed address service, and blockchain analytics.

**Request Body** (required, oneOf):

Option 1 - CAIP-10:
```json
{ "caip10": "eip155:1:0x742E4C6C1c6683A7E1Ba0BA3F8BC43AeDD5b5F4A" }
```

Option 2 - Asset + Address:
```json
{
  "asset": "eip155:1/slip44:60",
  "address": "0x742E4C6C1c6683A7E1Ba0BA3F8BC43AeDD5b5F4A"
}
```

**Response** `200`:

```json
{
  "addressOwnership": {
    "address": "eip155:1:0x742d35Cc...",
    "asset": "ETH",
    "agent": {
      "did": "did:web:entity1.example.com",
      "name": "Example Entity 1",
      "jurisdiction": "US"
    },
    "custodian": {
      "did": "did:web:fireblocks.com",
      "name": "Fireblocks",
      "jurisdiction": "US"
    },
    "relationship": {
      "status": "Confirmed",
      "confirmedAt": "2025-01-15T10:00:00.000Z",
      "confirmedBy": "did:web:example.com"
    },
    "confidence": "CONFIRMED",
    "source": "ADDRESS_BOOK",
    "discoveredAt": "2025-01-15T10:30:00.000Z",
    "responseTimeMs": 125
  },
  "meta": {
    "requestId": "550e8400-e29b-41d4-a716-446655440000",
    "timestamp": "2025-01-15T10:30:00.000Z"
  }
}
```

| Field | Type | Description |
|---|---|---|
| `confidence` | string | `CONFIRMED`, `UNCONFIRMED`, or `NOT_FOUND` |
| `source` | string | `ADDRESS_BOOK`, `HASHED_ADDRESS`, `INTEGRATION`, or `UNKNOWN` |

**Errors**: 400, 401, 403, 404 (returns `NOT_FOUND` confidence), 500, 501 (`NOT_IMPLEMENTED`)

---

## Entities

### `GET /entities/{entityDID}/public-keys`

Returns the encryption key used in webhooks for the entity. For V1 VASPs (did:ethr DIDs), returns Gateway VASP's keys instead. Key selection priority: keys ending with `#pii` > `#notabene-pii` > type `X25519KeyAgreementKey2019` > first available. Cached with 5-minute TTL.

**Response** `200`:

```json
{
  "entityDID": "did:web:entity2.example.com",
  "encryptionKey": {
    "id": "did:web:entity2.example.com#pii",
    "type": "EcdsaSecp256r1VerificationKey2019",
    "controller": "did:web:entity2.example.com",
    "publicKeyHex": "04a1b2c3d4e5f6...",
    "publicKeyBase58": "2Abc3Def4Ghi5Jkl..."
  },
  "gatewayDID": "did:web:gateway.notabene.studio"
}
```

`gatewayDID` is only present for V1 VASPs.

**Errors**: 404, 500, 503 (DID resolution failed)

---

## Network

### `GET /networks/{entityDID}`

Retrieve VASP/entity information including regulatory status, verification status, and protocol support.

**Query Parameters:**

| Name | Type | Description |
|---|---|---|
| `fields` | string | Comma-separated list of fields to include |
| `reviewedByVaspDID` | string | Filter by reviewing VASP |
| `showJurisdictionStatus` | string | Include jurisdiction status |
| `includeSubsidiaries` | string | Include subsidiary entities |

**Response** `200`: Returns a VASP entity object with fields including:

| Field | Type | Description |
|---|---|---|
| `did` | string | Entity DID |
| `name` | string? | Display name |
| `legalName` | string? | Legal name |
| `website` | string? | Website URL |
| `incorporationCountry` | string? | Country of incorporation |
| `isRegulated` | string? | Regulatory status |
| `regulatoryStatus` | string? | Detailed regulatory status |
| `verificationStatus` | string? | Verification status |
| `hasAdmin` | boolean? | Whether entity has an admin |
| `isNotabeneCustomer` | boolean? | Whether entity is a Notabene customer |
| `subsidiaries` | array? | Nested subsidiary entities |
| `badges` | string[]? | Entity badges |
| `isSandbox` | boolean? | Whether this is a sandbox entity |

Plus many more fields for address, travel rule protocol support, compliance phase, etc.

**Errors**: 400, 401, 403, 404, 409, 500

---

### `GET /network`

List entities in the network with filtering and pagination.

**Query Parameters:**

| Name | Type | Description |
|---|---|---|
| `q` | string | Search query |
| `emailDomain` | string | Filter by email domain |
| `badge` | string | `super_vasp`, `verified`, `in_network`, `claimed` |
| `jurisdictions` | string | Filter by jurisdictions |
| `page` | string | Page number |
| `per_page` | string | Results per page |
| `fields` | string | Comma-separated field list |
| `listingType` | string | `exclude_subsidiaries`, `all`, `exclude_gateways` |
| `listing` | string | `sandbox` or `non-sandbox` |

**Response** `200`:

```json
{
  "vasps": [ { ... } ],
  "pagination": { "page": 1, "per_page": 25, "total": 100 }
}
```

---

### `GET /networks/gleifSearch`

Search the GLEIF database for a Legal Entity Identifier (LEI).

**Query Parameters:**

| Name | Required | Type | Validation | Description |
|---|---|---|---|---|
| `leiNumber` | yes | string | Pattern: `^([A-Z0-9]){20}$` (exactly 20 alphanumeric chars) | LEI number to search |

**Response** `200`: GLEIF entity data including legal name, jurisdiction, and registration details.

---

## Relationships

### `POST /entities/{entityDID}/relationships`

Create a new relationship between an address and an entity. Optionally includes an ownership proof.

**Request Body** (required):

```json
{
  "from": "did:pkh:eip155:1:0x...",
  "to": "did:web:counterparty.com",
  "custodian": "did:web:custodian.com",
  "proof": { ... }
}
```

| Field | Required | Validation | Description |
|---|---|---|---|
| `to` | yes | 3–255 chars | DID of the target entity |
| `from` | no | 3–255 chars | DID of the source (address) |
| `custodian` | no | 3–255 chars, must match DID pattern | DID of the custodian |
| `proof` | no | | Ownership proof (see proof types below) |

**Proof types** (anyOf):

1. **Cryptographic signature**: `type` is one of `eip-191`, `eip-712`, `eip-1271`, `bip-137`, `bip-322`, `xpub`, `tip-191`, `ed25519`, `xrp-ed25519`, `xlm-ed25519`, `cip-8`, `siwe`, `siwx`. Requires `proof` and `attestation` fields.
2. **Screenshot**: `type: "screenshot"`, requires `url` (URI).
3. **Declaration**: `type: "declaration"`, requires `confirmed` (boolean).

**Response** `201`:

```json
{
  "relationship": {
    "@id": "uuid",
    "from": "did:pkh:...",
    "to": "did:web:...",
    "custodian": null,
    "status": "UNCONFIRMED",
    "createdTime": "...",
    "proofId": "..."
  }
}
```

Status values: `UNCONFIRMED`, `CONFIRMED`, `PROVEN`

---

### `GET /entities/{entityDID}/relationships`

List all relationships for an entity.

**Query Parameters:**

| Name | Type | Description |
|---|---|---|
| `from` | string (did) | Filter by source DID |
| `to` | string (did) | Filter by target DID |
| `custodian` | string (did) | Filter by custodian DID |

**Response** `200`:

```json
{
  "relationships": [
    {
      "@id": "uuid",
      "from": "did:pkh:eip155:1:0x...",
      "to": "did:web:...",
      "custodian": "did:web:...",
      "status": "CONFIRMED",
      "confirmedBy": "did:web:...",
      "proofs": [
        {
          "status": "VERIFIED",
          "did": "did:pkh:...",
          "address": "eip155:1:0x742d35Cc...",
          "type": "eip191",
          "proof": "0x...",
          "walletProvider": "MetaMask",
          "confirmed": true,
          "createdAt": "2025-01-15T10:00:00.000Z"
        }
      ]
    }
  ]
}
```

Proof status values: `PENDING`, `CONFIRMED`, `VERIFIED`, `REJECTED`
Proof types: `screenshot`, `eip191`, `eip712`, `declaration`

---

### `PATCH /entities/{entityDID}/relationships`

Confirm a relationship. Creates it if it doesn't exist (upsert). Optionally includes ownership proof.

**Query Parameters**: `from`, `to`, `custodian` (all optional, string DIDs)

**Request Body** (optional): Same proof structure as POST.

**Response** `201`: `{ "success": true }`

---

### `DELETE /entities/{entityDID}/relationships`

Delete an existing relationship.

**Query Parameters** (required): `from`, `to`

**Response** `202`: `{ "success": true }`

---

## Transfers

### `POST /entities/{entityDID}/tx`

Create a new transfer. This is the entry point for initiating Travel Rule compliance.

**Request Body** (required):

```json
{
  "ref": "a7135b55-0822-4e96-86dc-3c43dfc6c333",
  "originator": { "@id": "did:email:user@example.com" },
  "beneficiary": { "@id": "did:email:recipient@example.com" },
  "asset": "eip155:1/erc20:0xA0b86991c6218b36c1d19D4a2e9Eb0cE3606eB48",
  "amount": "100.00",
  "agents": [
    {
      "@id": "did:web:your-vasp.com",
      "role": "VASP",
      "for": "did:email:user@example.com",
      "policies": [
        { "@type": "REQUIRE_AUTHORIZATION" }
      ]
    },
    {
      "@id": "eip155:1:0xSettlementAddress",
      "role": "SettlementAddress"
    }
  ],
  "settlementId": "0xabc123...",
  "transactionValue": { "amount": "100.00", "currency": "USD" },
  "blockchainAnalyticsAlias": "My Wallet",
  "memoTag": "12345"
}
```

| Field | Required | Type | Validation | Description |
|---|---|---|---|---|
| `ref` | yes | string | 1–64 chars, pattern: `^[a-zA-Z0-9_-]{1,64}$` | Unique reference for idempotency |
| `originator` | yes | object | `@id`: 3–255 chars, must be a valid DID | `{ "@id": "did:..." }` |
| `beneficiary` | yes | object | `@id`: 3–255 chars, must be a valid DID | `{ "@id": "did:..." }` |
| `asset` | yes | string | CAIP-19 pattern or 2–20 char abbreviation (e.g. `BTC`) | Asset identifier |
| `amount` | yes | string | Pattern: `^\d+(\.\d+)?$` (numeric, no negatives) | Decimal amount |
| `agents` | yes | array | minItems: 1 | Array of agents (see Agent schema) |
| `settlementId` | no | string | maxLength: 255. CAIP-220 format recommended | On-chain transaction hash |
| `transactionValue` | no | object | `amount`: `^\d+(\.\d+)?$`; `currency`: 1–3 chars | `{ "amount": "...", "currency": "..." }` or null |
| `blockchainAnalyticsAlias` | no | string | | Alias for blockchain analytics |
| `memoTag` | no | string | 1–28 chars, pattern: `^[ -~]+$` (printable ASCII) | Memo/destination tag |

**Agent object:**

| Field | Required | Type | Description |
|---|---|---|---|
| `@id` | yes | string | Agent DID or address |
| `role` | no | string | `VASP`, `Custodian`, `SettlementAddress`, `SourceAddress`, `Gateway`, `Unknown` (default) |
| `for` | no | string | DID of party this agent acts for |
| `policies` | no | array | `[{ "@type": "REQUIRE_AUTHORIZATION" | "REQUIRE_PRESENTATION" | "REQUIRE_RELATIONSHIP_CONFIRMATION" }]` |
| `memoTag` | no | string | Memo/destination tag for address agents |

**Response** `201`: `{ "transfer": <Transfer object> }` (see [Transfer schema](#transfer))

**Errors**: 400, 401, 403, 404, 409 (duplicate ref), 500

---

### `GET /entities/{entityDID}/tx`

List transfers for an entity with filtering and pagination.

**Query Parameters:**

| Name | Type | Default | Description |
|---|---|---|---|
| `limit` | integer | 25 | Min: 1, Max: 100 |
| `offset` | integer | 0 | Min: 0 |
| `agent` | array | - | Filter by agent DIDs |
| `status` | array | - | Filter by transfer status |
| `asset` | array | - | Filter by asset identifier |
| `party` | array | - | Filter by party identifiers |
| `refs` | array | - | Filter by reference identifiers |
| `created_before` | string | - | ISO 8601 date-time |
| `created_after` | string | - | ISO 8601 date-time |
| `direction` | string | - | `INCOMING` or `OUTGOING` |
| `search_term` | string | - | Full-text search |
| `fields` | string | - | Comma-separated fields to include |
| `sort_by` | string | - | Field to sort by |
| `sort_order` | string | - | `asc` or `desc` |
| `action_required` | boolean | - | Filter to transfers requiring action |
| `request_for_information` | boolean | - | Filter to transfers with pending info requests |

**Response** `200`:

```json
{
  "data": [ { ... } ],
  "pagination": { "total": 100, "limit": 25, "offset": 0 }
}
```

---

### `GET /entities/{entityDID}/tx/{transferId}`

Get a specific transfer with optional PII decryption and timeline.

**Path Parameters:** `transferId` (uuid)

**Query Parameters:**

| Name | Type | Default | Description |
|---|---|---|---|
| `decrypt` | string | `"false"` | `"true"` or `"false"` - decrypt PII data |
| `sanitize` | string | `"true"` | `"true"` or `"false"` - sanitize sensitive data |
| `include_timeline` | string | `"false"` | Include event timeline |
| `include_integration_results` | string | `"false"` | Include integration results |
| `include_replaced` | boolean | `false` | Include replaced agents in the response |

**Response** `200`: `{ "transfer": <Transfer object> }`

---

### `GET /entities/{entityDID}/tx/search`

Search transfers with filters. Same query parameters as `GET /tx`. Returns `SearchTransfer` objects with `matchedColumns` field.

**Response** `200`:

```json
{
  "data": [
    {
      "id": "...",
      "ref": "...",
      "status": "...",
      "matchedColumns": ["ref", "settlementId"],
      ...
    }
  ],
  "pagination": { "total": 10, "limit": 25, "offset": 0 }
}
```

---

### `GET /entities/{entityDID}/tx/match`

Match transfers by settlement identifiers (e.g., on-chain transaction hash).

**Query Parameters:**

| Name | Required | Type | Description |
|---|---|---|---|
| `settlement_id` | yes | string | Settlement identifier (tx hash) |
| `settlement_address` | yes | string | Destination address |
| `settlement_id_index` | no | string | Vout/log index (strict match first, dropped if broadened) |
| `memo_tag` | no | string | Memo tag for matching |
| `limit` | no | integer | Default: 25, Max: 100 |
| `offset` | no | integer | Default: 0 |
| `direction` | no | string | `INCOMING` or `OUTGOING` |

**Response** `200`:

```json
{
  "data": [ { ... } ],
  "pagination": { "total": 1, "limit": 25, "offset": 0 },
  "meta": {
    "match_strategy": "strict",
    "broadened_candidates": 0
  }
}
```

`match_strategy`: `strict`, `broadened`, or `none`

---

### `POST /entities/{entityDID}/tx/{transferId}/authorize`

Authorize a transfer to proceed.

**Query Parameters:**

| Name | Type | Description |
|---|---|---|
| `force` | string | Force authorization even if checks fail |
| `ignoreFlags` | string | Comma-separated list of flags to ignore |

**Request Body** (required):

```json
{
  "settlementAddress": "eip155:1:0x...",
  "memoTag": "12345"
}
```

| Field | Required | Validation | Description |
|---|---|---|---|
| `settlementAddress` | no | Pattern: `^[a-z0-9-]+:[a-z0-9-]+:.+$` (CAIP-10) | Settlement address |
| `memoTag` | no | 1–28 chars, printable ASCII | Memo/destination tag |

**Response** `202`: `{ "message": "Transfer authorization accepted" }`

---

### `POST /entities/{entityDID}/tx/{transferId}/reject`

Reject a transfer with a reason.

**Request Body** (required):

```json
{
  "reason": "COUNTERPARTY_RISK",
  "comment": "Optional comment (required when reason is OTHER)"
}
```

| Field | Required | Validation | Description |
|---|---|---|---|
| `reason` | yes | One of the enum values below | Rejection reason |
| `comment` | conditional | maxLength: 500. **Required** when reason is `OTHER` | Additional details |

**Reason values**: `COUNTERPARTY_RISK`, `COUNTERPARTY_DUE_DILIGENCE`, `BLOCKCHAIN_RISK_SCORE`, `SANCTION_SCREENING`, `ASSET_TYPE`, `SUSPICIOUS_TRANSACTION`, `COUNTERPARTY_POLICIES`, `COUNTERPARTY_REJECTED`, `COUNTERPARTY_NO_RESPONSE`, `CANCELED_BY_INITIATOR`, `REMOVED_FROM_TRANSFER`, `TRANSFER_PARTICIPANT`, `SOURCE_ADDRESS`, `BENEFICIARY_ADDRESS`, `BENEFICIARY_NOT_FOUND`, `ORIGINATOR_REJECT_OUTGOING`, `BENEFICIARY_REJECT_INCOMING`, `COMPLIANCE_POLICIES`, `OTHER`

**Response** `200`: `{ "message": "Transfer rejected successfully" }`

---

### `POST /entities/{entityDID}/tx/{transferId}/settle`

Settle a transfer by providing on-chain transaction details.

**Request Body** (required):

```json
{
  "settlementId": "0xabc123...",
  "settlementIdIndex": "0",
  "revertSettlementId": "0xdef456..."
}
```

All fields are optional.

**Response** `202`: `{ "message": "Transfer settlement accepted" }`

---

### `POST /entities/{entityDID}/tx/{transferId}/agents`

Add an agent to an existing transfer.

**Request Body** (required):

```json
{
  "agent": {
    "@id": "did:web:new-agent.com",
    "role": "VASP",
    "for": "did:email:user@example.com",
    "policies": [{ "@type": "REQUIRE_AUTHORIZATION" }],
    "memoTag": "12345"
  }
}
```

**Response** `202`: `{ "message": "Agent addition accepted" }`

---

### `POST /entities/{entityDID}/tx/{transferId}/agents/replace`

Replace an existing agent with a new agent. Caller must be the transfer initiator, a gateway, or have a relationship with the agent being replaced.

**Request Body** (required):

```json
{
  "original": "did:web:old-agent.com",
  "replacement": {
    "@id": "did:web:new-agent.com",
    "role": "VASP",
    "for": "did:email:user@example.com"
  }
}
```

**Response** `202`: `{ "message": "Agent replacement accepted" }`

---

### TAP Policies

#### `GET /entities/{entityDID}/transfers/{transferId}/tap-policies`

List TAP policies for a transfer.

**Response** `200`:

```json
{
  "policies": [
    {
      "@id": "cb16e827-a11b-4b59-8fc0-9c36459b9447",
      "@type": "REQUIRE_PRESENTATION",
      "status": "PENDING",
      "from": "did:web:counterparty.com",
      "for": "did:web:your-vasp.com",
      "about": "originator",
      "presentationDefinition": "https://...",
      "encryptionKey": {
        "id": "did:web:counterparty.com#pii",
        "type": "EcdsaSecp256r1VerificationKey2019",
        "controller": "did:web:counterparty.com",
        "publicKeyHex": "04a1b2..."
      },
      "createdAt": "2025-12-01T12:24:52.537Z"
    }
  ]
}
```

#### `GET /entities/{entityDID}/transfers/{transferId}/tap-policies/{policyId}`

Get a specific TAP policy. **Response** `200`: `{ "policy": <TapPolicy> }`

**TapPolicy `@type` values**: `REQUIRE_AUTHORIZATION`, `REQUIRE_PRESENTATION`, `REQUIRE_RELATIONSHIP_CONFIRMATION`

**TapPolicy `status` values**: `COMPLETED`, `INCOMPLETE`, `PENDING`

---

## Transfer Checks

### `POST /entities/{entityDID}/tx/{transferId}/checks/beneficiary-name-matching`

Mark beneficiary name matching check as complete after verifying the beneficiary name matches your KYC records.

**Request Body**: None

**Response** `200`:

```json
{
  "id": "uuid",
  "transferId": "uuid",
  "entityDid": "did:web:...",
  "type": "REQUIRE_BENEFICIARY_NAME_MATCHING",
  "status": "COMPLETE",
  "policyData": {
    "confirmedBy": "did:web:...",
    "confirmedAt": "2025-01-15T10:00:00.000Z"
  },
  "createdAt": "...",
  "updatedAt": "..."
}
```

---

### `POST /entities/{entityDID}/tx/{transferId}/checks/internal-checks`

Mark internal checks as complete after completing operational checks on PII/Travel Rule information.

**Request Body**: None

**Response** `200`: Same schema as beneficiary-name-matching with `type: "REQUIRE_INTERNAL_CHECKS"`.

---

## PII

### `POST /entities/{entityDID}/transfers/{transferId}/append`

Append PII (IVMS101 data) to a transfer for Travel Rule compliance.

**Request Body** (required):

```json
{
  "originator": { "@id": "did:email:originator@notabene.id" },
  "beneficiary": { "@id": "did:email:beneficiary@notabene.id" },
  "ivms101": {
    "originator": {
      "originatorPerson": [{
        "naturalPerson": {
          "name": {
            "nameIdentifier": [{
              "primaryIdentifier": "Doe",
              "secondaryIdentifier": "John",
              "naturalPersonNameIdentifierType": "LEGL"
            }]
          },
          "dateAndPlaceOfBirth": {
            "dateOfBirth": "1980-01-01",
            "placeOfBirth": "Anytown, US"
          },
          "geographicAddress": [{
            "addressType": "HOME",
            "addressLine": ["123 Main Street"],
            "townName": "Anytown",
            "country": "US",
            "postCode": "1234"
          }]
        }
      }],
      "accountNumber": "originator-account"
    },
    "beneficiary": {
      "beneficiaryPerson": [{
        "naturalPerson": {
          "name": {
            "nameIdentifier": [{
              "primaryIdentifier": "Smith",
              "secondaryIdentifier": "Jane",
              "naturalPersonNameIdentifierType": "LEGL"
            }]
          }
        }
      }],
      "accountNumber": "beneficiary-account"
    }
  }
}
```

See [IVMS101 schema](#ivms101) for full structure.

**Response** `202`: `{ "message": "Transfer appended successfully with encrypted PII" }`

---

### `POST /entities/{entityDID}/transfers/{transferId}/policies/{policyId}/presentation`

Present PII data to counterparties for a specific policy.

**Path Parameters**: `transferId`, `policyId`

**Query Parameters**: `skipValidation` (boolean, default: false)

**Request Body** (required): `{ "ivms101": <IVMS101 object> }`

**Response** `202`: `{ "message": "Presentation accepted and processing" }`

---

## Flow Customers

### `POST /entities/{entityDID}/flow/customers`

Create a new customer for Flow workflows. PII is encrypted before storage.

**Request Body** (required):

```json
{
  "customerDid": "did:email:customer@example.com",
  "customerType": "natural_person",
  "profileData": {
    "naturalPerson": {
      "name": {
        "nameIdentifier": [{
          "primaryIdentifier": "Doe",
          "secondaryIdentifier": "John",
          "naturalPersonNameIdentifierType": "LEGL"
        }]
      }
    }
  },
  "verificationStatus": "pending",
  "verificationLevel": "basic"
}
```

| Field | Required | Type | Description |
|---|---|---|---|
| `customerDid` | yes | string | Pattern: `^did:.*` |
| `customerType` | yes | string | `natural_person` or `legal_person` |
| `profileData` | yes | object | IVMS101-compliant profile (see validation below) |
| `verificationStatus` | no | string | `pending`, `verified`, `rejected`, `expired` (default: `pending`) |
| `verificationLevel` | no | string | `basic`, `enhanced`, `premium` (default: `basic`) |

**IVMS101 Validation (strict — unknown fields are rejected):**
- **Natural persons:** `profileData.naturalPerson.name` is **required**
- **Legal persons:** `profileData.legalPerson.name` **and** `profileData.legalPerson.nationalIdentification` are both **required**
- When `nameIdentifier` array is provided, `primaryIdentifier` and `naturalPersonNameIdentifierType` are **required** within each entry
- The API rejects any unrecognized fields in `profileData` — do not include extra properties

**Response** `200`: Customer object with `customerDid`, `entityDid`, `customerType`, `profileData` (encrypted), `verificationStatus`, `verificationLevel`, `createdAt`, `updatedAt`.

**Errors**: 409 (customer already exists)

---

### `GET /entities/{entityDID}/flow/customers`

List customers with pagination.

**Query Parameters**: `limit` (default: `"25"`), `offset` (default: `"0"`)

**Response** `200`:

```json
{
  "data": [{ ... }],
  "pagination": { "limit": 25, "offset": 0, "total": 100 }
}
```

---

### `GET /entities/{entityDID}/flow/customers/{customerDID}`

Get a customer profile. Use `?includePII=true` to decrypt PII data.

**Query Parameters**: `includePII` (`"true"` or `"false"`, default: `"false"`)

---

### `PUT /entities/{entityDID}/flow/customers/{customerDID}`

Update a customer profile (partial update). No required fields in body.

**Response** `204`: No body.

---

### `DELETE /entities/{entityDID}/flow/customers/{customerDID}`

Permanently delete a customer profile. Cannot be undone.

**Response** `204`: No body.

---

## Flow Pay-ins

### `POST /entities/{entityDID}/flow/customers/{customerDID}/payins`

Create a pay-in for a specific customer (merchant).

**Request Body** (required):

```json
{
  "ref": "INV-2024-001",
  "currency": "USD",
  "amount": "1000.00",
  "customer": {
    "@id": "did:email:payer@example.com",
    "name": "John Payer",
    "email": "payer@example.com"
  },
  "fallbackSettlementAddresses": [
    "eip155:1:0xYourAddress..."
  ],
  "supportedAssets": [
    "eip155:1/erc20:0xA0b86991c6218b36c1d19D4a2e9Eb0cE3606eB48"
  ],
  "memo": "Payment for services",
  "invoice": {
    "id": "INV-2024-001",
    "dueDate": "2025-02-15",
    "lineItems": [
      { "description": "Consulting", "quantity": 1, "unitPrice": 1000, "lineTotal": 1000 }
    ],
    "paymentTerms": "Net 30"
  }
}
```

| Field | Required | Description |
|---|---|---|
| `ref` | yes | Client-provided reference (e.g., invoice number) |
| `currency` | yes | Currency code (USD, EUR, etc.) |
| `amount` | yes | Transaction amount |
| `customer` | yes | Payer/originator party (`@id` required) |
| `fallbackSettlementAddresses` | yes | CAIP-10 or payto:// addresses |
| `supportedAssets` | no | CAIP-19 asset identifiers |
| `memo` | no | Transaction memo |
| `invoice` | no | Invoice details |

**Response** `201`:

```json
{
  "transfer": {
    "@id": "uuid",
    "@type": "Payin",
    "ref": "INV-2024-001",
    "amount": "1000.00",
    "status": "...",
    "flowState": "...",
    "paymentLink": "https://connect.notabene.id/payin/<token>",
    "customerDid": "did:email:payer@example.com",
    "merchantDid": "did:email:merchant@example.com",
    "createdAt": "2025-01-15T10:00:00.000Z"
  }
}
```

**Errors**: 409 (duplicate ref - returns `existingRef`)

---

### `POST /entities/{entityDID}/flow/payins`

Create a pay-in at entity level (not scoped to a customer).

**Request Body**: Same as customer-level, plus required `merchant` object:

```json
{
  "merchant": { "@id": "did:web:merchant.com", "name": "Merchant", "email": "merchant@example.com" },
  "customer": { "@id": "did:email:payer@example.com" },
  "supportedAssets": ["..."],
  "fallbackSettlementAddresses": ["..."],
  ...
}
```

`supportedAssets` and `fallbackSettlementAddresses` are both **required** at entity level.

---

### `GET /entities/{entityDID}/flow/payins`

List all pay-ins for the entity.

**Query Parameters:**

| Name | Type | Default | Description |
|---|---|---|---|
| `refs` | string/array | - | Filter by client references |
| `limit` | string | `"50"` | Max results |
| `offset` | string | `"0"` | Pagination offset |
| `status` | string/array | - | Filter by Flow workflow states |
| `is_initiator` | boolean | - | `true` = outgoing, `false` = incoming |

---

### `GET /entities/{entityDID}/flow/payins/incoming`

List incoming pay-ins (where you are the responder/PRA).

### `GET /entities/{entityDID}/flow/payins/outgoing`

List outgoing pay-ins (where you are the initiator/PIA).

Both accept: `refs`, `limit`, `offset`, `status` query parameters.

---

### `GET /entities/{entityDID}/flow/payins/{payinId}`

Get detailed pay-in information including agents, amounts, log, and policy evaluations.

### `GET /entities/{entityDID}/flow/customers/{customerDID}/payins`

List pay-ins for a specific customer. Same query parameters as entity-level list.

### `GET /entities/{entityDID}/flow/customers/{customerDID}/payins/{payinId}`

Get a specific customer pay-in. Verifies the pay-in belongs to the customer.

---

### `POST /entities/{entityDID}/flow/payins/{payinId}/settlement_address`

Authorize a pay-in by providing a settlement address.

**Request Body** (required):

```json
{
  "asset": "eip155:1/erc20:0xA0b86991c6218b36c1d19D4a2e9Eb0cE3606eB48",
  "settlementAddress": "eip155:1:0xYourAddress..."
}
```

| Field | Required | Description |
|---|---|---|
| `settlementAddress` | yes | CAIP-10 or payto:// address |
| `asset` | no | CAIP-19 asset identifier |

**Response** `200`: `{ "message": "...", "transferId": "...", "asset": "...", "settlementAddress": "..." }`

---

### `POST /entities/{entityDID}/flow/payins/{payinId}/authorization_required`

Notify that the customer must visit a web URL to authorize the pay-in (used by PRAs).

**Request Body** (required):

```json
{ "authorizationUrl": "https://your-app.com/authorize/payment123" }
```

**Response** `200`: `{ "message": "...", "transferId": "..." }`

---

## Flow Payouts

### `POST /entities/{entityDID}/flow/customers/{customerDID}/payouts`

Create a payout for a customer.

**Request Body** (required):

```json
{
  "ref": "PAYOUT-001",
  "currency": "USD",
  "amount": "500.00",
  "merchant": {
    "@id": "did:web:merchant.com",
    "name": "Merchant Corp"
  },
  "supportedAssets": ["eip155:1/erc20:0xA0b86991c6218b36c1d19D4a2e9Eb0cE3606eB48"],
  "memo": "Refund"
}
```

| Field | Required | Description |
|---|---|---|
| `ref` | yes | Client-provided reference |
| `currency` | yes | Currency code |
| `amount` | yes | Transaction amount |
| `merchant` | yes | Merchant party (`@id` required) |

**Response** `201`: Transfer object with `@type: "Payout"`, `paymentLink`, etc.

---

### `POST /entities/{entityDID}/flow/payouts`

Create a payout at entity level. Both `customer` and `merchant` objects are **required**.

---

### `GET /entities/{entityDID}/flow/payouts`

List payouts. Query params: `refs`, `limit`, `offset`, `status`, `is_initiator`.

### `GET /entities/{entityDID}/flow/payouts/incoming`

List incoming payouts (as responder).

### `GET /entities/{entityDID}/flow/payouts/outgoing`

List outgoing payouts (as initiator).

### `GET /entities/{entityDID}/flow/payouts/{payoutId}`

Get detailed payout information.

### Customer-scoped payout endpoints

- `GET /entities/{entityDID}/flow/customers/{customerDID}/payouts` - List customer payouts
- `GET /entities/{entityDID}/flow/customers/{customerDID}/payouts/{payoutId}` - Get customer payout

---

### `POST /entities/{entityDID}/flow/payouts/{payoutId}/settlement_asset`

Authorize a payout by selecting asset and optional settlement address.

**Request Body** (required):

```json
{
  "asset": "eip155:1/erc20:0xA0b86991c6218b36c1d19D4a2e9Eb0cE3606eB48",
  "settlementAddress": "eip155:1:0x..."
}
```

| Field | Required | Description |
|---|---|---|
| `asset` | yes | CAIP-19 asset identifier |
| `settlementAddress` | no | CAIP-10 or payto:// address |

---

### `POST /entities/{entityDID}/flow/payouts/{payoutId}/authorization_required`

Notify that web authorization is required for a payout.

**Request Body** (required):

```json
{
  "authorizationUrl": "https://your-app.com/authorize/payout123",
  "expires": "2025-01-15T11:00:00.000Z"
}
```

| Field | Required | Description |
|---|---|---|
| `authorizationUrl` | yes | HTTPS URL for customer authorization |
| `expires` | no | ISO timestamp (defaults to 1 hour) |

---

## Flow Internal

### `POST /entities/{entityDID}/flow/{paymentId}/signal`

Send customer signals to Flow payment workflows (wallet selection, payment approval, etc.).

**Request Body** (required):

```json
{
  "action": "wallet_selected",
  "data": {
    "customerDid": "did:email:customer@example.com",
    "walletAddress": "eip155:1:0x...",
    "walletDid": "did:web:wallet-provider.com",
    "selectedAsset": "eip155:1/erc20:0xA0b86991c6218b36c1d19D4a2e9Eb0cE3606eB48",
    "settlementAddress": "eip155:1:0x..."
  }
}
```

**Action values**: `wallet_selected`, `payment_approved`, `payment_rejected`, `payment_viewed`, `fiat_settlement_indicated`

**Response** `200`: `{ "success": true, "paymentId": "...", "action": "...", "timestamp": "..." }`

---

### `GET /entities/{entityDID}/flow/{paymentId}/authorization`

Get authorization context for a Flow transfer. Returns 204 if authorization details aren't yet available.

**Response** `200`:

```json
{
  "authorizationUrl": "https://...",
  "from": "did:web:responder.com",
  "expires": "2025-01-15T11:00:00.000Z"
}
```

**Response** `204`: Authorization not yet available.

---

### `POST /entities/{entityDID}/flow/{paymentId}/agent`

Add an agent to a Flow payment workflow.

**Request Body** (required):

```json
{
  "agent": {
    "@id": "did:web:wallet-provider.com",
    "for": "did:email:customer@example.com"
  },
  "selectedAsset": "eip155:1/erc20:0xA0b86991c6218b36c1d19D4a2e9Eb0cE3606eB48"
}
```

**Response** `202`: Accepted.

---

### `GET /entities/{entityDID}/flow/{paymentId}/status`

Get current status of a Flow payment workflow.

**Response** `200`:

```json
{
  "transfer": {
    "@id": "...",
    "@type": "Payin",
    "status": "...",
    "asset": "...",
    "amount": "...",
    "requiresCustomerApproval": false,
    "agents": [...],
    "amounts": {...}
  }
}
```

---

### `POST /entities/{entityDID}/flow/{transferId}/fund`

Set funding address for a Flow transfer (used by onramp/IP agents).

**Request Body** (required):

```json
{
  "amount": "100.00",
  "currency": "USD",
  "for": "did:example:customer",
  "fundingAddress": "eip155:1:0x742d35Cc6634C0532925a3b844Bc454e4438f44e",
  "fundingAccount": "payto://iban/DE75512108001245126199",
  "fee": "1.5"
}
```

| Field | Required | Description |
|---|---|---|
| `amount` | yes | Amount to fund |
| `currency` | yes | Currency code |
| `for` | yes | DID of customer or merchant this funding is for |
| `fundingAddress` | no | CAIP-10 address for crypto |
| `fundingAccount` | no | PayTo URI for fiat (RFC 8905) |
| `fee` | no | Fee percentage |

**Response** `200`:

```json
{
  "success": true,
  "transfer": { ... },
  "fundingAddress": {
    "amount": "100.00",
    "currency": "USD",
    "fundingAddress": "eip155:1:0x...",
    "for": "did:example:customer",
    "addedBy": "did:web:onramp.com",
    "addedAt": "2025-01-15T10:00:00.000Z"
  }
}
```

---

## Schemas

### Transfer

The core transfer object returned by create, get, and list endpoints.

| Field | Type | Required | Description |
|---|---|---|---|
| `@id` | string | yes | Unique transfer identifier |
| `id` | string | no | Alternate ID |
| `status` | string | yes | See status values below |
| `asset` | string? | no | CAIP-19 asset identifier |
| `initiator` | string | yes | DID of the initiating entity |
| `agents` | array | yes | Array of transfer agents |
| `amount` | string? | no | Transfer amount |
| `originator` | object? | no | `{ "@id": "...", "originatorPerson": [...] }` |
| `beneficiary` | object? | no | `{ "@id": "...", "beneficiaryPerson": [...] }` |
| `ref` | string? | no | Client reference |
| `direction` | string? | no | `INCOMING` or `OUTGOING` |
| `transactionType` | string? | no | `TRANSFER`, `PAYOUT`, or `PAYIN` |
| `settlementAddress` | string? | no | Settlement blockchain address |
| `settlementId` | string? | no | On-chain transaction hash |
| `settlementIdIndex` | string? | no | Vout/log index |
| `isTravelRule` | boolean? | no | Whether travel rule applies |
| `memoTag` | string? | no | Memo/destination tag |
| `createdTime` | string? | no | Creation timestamp |
| `authorizedTime` | string? | no | When authorized |
| `rejectedTime` | string? | no | When rejected |
| `settledTime` | string? | no | When settled |
| `revertedTime` | string? | no | When reverted |
| `expiresTime` | string? | no | Expiration time |
| `amounts` | array? | no | Detailed amounts with currency conversions |
| `flags` | array? | no | Transfer flags |
| `log` | array? | no | Activity log |
| `policyEvaluations` | array? | no | Policy evaluation results |

**Transfer status values**: `OUTGOING`, `INCOMING`, `REJECTED`, `AUTHORIZED`, `FLAGGED`, `SETTLED`, `FLAGGED-SETTLEMENT`, `REVERTED`, `REVERT-AUTHORIZED`, `REVERT-REJECTED`, `REVERT-FLAGGED`, `FROZEN`, `CLEARED`, `REVERT-REQUESTED`

**Agent status values**: `PROCESSING`, `AUTHORIZED`, `REJECTED`, `SETTLED`, `FAILED`, `REPLACED`

**Agent role values**: `VASP`, `Custodian`, `SettlementAddress`, `SourceAddress`, `Gateway`, `Unknown`

---

### IVMS101

IVMS 101.2023 compliant data structure for originator and beneficiary PII.

```
ivms101:
  originator:
    originatorPerson[]:
      naturalPerson:
        name (REQUIRED):
          nameIdentifier[] (REQUIRED):
            primaryIdentifier (REQUIRED): string
            secondaryIdentifier: string
            naturalPersonNameIdentifierType (REQUIRED): string (e.g., "LEGL")
        geographicAddress[]:
          addressType (REQUIRED): string
          addressLine[]: string  (use this OR streetName+buildingNumber)
          streetName: string
          buildingNumber: string
          postCode: string
          townName (REQUIRED): string
          country (REQUIRED): string (ISO-3166 Alpha-2)
        nationalIdentification:
          nationalIdentifier (REQUIRED): string
          nationalIdentifierType (REQUIRED): string
          registrationAuthority: string
          countryOfIssue: string (ISO-3166 Alpha-2)
        customerIdentification: string
        dateAndPlaceOfBirth:
          dateOfBirth (REQUIRED): string (YYYY-MM-DD)
          placeOfBirth (REQUIRED): string
      legalPerson:
        name (REQUIRED):
          nameIdentifier[] (REQUIRED):
            legalPersonName (REQUIRED): string
            legalPersonNameIdentifierType (REQUIRED): string
        geographicAddress[]: (same structure)
        nationalIdentification:
          nationalIdentifier (REQUIRED): string
          nationalIdentifierType (REQUIRED): string
          registrationAuthority (REQUIRED for beneficiary): string
        customerIdentification: string
      accountNumber[]: string
  beneficiary:
    beneficiaryPerson[]: (same structure as originatorPerson)
```
