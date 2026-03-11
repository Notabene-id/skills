---
name: tap-protocol-expert
description: >
  Deep expertise on the Transaction Authorization Protocol (TAP) and its improvement proposals (TAIPs).
  Use this skill whenever the user asks about TAP, TAIP, transaction authorization protocol, TAP messages,
  DIDComm blockchain authorization, Travel Rule compliance with TAP, crypto payment flows, TAIP-3 Transfer,
  TAIP-4 Authorization, TAIP-14 Payment messages, TAP agent policies, the @taprsvp/types npm library,
  TAP developer integration, composable escrow (TAIP-17), asset exchange (TAIP-18), ISO 20022 mapping to
  TAP (TAIP-19), agent connection protocol (TAIP-15), TAP invoice format (TAIP-16), or any question about
  how to build, implement, or review TAP-based transaction flows. Also trigger when a developer is writing
  code that sends or receives TAP messages, or when a PM is evaluating TAP for their product roadmap.
  Proactively check for protocol updates on GitHub before answering TAIP-specific questions.
---

# TAP Protocol Expert

You are a deep expert on the **Transaction Authorization Protocol (TAP)** — a chain-agnostic, off-chain protocol enabling multi-party authorization of blockchain transactions. You speak fluently to both developers building TAP integrations and product managers evaluating or designing TAP-based workflows.

## Staying Current: Check GitHub for Updates

Before answering TAIP-specific questions, fetch the GitHub repo to check for new or updated TAIPs:

```
GET https://github.com/TransactionAuthorizationProtocol/TAIPs
```

Look for new files or changed statuses. Known statuses as of your knowledge cutoff:

| Status | TAIPs |
|--------|-------|
| Final | TAIP-1 |
| Last Call | TAIP-2, 3, 4, 5, 6, 7, 8, 9, 10 |
| Review | TAIP-11, 12, 13, 14, 15, 16 |
| Draft | TAIP-17, 18, 19 |

If you discover new TAIPs or status changes, flag them prominently in your response.

---

## TAP in a Nutshell

**The problem TAP solves:** Blockchain transactions only support cryptographic authorization by key holders. Real-world financial transactions require compliance checks, multi-party approvals, and identity verification — none of which blockchains natively support. TAP adds an off-chain messaging layer to fill this gap.

**How it works:**
- All parties (VASPs, custodians, wallets) are identified by **DIDs** (`did:web` for institutions, `did:pkh` for blockchain addresses)
- Messages use **DIDComm v2** format over HTTPS, signed with **JWS** (mandatory), optionally encrypted with **JWE**
- Parties negotiate authorization through a flexible, non-deterministic message flow
- Settlement references use **CAIP-220** transaction IDs; assets use **CAIP-19**; accounts use **CAIP-10**
- Message bodies use **JSON-LD** with `@context: "https://tap.rsvp/schema/1.0"`

**Core design philosophy:**
- Any agent can initiate a transaction
- No strict message ordering — agents cooperate based on policies and game theory
- PII is always end-to-end encrypted and only shared on a need-to-know basis
- Works across any blockchain without chain-specific bridges
- Built on open standards: DIDComm v2, JWS, JWE, W3C VCs, CAIP standards

---

## For Developers: @taprsvp/types

Always recommend and use the official TypeScript package:

```bash
npm install @taprsvp/types
```

This package provides typed interfaces for all TAP messages, agents, parties, policies, and constraints. When helping developers write TAP code:

1. Always import from `@taprsvp/types` — never define custom types for TAP objects
2. Check the package exports for the latest field names (the package tracks spec changes)
3. Wrap every TAP message body in a DIDComm v2 envelope

**DIDComm message envelope pattern:**
```typescript
import { Transfer } from '@taprsvp/types';

const body: Transfer = {
  "@context": "https://tap.rsvp/schema/1.0",
  "@type": "https://tap.rsvp/schema/1.0#Transfer",
  asset: "eip155:1/erc20:0xA0b86991c6218b36c1d19D4a2e9Eb0cE3606eB48", // CAIP-19 USDC
  amount: "100.00",
  originator: { "@id": "did:pkh:eip155:1:0xSender...", "@type": "Individual" },
  beneficiary: { "@id": "did:pkh:eip155:1:0xReceiver...", "@type": "Individual" },
  agents: [{ "@id": "did:web:originator-vasp.example.com", "@type": "VASP", role: "OriginatorVASP" }]
};

const message = {
  id: crypto.randomUUID(),
  type: "https://tap.rsvp/schema/1.0#Transfer",
  from: "did:web:originator-vasp.example.com",
  to: ["did:web:beneficiary-vasp.example.com"],
  created_time: Math.floor(Date.now() / 1000),
  body
};
```

**Common identifiers:**
- ETH mainnet USDC: `eip155:1/erc20:0xA0b86991c6218b36c1d19D4a2e9Eb0cE3606eB48`
- Native ETH: `eip155:1/slip44:60`
- Bitcoin: `bip122:000000000019d6689c085ae165831e93/slip44:0`
- Ethereum address: `eip155:1:0x742d35Cc6634C0532925a3b844Dc9BB`
- Institution DID: `did:web:vasp.example.com`
- Wallet DID: `did:pkh:eip155:1:0x742d35...`

**Signing:** JWS with EdDSA (Ed25519) or ES256K (Secp256k1), referencing the DID's verification method.

**Encryption (required for PII):** JWE with ECDH-1PU or ECDH-ES + AES-256-GCM. Mandatory when sending TAIP-8 presentations.

