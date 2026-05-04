# Agent and Party Objects — Complete Field Reference

Full specifications for TAIP-5 Agent objects and TAIP-6 Party objects. These are the core identity building blocks of every TAP message.

---

## TAIP-6: Party Object

Parties are the **ultimate principals** of a transaction — the humans, businesses, or entities that own the assets being transferred. Parties are NOT responsible for routing or authorization mechanics; agents handle that.

```typescript
{
  "@id": string,         // Required. DID or IRI identifying the party
  "@type": string,       // Required. Party type (see values below)

  // Optional identity fields
  name?: string,         // Full legal name
  leiCode?: string,      // ISO 17442 LEI (TAIP-11) — 20-char alphanumeric
  nameHash?: string,     // SHA-256 of normalized name (TAIP-12)

  // Optional contact/profile
  email?: string,
  telephone?: string,
  description?: string,  // Free text
  url?: string,          // Website or profile URL
}
```

### Party `@type` Values

| `@type` | Description |
|---------|-------------|
| `Individual` | A natural person |
| `Organization` | A legal entity (company, institution) |
| `CorporateAction` | Used for corporate actions (e.g., dividends, splits) |

### Special Party Roles in Message Types

Some message types use different field names for parties:

| Message | Originator Role | Beneficiary Role |
|---------|----------------|-----------------|
| Transfer (TAIP-3) | `originator` | `beneficiary` |
| Payment (TAIP-14) | `customer` | `merchant` |
| Lock (TAIP-17) | `originator` (depositor) | `beneficiary` (recipient on capture) |
| Connect (TAIP-15) | `requester` | `principal` |
| RFQ (TAIP-18) | `requester` | `provider` (optional — TAIP-6 Party of liquidity provider; omit to broadcast) |
| UpdateParty (TAIP-6) | `partyType: "originator"` | `partyType: "beneficiary"` |

### TAIP-11: LEI Code

For organizations, include the ISO 17442 Legal Entity Identifier:

```json
{
  "@id": "did:web:exchange.example.com",
  "@type": "Organization",
  "name": "Acme Exchange Ltd",
  "leiCode": "5493001KJTIIGC8Y1R12"
}
```

The `leiCode` is 20 alphanumeric characters. Verify at https://www.gleif.org.

### TAIP-12: SHA-256 Name Hashing

For privacy-preserving name matching, use the `nameHash` field:

**Algorithm:**
1. Convert name to UPPERCASE
2. Strip all whitespace
3. SHA-256 hash → hex string

```
"Acme Corp" → "ACMECORP" → sha256 → "a7f3d2..."
```

```typescript
// TypeScript implementation
const nameHash = (name: string): string =>
  crypto.createHash('sha256')
    .update(name.toUpperCase().replace(/\s/g, ''))
    .digest('hex');
```

Use `nameHash` when you want to verify identity without transmitting the name in plaintext:

```json
{
  "@id": "did:pkh:eip155:1:0xAbc123...",
  "@type": "Individual",
  "nameHash": "a7f3d2c1b8e4f6..."
}
```

### Party Examples

**Individual party (wallet user):**
```json
{
  "@id": "did:pkh:eip155:1:0x742d35Cc6634C0532925a3b844Dc9BB",
  "@type": "Individual",
  "name": "Alice Smith"
}
```

**Organization (corporate customer):**
```json
{
  "@id": "did:web:treasury.acme.example.com",
  "@type": "Organization",
  "name": "Acme Corporation",
  "leiCode": "549300XKTVIUROZP4517",
  "url": "https://acme.example.com"
}
```

**Anonymous individual (with name hash only):**
```json
{
  "@id": "did:pkh:eip155:1:0xDeadBeef...",
  "@type": "Individual",
  "nameHash": "3d7a2f1c9b0e84a5..."
}
```

---

## TAIP-5: Agent Object

