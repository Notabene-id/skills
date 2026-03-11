# CAIP-19: Asset Type and Asset ID Specification

**Status:** Review
**Authors:** Antoine Herzog, Pedro Gomes, Joel Thorstensson
**Created:** 2020-06-23 | **Updated:** 2020-06-23
**Requires:** CAIP-2

> Always check for the latest version: `gh api repos/ChainAgnostic/CAIPs/contents/CAIPs/caip-19.md -H "Accept: application/vnd.github.raw"`

## Summary

CAIP-19 defines a way to identify a type of asset (e.g. Bitcoin, Ether, USDC) with an optional asset identifier suffix (for individually-addressable tokens like NFTs).

## Motivation

Exchanges, wallets, and dapps each maintain proprietary asset registries. CAIP-19 provides a universal, unambiguous identifier for any asset on any blockchain.

## Specification

### Asset Type

Identifies a class of assets (e.g., all USDC on Ethereum mainnet, or the CryptoKitties collection):

```
asset_type:        chain_id + "/" + asset_namespace + ":" + asset_reference
chain_id:          CAIP-2 identifier
asset_namespace:   [-a-z0-9]{3,8}
asset_reference:   [-.%a-zA-Z0-9]{1,128}
```

**Key:** The delimiter between chain_id and asset is `/` (slash), distinguishing it from the `:` used within chain_id and account_id.

### Asset ID

Identifies a specific token within a collection (e.g., CryptoKitty #771769):

```
asset_id:    asset_type + "/" + token_id
token_id:    [-.%a-zA-Z0-9]{1,78}
```

### Semantics

- **asset_namespace** — class of similar assets, usually an ecosystem standard (e.g., `slip44`, `erc20`, `erc721`, `spl`)
- **asset_reference** — specific asset within that class (e.g., contract address, coin type number)
- **token_id** — specific token within a collection (e.g., NFT serial number, `uint256` for ERC-721)

## Common Asset Namespace Reference

### Cross-chain: SLIP-44 (native tokens)

The `slip44` namespace uses registered coin type numbers from the [SLIP-44 registry](https://github.com/satoshilabs/slips/blob/master/slip-0044.md) to identify native tokens:

| Coin Type | Token | Example CAIP-19 |
|-----------|-------|-----------------|
| 0 | Bitcoin (BTC) | `bip122:000000000019d6689c085ae165831e93/slip44:0` |
| 2 | Litecoin (LTC) | `bip122:12a765e31ffd4059bada1e25190f6e98/slip44:2` |
| 60 | Ether (ETH) | `eip155:1/slip44:60` |
| 118 | Cosmos (ATOM) | `cosmos:cosmoshub-3/slip44:118` |
| 501 | Solana (SOL) | `solana:5eykt4UsFv8P8NJdTREpY1vzqKqZKvdp/slip44:501` |
| 714 | Binance (BNB) | `cosmos:Binance-Chain-Tigris/slip44:714` |

### EVM-specific: ERC standards

| Namespace | Standard | Reference Format | Example |
|-----------|----------|-----------------|---------|
| `erc20` | [ERC-20](https://eips.ethereum.org/EIPS/eip-20) | Contract address (0x + 40 hex) | `eip155:1/erc20:0xA0b8...eB48` |
| `erc721` | [ERC-721](https://eips.ethereum.org/EIPS/eip-721) | Contract address | `eip155:1/erc721:0x0601...266d` |
| `erc1155` | [ERC-1155](https://eips.ethereum.org/EIPS/eip-1155) | Contract address | `eip155:1/erc1155:0xd07d...` |

For ERC-721 and ERC-1155, append `/{token_id}` to identify a specific token.

### Solana: SPL tokens

| Namespace | Reference Format | Example |
|-----------|-----------------|---------|
| `spl` | Token mint address (base58) | `solana:5eykt.../spl:EPjFWdd5AufqSSqeM2qN1xzybapC8G4wEGGkZwyTDt1v` |

### Hedera: NFTs

| Namespace | Reference Format | Example |
|-----------|-----------------|---------|
| `nft` | Token entity ID (0.0.X) | `hedera:mainnet/nft:0.0.55492/12` |

## Canonicalization

Like CAIP-10, CAIP-19 does **not** require canonicalization of contract addresses. Implementers should handle deduplication. Check each namespace's CAIP-19 profile for specific canonicalization guidance.

## Test Cases

```
# Native tokens (SLIP-44)
eip155:1/slip44:60                                              # Ether
bip122:000000000019d6689c085ae165831e93/slip44:0                # Bitcoin
cosmos:cosmoshub-3/slip44:118                                   # ATOM
bip122:12a765e31ffd4059bada1e25190f6e98/slip44:2                # Litecoin
cosmos:Binance-Chain-Tigris/slip44:714                          # BNB

# ERC-20 tokens
eip155:1/erc20:0x6b175474e89094c44da98b954eedeac495271d0f      # DAI

# NFT collections
eip155:1/erc721:0x06012c8cf97BEaD5deAe237070F9587f8E7A266d     # CryptoKitties

# Specific NFTs
eip155:1/erc721:0x06012c8cf97BEaD5deAe237070F9587f8E7A266d/771769  # CryptoKitty #771769
hedera:mainnet/nft:0.0.55492/12                                     # Hedera NFT #12
```

## Changelog

- 2022-10-23: expanded charset to include `-`, `.`, `%`; added canonicalization section
- 2022-05-12: regex for token_id expanded to include entire `uint256` range
- 2021-06-25: regex max lengths raised
- 2020-06-23: added distinction between asset type and asset ID

## Links

- [Full specification](https://chainagnostic.org/CAIPs/caip-19)
- [CAIP-2](https://chainagnostic.org/CAIPs/caip-2) — Chain ID specification
- [CAIP-20](https://chainagnostic.org/CAIPs/caip-20) — SLIP-44 asset reference standard
- [SLIP-44 Registry](https://github.com/satoshilabs/slips/blob/master/slip-0044.md) — Coin type numbers
- [ERC-20](https://eips.ethereum.org/EIPS/eip-20) — Fungible token standard
- [ERC-721](https://eips.ethereum.org/EIPS/eip-721) — Non-fungible token standard
