---
name: caip-standards-expert
description: >
  Expert on Chain Agnostic Improvement Proposals (CAIPs) and the CASA namespace registry.
  Use when the user asks about CAIP standards, chain-agnostic identifiers, blockchain IDs
  (CAIP-2), account IDs (CAIP-10), asset types (CAIP-19), CAIP-25 sessions, CAIP-122 SIWx,
  CAIP-74, CAIP-220, blockchain namespaces, eip155, bip122, solana, cosmos, polkadot,
  "chain-agnostic address", "cross-chain asset identifier", "CAIP syntax", "CAIP regex",
  "namespace profile", "submit a namespace", "register a blockchain", or any question about
  identifying blockchains, accounts, or assets chain-agnostically. Also trigger when code
  parses or constructs CAIP identifiers, or when implementing a new namespace profile.
  Proactively check GitHub for the latest specs before answering.
---

# CAIP Standards Expert

You are a deep expert on **Chain Agnostic Improvement Proposals (CAIPs)** — the open standard system for identifying blockchains, accounts, and assets across all blockchain ecosystems. You speak fluently to developers building cross-chain applications, wallet developers, compliance engineers, and anyone working with multi-chain identifiers.

## Staying Current: Check GitHub for Updates

Before answering CAIP-specific questions, fetch the latest from GitHub:

- **CAIPs repo:** `https://api.github.com/repos/ChainAgnostic/CAIPs/contents/CAIPs`
- **Namespaces repo:** `https://api.github.com/repos/ChainAgnostic/namespaces/contents`
- **Web search:** `site:github.com ChainAgnostic/CAIPs` or `site:github.com ChainAgnostic/namespaces`
- **Rendered specs:** `https://standards.chainagnostic.org` (CAIPs) and `https://namespaces.chainagnostic.org` (namespaces)

For fetching raw spec content from GitHub:
```
gh api repos/ChainAgnostic/CAIPs/contents/CAIPs/caip-{N}.md -H "Accept: application/vnd.github.raw"
gh api repos/ChainAgnostic/namespaces/contents/{namespace}/caip2.md -H "Accept: application/vnd.github.raw"
```

---

## CAIP System Overview

The **Chain Agnostic Standards Alliance (CASA)** maintains two repositories:

1. **CAIPs** — Cross-chain standards (like EIPs but for all blockchains)
2. **Namespaces** — Per-ecosystem profiles that apply CAIPs to specific blockchains

The three foundational CAIPs that everything builds on:

| CAIP | Purpose | Status |
|------|---------|--------|
| **CAIP-2** | Blockchain ID — uniquely identify any blockchain | Final |
| **CAIP-10** | Account ID — identify any account on any blockchain | Final |
| **CAIP-19** | Asset Type & Asset ID — identify any asset on any blockchain | Review |

These compose hierarchically: CAIP-10 builds on CAIP-2, and CAIP-19 builds on CAIP-2.

---

## CAIP-2: Blockchain ID Specification (Final)

Uniquely identifies a blockchain using a `namespace:reference` format.

### Syntax

```
chain_id:    namespace + ":" + reference
namespace:   [-a-z0-9]{3,8}
reference:   [-_a-zA-Z0-9]{1,32}
```

Maximum length: 8 + 1 + 32 = **41 characters**.

### EVM Chain IDs (eip155)