Agents are the **service providers and technical participants** in a TAP flow — VASPs, wallets, custodians, exchanges, and intermediaries. Every agent has a DID, a type, and typically a role in the specific transaction.

```typescript
{
  "@id": string,          // Required. DID of the agent (did:web or did:pkh)

  "@type": string,        // Required. Agent type (see values below)

  role?: string,          // Role in this transaction (see values below)

  for?: string,           // DID of the party this agent represents
                          // (links agent to the originator or beneficiary party)

  // Policies the OTHER agents must comply with to satisfy THIS agent
  policies?: Policy[],    // Array of TAIP-7 policy objects

  // Discovery fallback — use DID Document resolution first
  serviceUrl?: string,    // Fallback TAP endpoint URL (HTTPS required)

  // Optional display metadata
  name?: string,          // Display name ("Notabene", "Coinbase Custody")
  url?: string,           // Agent's public website
  logo?: string,          // URL to logo image
  description?: string,   // Free text description
  email?: string,         // Contact email
  telephone?: string,     // Contact phone

  // TAIP-11 for institutional agents
  leiCode?: string,       // ISO 17442 LEI code (20-char)
}
```

### Agent `@type` Values

| `@type` | Description |
|---------|-------------|
| `VASP` | Virtual Asset Service Provider |
| `CryptoWalletProvider` | Wallet software/service provider |
| `CryptoWallet` | A specific wallet instance (usually `did:pkh`) |
| `Custodian` | Custodial asset holder |
| `MessageRelay` | Message routing intermediary |
| `ExchangeAgent` | Asset exchange provider (TAIP-18) |
| `EscrowAgent` | Escrow service provider (TAIP-17) |
| `PaymentProcessor` | Payment processing intermediary |

### Agent `role` Values

Role describes this agent's function in **this specific transaction** (not the agent's general purpose):

| `role` | Description | Used In |
|--------|-------------|---------|
| `OriginatorVASP` | VASP serving the sender | Transfer |
| `BeneficiaryVASP` | VASP serving the receiver | Transfer |
| `CustodialWalletProvider` | Manages wallet on behalf of a party | Transfer |
| `NonCustodialWallet` | Self-custodied wallet (no intermediary) | Transfer |
| `IntermediaryVASP` | Routing or correspondent VASP | Transfer |
| `EscrowAgent` | Holds funds in escrow (exactly one required) | Lock (TAIP-17) |
| `ExchangeAgent` | Provides exchange quotes | RFQ / Quote (TAIP-18) |
| `PaymentProcessor` | Processes the payment | Payment |

### The `for` Field: Linking Agents to Parties

The `for` field identifies **which party this agent acts on behalf of**. This links the agent hierarchy to the party hierarchy:

```json
{
  "@id": "did:web:myvasp.example.com",
  "@type": "VASP",
  "role": "OriginatorVASP",
  "for": "did:pkh:eip155:1:0xSender..."   // ← Points to originator party's DID
}
```

Multiple agents can represent the same party (e.g., a VASP and a separate compliance agent both acting for the originator).

### Discovery: DID Document Resolution vs `serviceUrl`

Agents find each other's TAP endpoints via **DID Document resolution first**. The `serviceUrl` field in the agent object is a fallback only:

```
Priority order:
1. Resolve DID → find service endpoint with type "TAPMessaging"
2. Fall back to serviceUrl if DID resolution fails or has no TAP service
```

The DID Document should include:
```json
{
  "service": [{
    "id": "#tap-messaging",
    "type": "TAPMessaging",
    "serviceEndpoint": "https://tap.myvasp.example.com"
  }]
}
```

### Out-of-Band (OOB) Message Delivery

To initiate contact with an agent not yet in the thread, embed the DIDComm OOB invitation in a URL:

```
https://tap.myvasp.example.com?_oob=<base64url-encoded-message>
https://tap.myvasp.example.com?_oobid=<message-id>
```

### Agent Policies (TAIP-7)

