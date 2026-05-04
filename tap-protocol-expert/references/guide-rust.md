# TAP Rust Implementation (`tap-rs`)

**Repository:** [TransactionAuthorizationProtocol/tap-rs](https://github.com/TransactionAuthorizationProtocol/tap-rs)

A full Rust implementation of the Transaction Authorization Protocol. Provides typed structs for all TAP messages, DIDComm v2 message handling, cryptographic signing/encryption, and DID resolution. Also compiles to WASM for use in the `@taprsvp/agent` TypeScript package.

---

## Crate Structure

The repo is a Cargo workspace. Key crates:

| Crate | Purpose |
|-------|---------|
| `tap-types` | Core TAP message types and serialization |
| `tap-agent` | Agent implementation with key management, DIDComm packing/unpacking |
| `tap-wasm` | WASM bindings (powers `@taprsvp/agent` in TypeScript) |

### Install

```bash
cargo add tap-types    # types only
cargo add tap-agent    # full agent functionality
```

---

## Core Types

All message structs implement serde `Serialize`/`Deserialize` and use JSON-LD conventions (`@context`, `@type`).

### Party (TAIP-6)

```rust
struct Party {
    id: String,              // "@id" — DID
    party_type: String,      // "@type" — "Individual" | "Organization"
    name: Option<String>,
    lei_code: Option<String>,    // ISO 17442 LEI (TAIP-11)
    name_hash: Option<String>,   // SHA-256 (TAIP-12)
    email: Option<String>,
    telephone: Option<String>,
    url: Option<String>,
    description: Option<String>,
}
```

### Agent (TAIP-5)

```rust
struct Agent {
    id: String,                  // "@id" — DID
    agent_type: String,          // "@type" — VASP, Custodian, etc.
    role: Option<String>,        // OriginatorVASP, BeneficiaryVASP, etc.
    for_party: Option<String>,   // "for" — DID of represented party
    policies: Option<Vec<Policy>>,
    service_url: Option<String>,
    name: Option<String>,
    url: Option<String>,
    lei_code: Option<String>,
}
```

### Transfer (TAIP-3)

```rust
struct Transfer {
    context: String,         // "@context" = "https://tap.rsvp/schema/1.0"
    transfer_type: String,   // "@type" = "https://tap.rsvp/schema/1.0#Transfer"
    asset: String,           // CAIP-19
    amount: String,          // decimal string
    originator: Party,
    beneficiary: Party,
    agents: Vec<Agent>,
    settlement_id: Option<String>,  // CAIP-220
    memo: Option<String>,
    expiry: Option<String>,         // ISO 8601
    purpose: Option<String>,        // ISO 20022 (TAIP-13)
    category_purpose: Option<String>,
}
```

### Authorization Messages (TAIP-4)

```rust
struct Authorize {
    settlement_address: Option<String>,  // CAIP-10
    settlement_asset: Option<String>,    // CAIP-19
    amount: Option<String>,
    expiry: Option<String>,
}

struct Settle {
    settlement_address: String,  // CAIP-10
    settlement_id: String,       // CAIP-220
    amount: Option<String>,
}

struct Reject { reason: String }
struct Cancel { by: String, reason: Option<String> }
struct Revert { settlement_address: String, reason: Option<String> }
struct AuthorizationRequired { authorization_url: String, expires: String }
```

### Additional Message Types

All message types from TAIP-5 through TAIP-18 are implemented:

- **Agent management:** `AddAgents`, `ReplaceAgent`, `RemoveAgent` (TAIP-5)
- **`UpdateParty`** (TAIP-6), **`UpdatePolicies`** (TAIP-7)
- **`ConfirmRelationship`** (TAIP-9)
- **`Payment`** with `Invoice` (TAIP-14/16)
- **`Connect`** with `Constraints` (TAIP-15)
- **`Lock`**, **`Capture`** (TAIP-17 — `Lock` was renamed from `Escrow` on 2026-05-01)
- **`RFQ`**, **`Quote`** (TAIP-18 — `RFQ` was renamed from `Exchange` on 2026-05-01)

---

## DIDComm v2 Envelope

All TAP messages are wrapped in DIDComm v2 messages:

```rust
struct DIDCommMessage {
    id: String,
    msg_type: String,           // "type" — TAP message URI
    from: String,               // sender DID
    to: Vec<String>,            // recipient DIDs
    thid: Option<String>,       // thread ID
    pthid: Option<String>,      // parent thread ID
    created_time: Option<u64>,  // unix timestamp
    expires_time: Option<u64>,
    body: serde_json::Value,    // TAP message body
}
```

---

## Signing and Encryption

- **Signing:** JWS with EdDSA (Ed25519) or ES256K (secp256k1)
- **Encryption:** JWE with ECDH-1PU or ECDH-ES + AES-256-GCM (required for PII in TAIP-8)

---

## WASM Compilation

The `tap-wasm` crate compiles the full Rust implementation to WebAssembly, producing the WASM binary used by `@taprsvp/agent`. This ensures cryptographic operations and message validation are identical across Rust and TypeScript.

Build:
```bash
wasm-pack build tap-wasm --target web
```

---

## Usage Example

```rust
use tap_types::{Transfer, Party, Agent};

let transfer = Transfer {
    context: "https://tap.rsvp/schema/1.0".to_string(),
    transfer_type: "https://tap.rsvp/schema/1.0#Transfer".to_string(),
    asset: "eip155:1/erc20:0xA0b86991c6218b36c1d19D4a2e9Eb0cE3606eB48".to_string(),
    amount: "1000.00".to_string(),
    originator: Party {
        id: "did:pkh:eip155:1:0xSender...".to_string(),
        party_type: "Individual".to_string(),
        name: Some("Alice Smith".to_string()),
        ..Default::default()
    },
    beneficiary: Party {
        id: "did:pkh:eip155:1:0xReceiver...".to_string(),
        party_type: "Individual".to_string(),
        ..Default::default()
    },
    agents: vec![
        Agent {
            id: "did:web:originator-vasp.example.com".to_string(),
            agent_type: "VASP".to_string(),
            role: Some("OriginatorVASP".to_string()),
            for_party: Some("did:pkh:eip155:1:0xSender...".to_string()),
            ..Default::default()
        },
    ],
    ..Default::default()
};

let json = serde_json::to_string_pretty(&transfer)?;
```

> **Note:** Exact struct field names may differ from the examples above. Always check the source at [github.com/TransactionAuthorizationProtocol/tap-rs](https://github.com/TransactionAuthorizationProtocol/tap-rs) for the current API. The Rust crate uses `serde(rename)` attributes to map between Rust snake_case and JSON-LD `@`-prefixed field names.
