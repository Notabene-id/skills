# CAIP-2: Blockchain ID Specification

**Status:** Final
**Authors:** Simon Warta, ligi, Pedro Gomes, Antoine Herzog
**Created:** 2019-12-05 | **Updated:** 2021-08-25

> Always check for the latest version: `gh api repos/ChainAgnostic/CAIPs/contents/CAIPs/caip-2.md -H "Accept: application/vnd.github.raw"`

## Summary

CAIP-2 defines a way to identify a blockchain (e.g. Ethereum Mainnet, Bitcoin, Cosmos Hub) in a human-readable, developer-friendly and transaction-friendly way.

## Motivation

Different ecosystems use different chain identifier systems (EIP-155 for Ethereum, genesis block hashes for Bitcoin, chain names for Cosmos). CAIP-2 provides a unified format that works across all of them.

## Specification

The blockchain ID ("chain ID") is a case-sensitive string:

```
chain_id:    namespace + ":" + reference
namespace:   [-a-z0-9]{3,8}
reference:   [-_a-zA-Z0-9]{1,32}
```

### Semantics

- **namespace** — identifies a class of similar blockchains (an ecosystem or standard). Examples: `eip155`, `bip122`, `cosmos`, `solana`, `polkadot`.
- **reference** — identifies a specific blockchain within that namespace. The resolution method is defined per namespace.

Each namespace has its own resolution method (e.g., `eth_chainId` RPC call for `eip155`, genesis block hash for `bip122`).

### Constraints

- Maximum length: 8 (namespace) + 1 (colon) + 32 (reference) = **41 characters**
- Case-sensitive
- URL-safe (no escaping needed in URL paths)
- Valid as filenames on case-sensitive UNIX filesystems

## Test Cases

```
# Ethereum mainnet
eip155:1

# Bitcoin mainnet (genesis block hash prefix)
bip122:000000000019d6689c085ae165831e93

# Litecoin
bip122:12a765e31ffd4059bada1e25190f6e98

# Cosmos Hub
cosmos:cosmoshub-2
cosmos:cosmoshub-3

# Binance chain (Cosmos SDK)
cosmos:Binance-Chain-Tigris

# StarkNet Testnet
starknet:SN_GOERLI

# Maximum length example (41 chars)
chainstd:8c3444cf8970a9e41a706fab93e7a6c4
```

## Common Namespace Reference Table

| Namespace | Ecosystem | Reference Format | How to Resolve |
|-----------|-----------|-----------------|----------------|
| `eip155` | Ethereum/EVM | Integer chain ID (decimal) | `eth_chainId` JSON-RPC |
| `bip122` | Bitcoin family | First 32 chars of genesis block hash (hex) | Genesis block hash lookup |
| `cosmos` | Cosmos/Tendermint | Chain name string | `status` RPC endpoint |
| `solana` | Solana | First 32 chars of genesis block hash (base58) | `getGenesisHash` RPC |
| `polkadot` | Polkadot/Substrate | First 32 chars of genesis block hash (hex, no 0x) | `chain_getBlockHash(0)` RPC |
| `tezos` | Tezos | Base58 network hash | Network hash from node |
| `starknet` | StarkNet | Network name string | Chain ID from network |
| `hedera` | Hedera | Network name (mainnet/testnet/previewnet) | Fixed values |
| `xrpl` | XRP Ledger | Network ID integer | Fixed values |
| `stellar` | Stellar | Network passphrase hash | SHA-256 of network passphrase |

## Links

- [Full specification](https://chainagnostic.org/CAIPs/caip-2)
- [EIP-155](https://eips.ethereum.org/EIPS/eip-155) — Ethereum chain ID standard
- [BIP-122](https://github.com/bitcoin/bips/blob/master/bip-0122.mediawiki) — Bitcoin chain definition
- [SLIP-44](https://github.com/satoshilabs/slips/blob/master/slip-0044.md) — Registered coin types
