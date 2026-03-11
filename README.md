# nb-skills

Claude Code skills for working with Notabene APIs and related standards.

Install these skills in your Claude Code project to give your coding agent deep knowledge of Notabene integrations, the TAP protocol, and CAIP standards.

## Skills

| Skill | Description |
|-------|-------------|
| [notabene-api](./notabene-api) | Integration guide for the Notabene Travel Rule API, Flow pay-ins API, and JavaScript SDK |
| [tap-protocol-expert](./tap-protocol-expert) | Deep expertise on the Transaction Authorization Protocol (TAP) and TAIPs |
| [caip-standards-expert](./caip-standards-expert) | Expert on Chain Agnostic Improvement Proposals (CAIPs) and the CASA namespace registry |

## Installation

### Option 1: Plugin Marketplace (recommended)

Add the marketplace and install the plugins you need:

```
/plugin marketplace add Notabene-id/skills
/plugin install notabene-api@notabene-skills
/plugin install tap-protocol-expert@notabene-skills
/plugin install caip-standards-expert@notabene-skills
```

### Option 2: Manual configuration

Add skills directly to your project's `.claude/settings.json`:

```json
{
  "skills": [
    "github:notabene/nb-skills/notabene-api",
    "github:notabene/nb-skills/tap-protocol-expert",
    "github:notabene/nb-skills/caip-standards-expert"
  ]
}
```

## License

This project is licensed under the [MIT License](./LICENSE).

## Resources

- [Notabene Developer Experience Portal](https://devx.notabene.id/)
- [TAP Protocol](https://github.com/TransactionAuthorizationProtocol/TAIPs)
- [CAIP Standards](https://github.com/ChainAgnostic/CAIPs)