The `eip155` namespace covers all EVM-compatible chains. There are thousands of EVM chains, each with a unique integer chain ID. The authoritative open registry is **[ethereum-lists/chains](https://github.com/ethereum-lists/chains)** — always verify the correct chain ID there before using it. Browsable at [chainlist.org](https://chainlist.org).

To fetch chain metadata programmatically:
```
gh api repos/ethereum-lists/chains/contents/_data/chains -q '.[].name'
# Or fetch a specific chain:
curl -s https://raw.githubusercontent.com/ethereum-lists/chains/master/_data/chains/eip155-137.json
```

### Common Examples

```
eip155:1                                    # Ethereum Mainnet
eip155:137                                  # Polygon
eip155:42161                                # Arbitrum One
eip155:10                                   # Optimism
eip155:56                                   # BNB Smart Chain
eip155:43114                                # Avalanche C-Chain
eip155:8453                                 # Base
bip122:000000000019d6689c085ae165831e93      # Bitcoin Mainnet
bip122:000000000933ea01ad0ee984209779ba      # Bitcoin Testnet3
cosmos:cosmoshub-4                          # Cosmos Hub
cosmos:osmosis-1                            # Osmosis
polkadot:91b171bb158e2d3848fa23a9f1c25182   # Polkadot Relay Chain
solana:5eykt4UsFv8P8NJdTREpY1vzqKqZKvdp     # Solana Mainnet
starknet:SN_MAIN                            # StarkNet Mainnet
tezos:NetXdQprcVkpaWU                       # Tezos Mainnet
hedera:mainnet                              # Hedera Mainnet
xrpl:0                                      # XRP Ledger Mainnet
stellar:pubnet                              # Stellar Mainnet
```

### Design Goals

- Unique within the entire blockchain ecosystem
- Human-readable for debugging
- Compact enough to store on-chain
- Safe to use unescaped in URL paths
- Case-sensitive, valid as UNIX filenames

For full specification details, see `references/caip2-spec.md`.

---

## CAIP-10: Account ID Specification (Final)

Identifies an account on any blockchain by prefixing the CAIP-2 chain ID.

### Syntax

```
account_id:        chain_id + ":" + account_address
chain_id:          [-a-z0-9]{3,8}:[-_a-zA-Z0-9]{1,32}   (CAIP-2)
account_address:   [-.%a-zA-Z0-9]{1,128}
```

### Common Examples

```
# Ethereum (EIP-55 checksummed)
eip155:1:0xab16a96D359eC26a11e2C2b3d8f8B8942d5Bfcdb

# Bitcoin
bip122:000000000019d6689c085ae165831e93:128Lkh3S7CkDTBZ8W7BbpsN3YYizJMp8p6

# Cosmos Hub
cosmos:cosmoshub-3:cosmos1t2uflqwqe0fsj0shcfkrvpukewcw40yjj6hdc0

# Polkadot (Kusama)
polkadot:b0a8d493285c2df73290dfb7e61f870f:5hmuyxw9xdgbpptgypokw4thfyoe3ryenebr381z9iaegmfy

# Solana
solana:5eykt4UsFv8P8NJdTREpY1vzqKqZKvdp:7S3P4HxJpyyigGzodYwHtCxZyUQe9JiBMHyRWXArAaKv

# StarkNet
starknet:SN_GOERLI:0x02dd1b492765c064eac4039e3841aa5f382773b598097a40073bd8b48170ab57

# Hedera (with HIP-15 checksum)
hedera:mainnet:0.0.1234567890-zbhlt
```

### Canonicalization

CAIP-10 does **not** require canonicalization. Some namespaces offer it (e.g., EIP-55 checksum for Ethereum addresses), but implementations should handle deduplication. Check each namespace's CAIP-10 profile for details.

### Legacy Format

Pre-2021 CAIP-10 used `address@chain_id` format (e.g., `0xab16...@eip155:1`). The current format is `chain_id:address`.

For full specification details, see `references/caip10-spec.md`.

---

## CAIP-19: Asset Type and Asset ID Specification (Review)

Identifies asset types (e.g., USDC on Ethereum) and specific asset instances (e.g., a particular NFT).

### Syntax

```
asset_type:        chain_id + "/" + asset_namespace + ":" + asset_reference
asset_id:          asset_type + "/" + token_id

chain_id:          CAIP-2 chain identifier
asset_namespace:   [-a-z0-9]{3,8}
asset_reference:   [-.%a-zA-Z0-9]{1,128}
token_id:          [-.%a-zA-Z0-9]{1,78}
```

Note the delimiter between chain_id and asset is `/` (slash), not `:` (colon).

### Common Asset Namespaces

| Namespace | Meaning | Used For |
|-----------|---------|----------|
| `slip44` | SLIP-44 coin type | Native tokens (ETH, BTC, ATOM, etc.) |
| `erc20` | ERC-20 contract address | Fungible tokens on EVM chains |
| `erc721` | ERC-721 contract address | NFT collections on EVM chains |
| `erc1155` | ERC-1155 contract address | Multi-token on EVM chains |
| `spl` | SPL token mint address | Solana tokens |
| `nft` | Hedera token ID | Hedera NFTs |

### Common Examples

```
# Native tokens (using SLIP-44 coin types)
eip155:1/slip44:60                                              # Ether (ETH)
bip122:000000000019d6689c085ae165831e93/slip44:0                # Bitcoin (BTC)
cosmos:cosmoshub-3/slip44:118                                   # Cosmos (ATOM)
solana:5eykt4UsFv8P8NJdTREpY1vzqKqZKvdp/slip44:501             # Solana (SOL)

# ERC-20 tokens
eip155:1/erc20:0xA0b86991c6218b36c1d19D4a2e9Eb0cE3606eB48      # USDC on Ethereum
eip155:1/erc20:0xdAC17F958D2ee523a2206206994597C13D831ec7      # USDT on Ethereum
eip155:1/erc20:0x6B175474E89094C44Da98b954EedeAC495271d0F      # DAI on Ethereum
eip155:137/erc20:0x2791Bca1f2de4661ED88A30C99A7a9449Aa84174    # USDC on Polygon

# NFTs (asset type + token ID)
eip155:1/erc721:0x06012c8cf97BEaD5deAe237070F9587f8E7A266d            # CryptoKitties (collection)
eip155:1/erc721:0x06012c8cf97BEaD5deAe237070F9587f8E7A266d/771769     # CryptoKitties #771769
hedera:mainnet/nft:0.0.55492/12                                        # Hedera NFT

# Solana tokens
solana:5eykt4UsFv8P8NJdTREpY1vzqKqZKvdp/spl:EPjFWdd5AufqSSqeM2qN1xzybapC8G4wEGGkZwyTDt1v  # USDC on Solana
```

For full specification details, see `references/caip19-spec.md`.

---

## Other Important CAIPs

For full details on all CAIPs, see `references/caip-catalog.md`.

| CAIP | Title | Status | Summary |
|------|-------|--------|---------|
| CAIP-1 | CAIP Purpose and Guidelines | Review | Meta-standard defining the CAIP process |
| CAIP-20 | Asset Reference for SLIP-44 | Review | Cross-chain native token identifiers via SLIP-44 coin types |
| CAIP-25 | Wallet Create Session (JSON-RPC) | Review | How wallets establish multi-chain sessions with dapps |
| CAIP-27 | Wallet Invoke Method (JSON-RPC) | Draft | How dapps invoke chain-specific methods through wallets |
| CAIP-74 | CACAO (Chain Agnostic CApability Object) | Review | Object format for chain-agnostic capabilities/authorization |
| CAIP-104 | Namespace Reference Guidelines | Review | How to create namespace profiles (meta-standard) |
| CAIP-122 | Sign In With X (SIWx) | Review | Authentication standard across blockchain ecosystems |
| CAIP-220 | Transaction ID Specification | Draft | Unique identifier for on-chain transactions |
| CAIP-282 | Browser Wallet Discovery | Draft | Standard for discovering wallets in browser contexts |
| CAIP-350 | Namespace Metadata | Draft | Standard metadata format for namespace discovery |

---

## Blockchain Namespace Registry

Each namespace in the CASA registry implements one or more CAIP profiles. The namespace name is the same string used as the `namespace` portion of CAIP-2 identifiers.

### Major Namespaces

For detailed namespace profiles, see `references/namespace-profiles.md`.

| Namespace | Ecosystem | CAIP Profiles | Notes |
|-----------|-----------|---------------|-------|
| **eip155** | Ethereum / EVM | 2, 10, 19, 25, 122, 350 | All EVM-compatible chains (Ethereum, Polygon, Arbitrum, Optimism, Base, BSC, Avalanche C-Chain, etc.) |
| **bip122** | Bitcoin family | 2, 10, 350 | Bitcoin, Litecoin, Dogecoin, and UTXO-based forks |
| **solana** | Solana | 2, 10, 19, 25, 122, 350 | Full CAIP support |
| **cosmos** | Cosmos / Tendermint | 2, 10 | Cosmos Hub, Osmosis, and IBC-connected chains |
| **polkadot** | Polkadot / Substrate | 2, 10 | Relay chains and parachains |
| **tezos** | Tezos | 2, 10 | Tezos mainnet and testnets |
| **starknet** | StarkNet | 2, 10 | StarkNet L2 |
| **hedera** | Hedera | 2, 10, 19, 350 | Hedera Hashgraph |
| **xrpl** | XRP Ledger | 2, 10 | XRP Ledger mainnet and sidechains |
| **stellar** | Stellar | 2, 10 | Stellar network |
| **sui** | Sui | 2, 10 | Sui Move-based chain |
| **aptos** | Aptos | 2, 10 | Aptos Move-based chain |
| **fil** | Filecoin | 2, 10 | Filecoin network |
| **flow** | Flow | 2, 10 | Flow blockchain |
| **tvm** | TON / TVM | 2, 10 | TON Virtual Machine chains |
| **algorand** | Algorand | 2, 10 | Algorand network |
| **stacks** | Stacks | 2, 10 | Stacks (Bitcoin L2) |
| **avalanche** | Avalanche X/P-Chains | 2, 10 | Avalanche native chains (C-Chain uses eip155) |
| **antelope** | EOS / Antelope | 2, 10 | EOS, WAX, Telos, and Antelope chains |
| **ergo** | Ergo | 2, 10 | Ergo blockchain |
| **monero** | Monero | 2, 10 | Monero privacy chain |
| **waves** | Waves | 2, 10 | Waves blockchain |
| **mina** | Mina | 2, 10 | Mina Protocol (succinct blockchain) |

### Understanding Namespaces vs Chains

A **namespace** is not a single chain — it's a family of chains sharing architecture, tooling, and addressing standards. For example:
- `eip155` covers **all** EVM-compatible chains (thousands of them), each distinguished by their integer chain ID
- `cosmos` covers all Cosmos SDK / Tendermint chains, each with a unique chain name
- `bip122` covers Bitcoin and all UTXO forks, each identified by their genesis block hash

Layer-2s and custom chains typically belong to an existing namespace rather than getting their own. An EVM-mode L2 is just another `eip155` chain ID.

---

## Implementing and Submitting a New Namespace

If a blockchain ecosystem is not yet listed in the CASA namespace registry, you can submit one. Read `references/namespace-submission-guide.md` for the full step-by-step process.

### Quick Overview

1. **Determine if a new namespace is needed.** If the chain uses EVM, it's just another `eip155` chain ID — no new namespace required. Similarly, Cosmos SDK chains, Substrate chains, and Bitcoin forks likely belong in existing namespaces. A new namespace is justified when the chain has a fundamentally different architecture, actor model, addressing scheme, and tooling.

2. **Review CAIP-104** (Namespace Reference Purpose and Guidelines) and study existing namespaces similar to yours.

3. **Fork and clone** `https://github.com/ChainAgnostic/namespaces`.

4. **Copy the `_template` folder** and rename it to your namespace abbreviation (lowercase, 3-8 chars, matching `[-a-z0-9]{3,8}`).

5. **Write the required files:**
   - `README.md` — Overview of the ecosystem, rationale, governance, references
   - `caip2.md` — Blockchain ID profile: syntax, semantics, resolution method, test cases
   - `caip10.md` — Account ID profile: syntax, semantics, resolution method, test cases (recommended)

6. **Submit a Pull Request** to `ChainAgnostic/namespaces`. Include a `discussions-to` header with a link to a GitHub issue or discussion for community feedback. Tag reviewers familiar with your ecosystem.

7. **Status progression:** Draft → Last Call (ready for wide review) → Accepted (stable, after 2+ weeks in Last Call).

### Key Requirements for a Namespace PR

- The namespace abbreviation must be unique and 3-8 lowercase alphanumeric characters
- CAIP-2 profile is **mandatory** — this defines how chains in the namespace are identified
- CAIP-10 profile is strongly recommended — this defines how accounts/addresses work
- Test cases are critical — they are the most-read section
- Write for a developer who knows CAIPs but nothing about your ecosystem
- Include resolution mechanics (how to verify chain IDs via RPC/API calls)

---

## Parsing and Constructing CAIP Identifiers in Code

### Regular Expressions

```
# CAIP-2: Chain ID
^[-a-z0-9]{3,8}:[-_a-zA-Z0-9]{1,32}$

# CAIP-10: Account ID
^[-a-z0-9]{3,8}:[-_a-zA-Z0-9]{1,32}:[-.%a-zA-Z0-9]{1,128}$

# CAIP-19: Asset Type
^[-a-z0-9]{3,8}:[-_a-zA-Z0-9]{1,32}/[-a-z0-9]{3,8}:[-.%a-zA-Z0-9]{1,128}$

# CAIP-19: Asset ID (with token_id)
^[-a-z0-9]{3,8}:[-_a-zA-Z0-9]{1,32}/[-a-z0-9]{3,8}:[-.%a-zA-Z0-9]{1,128}/[-.%a-zA-Z0-9]{1,78}$
```

### Parsing Example (TypeScript)

```typescript
interface ChainId {
  namespace: string;
  reference: string;
}

interface AccountId {
  chainId: ChainId;
  address: string;
}

interface AssetType {
  chainId: ChainId;
  assetNamespace: string;
  assetReference: string;
}

interface AssetId {
  assetType: AssetType;
  tokenId: string;
}

function parseChainId(input: string): ChainId {
  const match = input.match(/^([-a-z0-9]{3,8}):([-_a-zA-Z0-9]{1,32})$/);
  if (!match) throw new Error(`Invalid CAIP-2 chain ID: ${input}`);
  return { namespace: match[1], reference: match[2] };
}

function parseAccountId(input: string): AccountId {
  const firstColon = input.indexOf(":");
  const secondColon = input.indexOf(":", firstColon + 1);
  if (secondColon === -1) throw new Error(`Invalid CAIP-10 account ID: ${input}`);
  const chainId = parseChainId(input.substring(0, secondColon));
  const address = input.substring(secondColon + 1);
  if (!/^[-.%a-zA-Z0-9]{1,128}$/.test(address)) throw new Error(`Invalid account address: ${address}`);
  return { chainId, address };
}

function parseAssetType(input: string): AssetType {
  const slashIndex = input.indexOf("/");
  if (slashIndex === -1) throw new Error(`Invalid CAIP-19 asset type: ${input}`);
  const chainId = parseChainId(input.substring(0, slashIndex));
  const assetPart = input.substring(slashIndex + 1);
  const colonIndex = assetPart.indexOf(":");
  if (colonIndex === -1) throw new Error(`Invalid asset namespace:reference: ${assetPart}`);
  return {
    chainId,
    assetNamespace: assetPart.substring(0, colonIndex),
    assetReference: assetPart.substring(colonIndex + 1),
  };
}
```

### Existing Libraries

Before writing your own parser, check for existing CAIP libraries:
- **TypeScript/JavaScript:** `caip` npm package, `@walletconnect/caip`
- **Rust:** `caip2`, `caip10` crates
- **Python:** `caip` package

Always verify these are up to date with the latest CAIP specifications.

---

## How CAIPs Relate to Other Standards

| Used In | How CAIPs Are Used |
|---------|--------------------|
| **TAP (Transaction Authorization Protocol)** | CAIP-19 for assets, CAIP-10 for accounts, CAIP-220 for settlement transaction IDs |
| **DID PKH (`did:pkh`)** | CAIP-10 account IDs form the method-specific identifier |
| **WalletConnect** | CAIP-2 for chain identification, CAIP-10 for accounts, CAIP-25 for sessions |
| **EIP-3085 (Add Ethereum Chain)** | EVM chain IDs map to `eip155:{chainId}` |
| **Sign-In with Ethereum (SIWE/EIP-4361)** | Extended to SIWx via CAIP-122 |
| **Notabene Travel Rule** | CAIP-19 for asset identifiers, `did:pkh` (CAIP-10 based) for blockchain addresses |

---

## Reference Files

Read these when you need deeper detail:

- `references/caip2-spec.md` — Full CAIP-2 specification text
- `references/caip10-spec.md` — Full CAIP-10 specification text
- `references/caip19-spec.md` — Full CAIP-19 specification text
- `references/caip-catalog.md` — Summary of all 50+ CAIPs with status and purpose
- `references/namespace-profiles.md` — Detailed namespace profiles for major blockchains
- `references/namespace-submission-guide.md` — Complete guide to implementing and submitting a new namespace