Agents declare what they require from other agents using the `policies` array. These are the conditions that must be met before this agent will authorize:

```typescript
// Require explicit authorization signal from a specific role
{
  "@type": "RequireAuthorization",
  from?: string,       // DID of specific agent who must authorize
  fromRole?: string,   // Role that must authorize (e.g., "BeneficiaryVASP")
  fromAgent?: string,  // DID of specific agent (alias for `from`)
}

// Require a Verifiable Presentation (Travel Rule / KYC)
{
  "@type": "RequirePresentation",
  presentationDefinition: string,  // URL to PEX presentation definition
}

// Require proof that the beneficiary controls the settlement address
{
  "@type": "RequireRelationshipConfirmation",
  nonce: string,  // Challenge nonce for the CACAO proof
}

// Require payment purpose/category codes be included
{
  "@type": "RequirePurpose",
  fields: ("purpose" | "categoryPurpose")[],  // Which fields must be present
}
```

Policies are communicated via the `agents[].policies` array in any message or via a dedicated `UpdatePolicies` message.

### UpdatePolicies Message (TAIP-7)

Agents can update policies mid-flow:

```json
{
  "@context": "https://tap.rsvp/schema/1.0",
  "@type": "https://tap.rsvp/schema/1.0#UpdatePolicies",
  "policies": {
    "did:web:beneficiary-vasp.example.com": [
      {
        "@type": "RequirePresentation",
        "presentationDefinition": "https://beneficiary-vasp.example.com/kyc-pdef.json"
      }
    ]
  }
}
```

