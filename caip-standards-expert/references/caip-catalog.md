# CAIP Catalog — All Chain Agnostic Improvement Proposals

> Always fetch the latest list: `gh api repos/ChainAgnostic/CAIPs/contents/CAIPs -q '.[].name'`
> Fetch a specific CAIP: `gh api repos/ChainAgnostic/CAIPs/contents/CAIPs/caip-{N}.md -H "Accept: application/vnd.github.raw"`
> Rendered specs: https://standards.chainagnostic.org

## Foundational Identifiers

| CAIP | Title | Status | Summary |
|------|-------|--------|---------|
| **2** | Blockchain ID Specification | **Final** | `namespace:reference` format for identifying any blockchain |
| **10** | Account ID Specification | **Final** | `chain_id:account_address` for identifying any blockchain account |
| **19** | Asset Type and Asset ID | Review | `chain_id/asset_namespace:asset_reference[/token_id]` for any asset |
| **20** | Asset Reference for SLIP-44 | Review | Cross-chain native token identification via SLIP-44 coin types |

## Meta / Process

| CAIP | Title | Status | Summary |
|------|-------|--------|---------|
| **1** | CAIP Purpose and Guidelines | Review | Defines the CAIP process itself |
| **104** | Namespace Reference Guidelines | Review | How to create and structure namespace profiles |

## Wallet Interaction (JSON-RPC)

| CAIP | Title | Status | Summary |
|------|-------|--------|---------|
| **25** | Wallet Create Session | Review | Multi-chain session negotiation between dapps and wallets |
| **27** | Wallet Invoke Method | Draft | Chain-specific method invocation through wallets |
| **282** | Browser Wallet Discovery | Draft | Standard for discovering wallet providers in browsers |
| **294** | Browser Wallet Messaging | Draft | Transport for wallet-dapp communication |
| **295** | Browser Wallet Lifecycle | Draft | Wallet session lifecycle management |

## Authentication & Authorization

| CAIP | Title | Status | Summary |
|------|-------|--------|---------|
| **74** | CACAO (Chain Agnostic CApability Object) | Review | Object format for chain-agnostic signed capabilities |
| **122** | Sign In With X (SIWx) | Review | Generic sign-in standard for any blockchain |

## Transaction & Settlement

| CAIP | Title | Status | Summary |
|------|-------|--------|---------|
| **220** | Transaction ID Specification | Draft | Unique identifier for on-chain transactions |

## Multi-Chain & Interoperability

| CAIP | Title | Status | Summary |
|------|-------|--------|---------|
| **50** | Multi-chain Account ID | Draft | Identifier for accounts across multiple chains |
| **350** | Namespace Metadata | Draft | Standard metadata format for namespace discovery and profiles |

## Legacy / Superseded

Some early CAIPs have been superseded by the namespace system (CAIP-104):

| CAIP | Title | Status | Superseded By |
|------|-------|--------|---------------|
| **3** | Blockchain ID for EIP-155 | Superseded | `eip155` namespace CAIP-2 profile |
| **4** | Blockchain ID for BIP-122 | Superseded | `bip122` namespace CAIP-2 profile |
| **5** | Blockchain ID for Cosmos | Superseded | `cosmos` namespace CAIP-2 profile |
| **6** | Blockchain ID for LIP-9 | Superseded | Namespace CAIP-2 profile |
| **7** | Blockchain ID for Stellar | Superseded | `stellar` namespace CAIP-2 profile |
| **21** | Asset Type for ERC-20 | Superseded | `eip155` namespace CAIP-19 profile |
| **22** | Asset Type for ERC-721 | Superseded | `eip155` namespace CAIP-19 profile |

## Additional CAIPs

The full repo contains 50+ CAIPs. For the complete and current list, fetch from GitHub:
```bash
gh api repos/ChainAgnostic/CAIPs/contents/CAIPs -q '.[].name'
```

Or browse at https://standards.chainagnostic.org
