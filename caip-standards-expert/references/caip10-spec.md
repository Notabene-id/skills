# CAIP-10: Account ID Specification

**Status:** Final
**Author:** Pedro Gomes
**Created:** 2020-03-13 | **Updated:** 2022-10-23
**Requires:** CAIP-2

> Always check for the latest version: `gh api repos/ChainAgnostic/CAIPs/contents/CAIPs/caip-10.md -H "Accept: application/vnd.github.raw"`

## Summary

CAIP-10 defines a way to identify an account on any blockchain specified by a CAIP-2 chain ID. It is used for both EOAs (externally owned accounts) and smart contract addresses.

## Motivation

Standardize account identifiers across all blockchains to enable interoperability between dapps and wallets. Used extensively in `did:pkh` (Decentralized Identifiers based on blockchain addresses).

## Specification

```
account_id:        chain_id + ":" + account_address
chain_id:          [-a-z0-9]{3,8}:[-_a-zA-Z0-9]{1,32}   (CAIP-2)
account_address:   [-.%a-zA-Z0-9]{1,128}
```

### Character Set

The `account_address` allows: `a-z`, `A-Z`, `0-9`, `-`, `.`, `%`

No other non-alphanumerics (`:`, `/`, `\`) are allowed. Use URL-encoding (`%` + 2 hex chars) for special characters per RFC 3986.

### Canonicalization

CAIP-10 does **not** require canonicalization. Some namespaces provide optional canonicalization:
- **EIP-55** — mixed-case checksum for Ethereum addresses
- **HIP-15** — optional checksum suffix for Hedera addresses

Implementations should handle deduplication when comparing addresses with/without canonicalization.

## Test Cases

```
# Ethereum mainnet (EIP-55 checksummed)
eip155:1:0xab16a96D359eC26a11e2C2b3d8f8B8942d5Bfcdb

# Bitcoin mainnet
bip122:000000000019d6689c085ae165831e93:128Lkh3S7CkDTBZ8W7BbpsN3YYizJMp8p6

# Cosmos Hub
cosmos:cosmoshub-3:cosmos1t2uflqwqe0fsj0shcfkrvpukewcw40yjj6hdc0

# Kusama (Polkadot)
polkadot:b0a8d493285c2df73290dfb7e61f870f:5hmuyxw9xdgbpptgypokw4thfyoe3ryenebr381z9iaegmfy

# StarkNet
starknet:SN_GOERLI:0x02dd1b492765c064eac4039e3841aa5f382773b598097a40073bd8b48170ab57

# Hedera (with HIP-15 checksum)
hedera:mainnet:0.0.1234567890-zbhlt

# Maximum length (106 chars)
chainstd:8c3444cf8970a9e41a706fab93e7a6c4:6d9b0b4b9994e8a6afbd3dc3ed983cd51c755afb27cd1dc7825ef59c134a39f7
```

## Backwards Compatibility

### Pre-2021 Format (Legacy)

Before August 2021, CAIP-10 used `address@chain_id` format:
```
# Legacy (DO NOT USE)
0xab16a96d359ec26a11e2c2b3d8f8b8942d5bfcdb@eip155:1

# Current
eip155:1:0xab16a96D359eC26a11e2C2b3d8f8B8942d5Bfcdb
```

### Pre-2022 Character Set

Before October 2022, only alphanumeric characters were allowed in `account_address`. The `-`, `.`, and `%` characters were added to support ecosystems like Hedera and others with non-alphanumeric address formats.

## Changelog

- 2022-10-23: expanded charset to include `-`, `.`, `%`; added canonicalization section
- 2022-03-10: updated regex to incorporate CAIP-2 reference
- 2021-08-11: switched from `address@chain_id` to `chain_id:address`

## Links

- [Full specification](https://chainagnostic.org/CAIPs/caip-10)
- [CAIP-2](https://chainagnostic.org/CAIPs/caip-2) — Chain ID specification
- [EIP-55](https://eips.ethereum.org/EIPS/eip-55) — Ethereum address checksum
- [HIP-15](https://github.com/hashgraph/hedera-improvement-proposal/blob/main/HIP/hip-15.md) — Hedera address checksum
- [DID PKH specification](https://github.com/w3c-ccg/did-pkh/) — Uses CAIP-10 as method-specific identifier
