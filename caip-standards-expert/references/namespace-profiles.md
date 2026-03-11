# Namespace Profiles — Major Blockchain Ecosystems

> Always fetch the latest namespaces: `gh api repos/ChainAgnostic/namespaces/contents -q '.[].name' | sort`
> Fetch a namespace README: `gh api repos/ChainAgnostic/namespaces/contents/{namespace}/README.md -H "Accept: application/vnd.github.raw"`
> Fetch a CAIP profile: `gh api repos/ChainAgnostic/namespaces/contents/{namespace}/caip{N}.md -H "Accept: application/vnd.github.raw"`
> Browse rendered: https://namespaces.chainagnostic.org

## eip155 — Ethereum / EVM Chains

The largest namespace, covering **all** EVM-compatible chains. Thousands of chains share this namespace.

**CAIP-2 (Chain ID):** Integer chain ID from EIP-155, resolved via `eth_chainId` JSON-RPC.

```
eip155:1        # Ethereum Mainnet
eip155:5        # Goerli Testnet
eip155:11155111 # Sepolia Testnet
eip155:137      # Polygon
eip155:42161    # Arbitrum One
eip155:10       # Optimism
eip155:8453     # Base
eip155:56       # BNB Smart Chain
eip155:43114    # Avalanche C-Chain
eip155:250      # Fantom
eip155:100      # Gnosis Chain
```

