# Guide: Implementing and Submitting a New CASA Namespace

This guide walks through the complete process of creating and submitting a namespace profile for a blockchain ecosystem not yet listed in the [CASA namespace registry](https://namespaces.chainagnostic.org).

> **Before starting:** Fetch the latest contributing guidelines:
> `gh api repos/ChainAgnostic/namespaces/contents/CONTRIBUTING.md -H "Accept: application/vnd.github.raw"`

---

## Step 0: Do You Actually Need a New Namespace?

**Most chains do NOT need a new namespace.** Check first:

| If your chain is... | It belongs in... | Action needed |
|---------------------|-----------------|---------------|
| EVM-compatible (uses `eth_chainId`) | `eip155` namespace | Just register your chain ID at [ethereum-lists/chains](https://github.com/ethereum-lists/chains) |
| A Cosmos SDK / Tendermint chain | `cosmos` namespace | Add a section to the `cosmos` namespace docs if needed |
| A Substrate / Polkadot parachain | `polkadot` namespace | Add a section to the `polkadot` namespace docs if needed |
| A Bitcoin / UTXO fork | `bip122` namespace | Add your genesis hash to `bip122` examples |
| An "EVM-mode" Layer-2 | `eip155` namespace | Just another EVM chain ID |

**A new namespace IS justified when** the chain has a fundamentally different:
- Architecture and consensus model
- Account/address format and actor model
- Signing and transaction format
- RPC interface and tooling
- Security model

**Think of a namespace as:** a set of architectural assumptions, security models, actor models, and standards — addressing standards, runtimes, etc. The superset of possible chains sharing the same tooling and interfaces.

---

## Step 1: Study the Standards

Read these before writing anything:

1. **[CAIP-104](https://chainagnostic.org/CAIPs/caip-104)** — Namespace Reference Purpose and Guidelines (the meta-standard)
2. **[CAIP-2](https://chainagnostic.org/CAIPs/caip-2)** — Blockchain ID Specification
3. **[CAIP-10](https://chainagnostic.org/CAIPs/caip-10)** — Account ID Specification
4. **[CAIP-19](https://chainagnostic.org/CAIPs/caip-19)** — Asset Type and Asset ID Specification

Study 3-5 existing namespaces similar to yours:
```bash
# Browse similar namespaces
gh api repos/ChainAgnostic/namespaces/contents/{similar-namespace} -q '.[].name'

# Read their profiles
gh api repos/ChainAgnostic/namespaces/contents/{similar-namespace}/README.md -H "Accept: application/vnd.github.raw"
gh api repos/ChainAgnostic/namespaces/contents/{similar-namespace}/caip2.md -H "Accept: application/vnd.github.raw"
```

---

## Step 2: Fork and Clone

```bash
gh repo fork ChainAgnostic/namespaces --clone
cd namespaces
```

---

## Step 3: Create from Template

```bash
cp -r _template {your-namespace}
```

Your namespace abbreviation must be:
- **3-8 characters** (matching `[-a-z0-9]{3,8}`)
- **All lowercase**
- **Unique** across all registered namespaces
- **Descriptive** — should be recognizable to developers in the ecosystem

---

## Step 4: Write the README.md

The README introduces the ecosystem to developers who know CAIPs but nothing about your chain. Required sections:

```markdown
---
namespace-identifier: {your-namespace}
title: {Ecosystem Name}
author: ["Your Name (@github)"]
status: Draft
type: Informational
created: {YYYY-MM-DD}
---

# Namespace for {Ecosystem Name}

{1-2 paragraph layman-accessible explanation of what makes this ecosystem
unique and how it differs from more familiar chains like Ethereum.}

## Rationale

{~200 words on technical issues addressed by naming this namespace,
and any particularities developers should know.}

## Governance

{~200 words on the improvement proposal process, specification governance,
and any context a cross-chain implementer should know.}

## References

- [{Ecosystem docs}][] - {description}
- [{Key spec/standard}][] - {description}

[{Ecosystem docs}]: https://...
[{Key spec/standard}]: https://...

## Copyright
Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).
```

---

## Step 5: Write the CAIP-2 Profile (Required)

This is the **most important** profile — it defines how blockchains in your namespace are identified. File: `caip2.md`

Must include:

### Semantics
What inputs are needed to form a chain ID? What does the reference represent?

### Syntax
The exact algorithm to construct a valid CAIP-2 from native inputs. Include a regex for validation:
```
namespace:   {your-namespace}
reference:   {your regex, must fit [-_a-zA-Z0-9]{1,32}}
```

### Resolution Mechanics
How to verify a chain ID via RPC/API. Include a sample request/response:
```json
// Request
{ "method": "your_method", "params": [] }

// Response
{ "result": "..." }
```

### Test Cases
**This is the most-read section.** Include 3-5 manually composed, validated examples:
```
{your-namespace}:{mainnet-reference}    # Mainnet
{your-namespace}:{testnet-reference}    # Testnet
```

---

## Step 6: Write the CAIP-10 Profile (Strongly Recommended)

File: `caip10.md` — defines how accounts/addresses work in your namespace.

Must include:
- **Semantics** — what is an "account" or "address" in this ecosystem?
- **Syntax** — regex for `account_address` within the CAIP-10 format
- **Resolution mechanics** — how to validate an address via RPC
- **Canonicalization** — does the ecosystem have address checksums or normalization?
- **Test cases** — 3-5 examples with real-looking addresses

---

## Step 7: Optional Additional Profiles

Consider adding if your namespace supports them:

| Profile | File | When to include |
|---------|------|-----------------|
| CAIP-19 | `caip19.md` | If your chain has tokens/NFTs with standard addressing |
| CAIP-122 | `caip122.md` | If your chain supports sign-in / message signing |
| CAIP-25 | `caip25.md` | If wallets support session creation |
| CAIP-350 | `caip350.md` | Namespace metadata for discovery |

---

## Step 8: Submit the Pull Request

```bash
git checkout -b add-{your-namespace}-namespace
git add {your-namespace}/
git commit -m "Add {your-namespace} namespace with CAIP-2 and CAIP-10 profiles"
git push origin add-{your-namespace}-namespace
gh pr create --title "Add {Ecosystem Name} namespace" --body "..."
```

### PR Requirements

- Include a `discussions-to` header in your frontmatter with a link to a GitHub issue or discussion
- Tag reviewers familiar with your ecosystem (you may be asked to do so by editors)
- Your first PR must include at minimum: `README.md` + `caip2.md`
- If you requested feedback from other teams before submitting, reference those discussions

### Review Process

1. An editor manually reviews the first PR for a new namespace
2. The editor may request changes or ask for additional reviewers
3. Status progression: **Draft** → **Last Call** (2+ weeks of wide review) → **Accepted**

---

## Step 9: Local Testing (Optional but Recommended)

The namespaces site uses Jekyll. To preview locally:

```bash
bundle install
bundle exec jekyll serve
```

Then browse to `http://localhost:4000/{your-namespace}/` to verify rendering.

---

## Style Guide Reminders

- One sentence per line (easier diff review)
- Use `[Name][]` link aliases, not inline `[Name](url)` links
- Define link aliases at the end of the References section
- Link to rendered CASA URLs (`chainagnostic.org/CAIPs/caip-2`), not GitHub URLs
- Images go in the same namespace directory with naming: `readme-1.png`, `readme-2.png`
- Use GitHub-flavored Markdown

---

## Common Pitfalls

1. **Namespace too broad or too narrow** — A namespace should cover an entire family of chains sharing tooling and interfaces, not just one chain
2. **Missing test cases** — Test cases are the most important section; reviewers will focus on these
3. **No resolution mechanics** — Reviewers want to see how a developer can programmatically verify chain IDs and addresses
4. **Writing for experts only** — Target audience is a developer familiar with CAIPs but NOT with your ecosystem
5. **Submitting without community review** — Get feedback from your ecosystem's developers first