The key of the `policies` map is the DID of the agent **whose policies are being updated** (typically the sender's own DID — declaring their requirements).

### Agent Management Messages (TAIP-5)

**AddAgents** — Inject new agents into a thread (e.g., adding a compliance intermediary):
```json
{
  "@context": "https://tap.rsvp/schema/1.0",
  "@type": "https://tap.rsvp/schema/1.0#AddAgents",
  "agents": [
    {
      "@id": "did:web:compliance.example.com",
      "@type": "VASP",
      "role": "IntermediaryVASP",
      "for": "did:pkh:eip155:1:0xOriginator..."
    }
  ]
}
```

**ReplaceAgent** — Replace one agent with another (redirect, legal entity change):
```json
{
  "@context": "https://tap.rsvp/schema/1.0",
  "@type": "https://tap.rsvp/schema/1.0#ReplaceAgent",
  "original": "did:web:old-vasp.example.com",
  "replacement": {
    "@id": "did:web:new-vasp.example.com",
    "@type": "VASP",
    "role": "BeneficiaryVASP",
    "for": "did:pkh:eip155:1:0xBeneficiary..."
  }
}
```

**RemoveAgent** — Remove an agent from the thread:
```json
{
  "@context": "https://tap.rsvp/schema/1.0",
  "@type": "https://tap.rsvp/schema/1.0#RemoveAgent",
  "agent": "did:web:intermediary.example.com"
}
```

---

## Complete Example: Transfer with Full Agent/Party Structure

A real-world VASP-to-VASP transfer with all fields populated:

```json
{
  "id": "550e8400-e29b-41d4-a716-446655440000",
  "type": "https://tap.rsvp/schema/1.0#Transfer",
  "from": "did:web:originator-vasp.example.com",
  "to": ["did:web:beneficiary-vasp.example.com"],
  "created_time": 1700000000,
  "body": {
    "@context": "https://tap.rsvp/schema/1.0",
    "@type": "https://tap.rsvp/schema/1.0#Transfer",
    "asset": "eip155:1/erc20:0xA0b86991c6218b36c1d19D4a2e9Eb0cE3606eB48",
    "amount": "1000.00",
    "originator": {
      "@id": "did:pkh:eip155:1:0xAlice742d35...",
      "@type": "Individual",
      "name": "Alice Smith"
    },
    "beneficiary": {
      "@id": "did:pkh:eip155:1:0xBob8f3b21...",
      "@type": "Individual",
      "nameHash": "b3d9f2a1c4e7..."
    },
    "agents": [
      {
        "@id": "did:web:originator-vasp.example.com",
        "@type": "VASP",
        "role": "OriginatorVASP",
        "for": "did:pkh:eip155:1:0xAlice742d35...",
        "name": "Originator Exchange",
        "leiCode": "549300XKTVIUROZP4517",
        "policies": [
          {
            "@type": "RequireAuthorization",
            "fromRole": "BeneficiaryVASP"
          }
        ]
      },
      {
        "@id": "did:web:beneficiary-vasp.example.com",
        "@type": "VASP",
        "role": "BeneficiaryVASP",
        "for": "did:pkh:eip155:1:0xBob8f3b21...",
        "name": "Beneficiary Exchange",
        "serviceUrl": "https://tap.beneficiary-vasp.example.com"
      }
    ],
    "purpose": "SALA",
    "memo": "Invoice #INV-2024-001"
  }
}
```

---

## TypeScript Usage with @taprsvp/types

```typescript
import { Agent, Party, Policy, Transfer } from '@taprsvp/types';

const originator: Party = {
  "@id": "did:pkh:eip155:1:0xAlice...",
  "@type": "Individual",
  "name": "Alice Smith"
};

const beneficiary: Party = {
  "@id": "did:pkh:eip155:1:0xBob...",
  "@type": "Individual"
};

const originatorVASP: Agent = {
  "@id": "did:web:myvasp.example.com",
  "@type": "VASP",
  "role": "OriginatorVASP",
  "for": "did:pkh:eip155:1:0xAlice...",
  "name": "My VASP",
  "leiCode": "5493001KJTIIGC8Y1R12",
  "policies": [
    {
      "@type": "RequireAuthorization",
      "fromRole": "BeneficiaryVASP"
    }
  ]
};

const beneficiaryVASP: Agent = {
  "@id": "did:web:beneficiaryvasp.example.com",
  "@type": "VASP",
  "role": "BeneficiaryVASP",
  "for": "did:pkh:eip155:1:0xBob..."
};

const transfer: Transfer = {
  "@context": "https://tap.rsvp/schema/1.0",
  "@type": "https://tap.rsvp/schema/1.0#Transfer",
  asset: "eip155:1/erc20:0xA0b86991c6218b36c1d19D4a2e9Eb0cE3606eB48",
  amount: "1000.00",
  originator,
  beneficiary,
  agents: [originatorVASP, beneficiaryVASP]
};
```

---

## Common Patterns and Gotchas

**Self-hosted wallets (no VASP):**
When an originator controls their own wallet with no VASP, use a `did:pkh` agent with `@type: "CryptoWallet"` and `role: "NonCustodialWallet"`:
```json
{
  "@id": "did:pkh:eip155:1:0xAlice...",
  "@type": "CryptoWallet",
  "role": "NonCustodialWallet",
  "for": "did:pkh:eip155:1:0xAlice..."
}
```
Note: The agent `@id` and party `@id` will be the same DID in this case.

**Adding an intermediary mid-flow:**
Use `AddAgents` to inject a new routing agent. All agents already in the thread must update their routing accordingly.

**Agent vs Party `@id`:**
- Institutional agents use `did:web:institution.example.com` (resolvable service)
- Blockchain wallet agents use `did:pkh:eip155:1:0xAddress` (blockchain identity)
- Parties can use either, but `did:pkh` is typical for individuals

**`for` field is required for role clarity:**
Without `for`, it's ambiguous which party an agent represents. Always include it for OriginatorVASP and BeneficiaryVASP agents.

**Lock (TAIP-17) requires exactly one `EscrowAgent`:**
The Lock message spec (TAIP-17, formerly named `Escrow`) mandates exactly one agent in the `agents` array with `role: "EscrowAgent"`. This agent controls the escrow lifecycle.
