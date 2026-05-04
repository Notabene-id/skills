# TAP TypeScript Libraries

Two official TypeScript packages for TAP:

| Package | What | Install |
|---------|------|---------|
| `@taprsvp/types` | Pure TypeScript types/interfaces for all TAP messages | `npm install @taprsvp/types` |
| `@taprsvp/agent` | Full WASM-backed TAP agent SDK (signing, encryption, DID resolution) | `npm install @taprsvp/agent` |

**Source repos:**
- Types: [TransactionAuthorizationProtocol/tap-ts](https://github.com/TransactionAuthorizationProtocol/tap-ts) (extracted from the TAIPs monorepo on 2026-05-01; full git history preserved via `git mv`. Use `git log --follow` in the new repo to trace any file back to its original commit. The npm package name `@taprsvp/types` is unchanged.)
- Agent: [TransactionAuthorizationProtocol/tap-rs/tap-ts](https://github.com/TransactionAuthorizationProtocol/tap-rs/tree/main/tap-ts)

---

## `@taprsvp/types` — Pure Types

Use this when you only need type definitions (e.g., building your own message handling on top of TAP types).

```typescript
import { Transfer, Agent, Party, Policy } from '@taprsvp/types';
```

All TAP message types, Party, Agent, Policy, Constraints, and Invoice interfaces are exported. See `references/message-types.md` and `references/agent-party-fields.md` for full field specs.

### Usage: Building a Transfer

```typescript
import { Transfer, Agent, Party } from '@taprsvp/types';

const originator: Party = {
  "@id": "did:pkh:eip155:1:0xSender...",
  "@type": "Individual",
  name: "Alice Smith"
};

const beneficiary: Party = {
  "@id": "did:pkh:eip155:1:0xReceiver...",
  "@type": "Individual"
};

const body: Transfer = {
  "@context": "https://tap.rsvp/schema/1.0",
  "@type": "https://tap.rsvp/schema/1.0#Transfer",
  asset: "eip155:1/erc20:0xA0b86991c6218b36c1d19D4a2e9Eb0cE3606eB48",
  amount: "1000.00",
  originator,
  beneficiary,
  agents: [
    {
      "@id": "did:web:originator-vasp.example.com",
      "@type": "VASP",
      role: "OriginatorVASP",
      for: originator["@id"],
      policies: [{ "@type": "RequireAuthorization", fromRole: "BeneficiaryVASP" }]
    },
    {
      "@id": "did:web:beneficiary-vasp.example.com",
      "@type": "VASP",
      role: "BeneficiaryVASP",
      for: beneficiary["@id"]
    }
  ]
};

// Wrap in DIDComm v2 envelope
const message = {
  id: crypto.randomUUID(),
  type: "https://tap.rsvp/schema/1.0#Transfer",
  from: "did:web:originator-vasp.example.com",
  to: ["did:web:beneficiary-vasp.example.com"],
  created_time: Math.floor(Date.now() / 1000),
  body
};
```

---

## `@taprsvp/agent` — Full TAP Agent SDK

Use this when you need to create agents, sign/encrypt messages, resolve DIDs, and handle the full DIDComm lifecycle. Built on WASM compiled from the Rust `tap-rs` crate — all cryptographic operations run inside the WASM boundary.

**Runtime dependency:** `@taprsvp/types` (re-exported)

### Architecture

```
TypeScript API (3.72KB gzipped)
    ↓
wasm-loader.ts (auto-detects browser vs Node.js)
    ↓
tap-wasm (272KB gzipped) — Rust → WASM
```

### Creating an Agent

```typescript
import { TapAgent } from '@taprsvp/agent';

// Generate new keys (Ed25519, P-256, or secp256k1)
const agent = await TapAgent.create({ keyType: 'Ed25519' });

// Or restore from existing key
const agent = await TapAgent.fromPrivateKey(hexPrivateKey, 'Ed25519');

console.log(agent.did);       // did:key:z6Mk...
console.log(agent.publicKey); // hex public key
```

**Supported key types:** `Ed25519`, `P-256`, `secp256k1`

### Packing and Unpacking Messages (DIDComm v2)

```typescript
// Sign a message (JWS — flattened format, Base64URL per RFC 7515)
const packed = await agent.pack(message);

// Unpack and verify a received message
const unpacked = await agent.unpack(envelope);
```

Encryption uses JWE with X25519 key agreement + AES-KW + Concat KDF (anoncrypt). Required for PII (TAIP-8 presentations).

### Message Creation Helpers

Factory functions that produce spec-compliant DIDComm messages with correct `@context`, `@type`, and envelope structure:

```typescript
import {
  createTransferMessage,
  createPaymentMessage,
  createAuthorizeMessage,
  createRejectMessage,
  createCancelMessage,
  createSettleMessage,
  createConnectMessage,
  createRFQMessage,    // formerly createExchangeMessage (renamed 2026-05-01)
  createQuoteMessage,
  createLockMessage,   // formerly createEscrowMessage (renamed 2026-05-01)
  createCaptureMessage,
  createBasicMessage,
  createDIDCommMessage,
} from '@taprsvp/agent';
```

> **Renames (2026-05-01):**
> - `Escrow` / `EscrowMessage` → `Lock` / `LockMessage` (validators and arbitraries renamed accordingly)
> - `Exchange` / `ExchangeMessage` → `RFQ` / `RFQMessage`
> - The `EscrowAgent` role name is preserved.

### DID Resolution

Built-in `did:key` resolver plus a pluggable interface for custom methods:

```typescript
const didDocument = await agent.resolveDID('did:key:z6Mk...');
```

### Utility Functions

```typescript
import {
  generatePrivateKey,
  generateUUID,
  isValidDID,
  isValidPrivateKey,
  validateKeyType,
  validateMessageStructure,
  validateTapMessageType,
  extractMessageTypeName,
  messageTypeToUri,
} from '@taprsvp/agent';
```

### Cleanup

Dispose of the WASM agent when done:

```typescript
agent.dispose();
```

---

## When to Use Which

| Scenario | Package |
|----------|---------|
| Type-checking TAP payloads in your existing DIDComm stack | `@taprsvp/types` |
| Building a TAP agent from scratch (signing, encryption, DID resolution) | `@taprsvp/agent` |
| Frontend code that only constructs message bodies (signing done server-side) | `@taprsvp/types` |
| Full end-to-end TAP messaging with key management | `@taprsvp/agent` |
| Veramo interop (JWS/JWE compatibility) | `@taprsvp/agent` |