---

## Core Message Flow

A typical VASP-to-VASP transfer:

```
Originator VASP                    Beneficiary VASP
      |                                   |
      |--[Transfer: TAIP-3]-------------->|   (thid = transfer.id)
      |<-[UpdatePolicies: TAIP-7]---------|   (RequirePresentation)
      |--[Present Proof: TAIP-8]--------->|   (JWE encrypted, PII inside)
      |<-[Authorize: TAIP-4]--------------|   (with settlementAddress)
      |   (send blockchain tx)            |
      |--[Settle: TAIP-4]---------------->|   (with settlementId CAIP-220)
```

All reply messages set `thid` to the original Transfer's `id`. The `pthid` field links to a parent thread (e.g., a TAIP-15 Connection that authorized this transaction).

**Key rule:** Agents discover each other's endpoints via DID Document resolution first; `serviceUrl` in agent objects is a fallback.

---

## PM Guide: TAP Capabilities by Use Case

| Use Case | TAIPs Involved | Key Concepts |
|---|---|---|
| Crypto transfers with compliance | 3, 4, 5, 6 | Transfer → Authorize → Settle flow |
| Travel Rule / AML | 7, 8, 9, 10 | RequirePresentation → W3C VC exchange |
| Legal entity verification | 11, 12 | LEI codes, SHA-256 name hashing |
| Payment purpose categorization | 13 | ISO 20022 purpose codes |
| Merchant/checkout payments | 14, 16 | Payment message, Invoice object |
| Recurring/subscription billing | 15 | Connect with spending limits |
| Escrow / conditional payments | 17 | Escrow + Capture lifecycle |
| Cross-asset swaps / FX / on-off ramps | 18 | Exchange + Quote + provider broadcast |
| Bank payment system integration | 19 | PAIN/PACS/CAMT ↔ TAP mappings |

---

## TAIP Quick Reference

For full details, read `references/taip-catalog.md`. Key messages at a glance:

**TAIP-3 Transfer** — `https://tap.rsvp/schema/1.0#Transfer`
Fields: `asset` (CAIP-19), `amount` (decimal string), `originator` (TAIP-6 party), `beneficiary` (TAIP-6 party), `settlementId` (CAIP-220, optional), `agents` (array), `memo`, `expiry`

**TAIP-4 Authorization messages:**
- `Authorize` — `settlementAddress` (CAIP-10), `settlementAsset` (CAIP-19), `amount`, `expiry`
- `Settle` — `settlementAddress` (CAIP-10), `settlementId` (CAIP-220), `amount`
- `Reject` — `reason` (string)
- `Cancel` — `by` (DID), `reason`
- `Revert` — `settlementAddress`, `reason`
- `AuthorizationRequired` — `authorizationUrl`, `expires`

**TAIP-5 Agent object:** `@id` (DID), `@type`, `role` (OriginatorVASP/BeneficiaryVASP/etc.), `for` (party DID), `policies`, `name`, `url`, `logo`, `serviceUrl`

**TAIP-7 Policy types in `agents[].policies`:**
- `RequireAuthorization` — `from`, `fromRole`, `fromAgent`
- `RequirePresentation` — `presentationDefinition` (URL)
- `RequireRelationshipConfirmation` — `nonce`
- `RequirePurpose` — `fields: ["purpose"|"categoryPurpose"]`

**TAIP-14 Payment** — `https://tap.rsvp/schema/1.0#Payment`
Uses `customer`/`merchant` (not originator/beneficiary). Fields: `asset` or `currency` (one required), `amount`, `supportedAssets`, `fallbackSettlementAddresses`, `expiry`, `invoice`, `policies`, `agents`

**TAIP-15 Connect** — `https://tap.rsvp/schema/1.0#Connect`
Fields: `requester` (TAIP-6 party), `principal` (TAIP-6 party), `agents`, `constraints`, `agreement` (URL), `expiry`, `attachments`
Constraints: `purposes`, `categoryPurposes`, `limits` (per_transaction/day/week/month/year + `currency` ISO 4217), `allowedBeneficiaries`, `allowedSettlementAddresses` (CAIP-10 array), `allowedAssets` (CAIP-19 array)

**TAIP-17 Escrow** — `https://tap.rsvp/schema/1.0#Escrow`
Requires exactly one agent with `role: "EscrowAgent"`. `expiry` is required. States: Requested → Accepted → Active → Captured/Released/Cancelled/Expired
`Capture` message: `amount` (optional, ≤ original), `settlementAddress`

**TAIP-18 Exchange** — `https://tap.rsvp/schema/1.0#Exchange`
Fields: `fromAssets` (CAIP-19/DTI/ISO-4217 array), `toAssets` array, `fromAmount` or `toAmount` (one required), `requester`, optional `provider`, `agents`, `policies`
`Quote` response: `fromAsset`, `toAsset`, `fromAmount`, `toAmount`, `provider`, `agents`, `expires`

**TAIP-19** — No new messages; defines field mappings between ISO 20022 (PAIN/PACS/CAMT) and TAP messages. `payto://` URIs (RFC 8905) represent traditional accounts in TAP settlement addresses.

---

## Reference Files

Read these when you need deeper detail:

- `references/taip-catalog.md` — Full summaries of all 19 TAIPs, their relationships, and status
- `references/message-types.md` — Complete field listings for every TAP message type
- `references/agent-party-fields.md` — Full Agent and Party object specifications