Chain IDs are self-registered at [ethereum-lists/chains](https://github.com/ethereum-lists/chains) and browsable at [chainlist.org](https://chainlist.org).

**CAIP-10 (Account ID):** `eip155:{chainId}:0x{40 hex chars}`. Optional EIP-55 checksum capitalization.

```
eip155:1:0xab16a96D359eC26a11e2C2b3d8f8B8942d5Bfcdb
```

**CAIP-19 (Assets):**
- Native ETH: `eip155:1/slip44:60`
- ERC-20 tokens: `eip155:{chainId}/erc20:0x{contractAddress}`
- ERC-721 NFTs: `eip155:{chainId}/erc721:0x{contractAddress}[/{tokenId}]`
- ERC-1155: `eip155:{chainId}/erc1155:0x{contractAddress}[/{tokenId}]`

```
eip155:1/erc20:0xA0b86991c6218b36c1d19D4a2e9Eb0cE3606eB48      # USDC
eip155:1/erc20:0xdAC17F958D2ee523a2206206994597C13D831ec7      # USDT
eip155:137/erc20:0x2791Bca1f2de4661ED88A30C99A7a9449Aa84174    # USDC on Polygon
eip155:1/erc721:0xBC4CA0EdA7647A8aB7C2061c2E118A18a936f13D/1234  # BAYC #1234
```

**Also supports:** CAIP-25, CAIP-122 (SIWE/SIWx), CAIP-350

---

## bip122 — Bitcoin Family

Covers Bitcoin and all UTXO-based forks (Litecoin, Dogecoin, Bitcoin Cash, etc.).

**CAIP-2 (Chain ID):** First 32 hex characters of the genesis block hash.

```
bip122:000000000019d6689c085ae165831e93   # Bitcoin Mainnet
bip122:000000000933ea01ad0ee984209779ba   # Bitcoin Testnet3
bip122:12a765e31ffd4059bada1e25190f6e98   # Litecoin
bip122:fdbe99b90c90bae7505796461471d89a   # Feathercoin
```

**CAIP-10 (Account ID):**
```
bip122:000000000019d6689c085ae165831e93:128Lkh3S7CkDTBZ8W7BbpsN3YYizJMp8p6
```

**CAIP-19 (Assets):** Native tokens use SLIP-44.
```
bip122:000000000019d6689c085ae165831e93/slip44:0   # BTC
bip122:12a765e31ffd4059bada1e25190f6e98/slip44:2   # LTC
```

---

## solana — Solana

**CAIP-2 (Chain ID):** First 32 characters of the genesis block hash (base58).

```
solana:5eykt4UsFv8P8NJdTREpY1vzqKqZKvdp     # Mainnet Beta
solana:4uhcVJyU9pJkvQyS88uRDiswHXSCkY3z     # Testnet
solana:EtWTRABZaYq6iMfeYKouRu166VU2xqa1      # Devnet
```

**CAIP-10:** `solana:{genesisHash}:{base58PublicKey}`
```
solana:5eykt4UsFv8P8NJdTREpY1vzqKqZKvdp:7S3P4HxJpyyigGzodYwHtCxZyUQe9JiBMHyRWXArAaKv
```

**CAIP-19:**
```
solana:5eykt4UsFv8P8NJdTREpY1vzqKqZKvdp/slip44:501                                         # SOL
solana:5eykt4UsFv8P8NJdTREpY1vzqKqZKvdp/spl:EPjFWdd5AufqSSqeM2qN1xzybapC8G4wEGGkZwyTDt1v  # USDC
```

**Also supports:** CAIP-25, CAIP-122, CAIP-350

---

## cosmos — Cosmos / Tendermint SDK

Covers all Cosmos SDK chains connected via IBC.

**CAIP-2 (Chain ID):** The chain's `chain-id` string from its genesis/config.

```
cosmos:cosmoshub-4          # Cosmos Hub
cosmos:osmosis-1            # Osmosis
cosmos:juno-1               # Juno
cosmos:Binance-Chain-Tigris # Binance Chain
cosmos:iov-mainnet          # IOV
```

**CAIP-10:** `cosmos:{chainId}:{bech32Address}`
```
cosmos:cosmoshub-4:cosmos1t2uflqwqe0fsj0shcfkrvpukewcw40yjj6hdc0
```

**CAIP-19:**
```
cosmos:cosmoshub-3/slip44:118              # ATOM
cosmos:Binance-Chain-Tigris/slip44:714     # BNB (native)
```

---

## polkadot — Polkadot / Substrate

Covers Polkadot relay chain, Kusama, and all parachains.

**CAIP-2 (Chain ID):** First 32 hex characters of genesis block hash (no `0x` prefix).

```
polkadot:91b171bb158e2d3848fa23a9f1c25182   # Polkadot Relay Chain
polkadot:b0a8d493285c2df73290dfb7e61f870f   # Kusama
```

**CAIP-10:** Uses SS58 addresses (lowercase).
```
polkadot:91b171bb158e2d3848fa23a9f1c25182:1zugcag7cJVBtVRnFxv556NtLemoKKm4
```

---

## Other Notable Namespaces

| Namespace | Chain ID Format | Account Format | Notes |
|-----------|----------------|----------------|-------|
| **tezos** | `tezos:{netHash}` (base58) | tz1/tz2/tz3/KT1 addresses | Tezos mainnet: `tezos:NetXdQprcVkpaWU` |
| **starknet** | `starknet:{networkName}` | 0x-prefixed felt addresses | `starknet:SN_MAIN`, `starknet:SN_GOERLI` |
| **hedera** | `hedera:{network}` | Entity IDs (0.0.X) | `hedera:mainnet`, supports HIP-15 checksums |
| **xrpl** | `xrpl:{networkId}` | r-addresses | `xrpl:0` (mainnet) |
| **stellar** | `stellar:{networkName}` | G.../M... addresses | `stellar:pubnet` |
| **sui** | Genesis hash prefix | 0x-prefixed addresses | Sui Move-based chain |
| **aptos** | Network identifier | 0x-prefixed addresses | Aptos Move-based chain |
| **fil** | `fil:{networkName}` | f0/f1/f2/f3/f4 addresses | Filecoin |
| **tvm** | Workchain + shard info | Raw/bounceable addresses | TON Virtual Machine chains |
| **algorand** | Genesis hash prefix | Base32 addresses | Algorand |
| **stacks** | `stacks:{chainId}` | SP/ST/SN addresses | Bitcoin L2 |
| **flow** | `flow:{networkName}` | Hex addresses | Flow blockchain |
| **mina** | Network identifier | B62 addresses | Mina Protocol |
| **antelope** | Chain ID hash | 12-char account names | EOS, WAX, Telos |
| **waves** | Chain byte | Base58 addresses | Waves blockchain |
| **monero** | Network byte | Long hex addresses | Monero privacy chain |
| **ergo** | Genesis hash prefix | Base58 addresses | Ergo blockchain |

## Complete Namespace List

As of last update, registered namespaces include:

```
aleo, alephium, algorand, antelope, aptos, arweave, avalanche,
bip122, casper, ccd, chia, conflux, cosmos, eip155, ergo, fil,
flow, haneul, hedera, hive, iota, koinos, mina, monero, mvx,
partisia, polkadot, quai, qubic, reef, solana, stacks, starknet,
stellar, sui, swift, tezos, tvm, vechain, wallet, waves, xrpl
```

Always fetch the current list from GitHub as new namespaces are added regularly.
