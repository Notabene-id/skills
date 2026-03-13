# nb-skills — Claude Skills Marketplace

This repository contains Claude Code skills distributed as `.skill` files (zip archives).

## Repository structure

Each skill has a source directory and a packaged `.skill` file:

```
<skill-name>/          # Source directory
  SKILL.md             # Frontmatter (name, description) + skill prompt
  references/          # Supporting docs, API refs, guides
<skill-name>.skill     # Packaged zip of the source directory
```

## Rules

### Keep .skill files in sync

Whenever you modify any file inside a skill's source directory (e.g. `notabene-api/references/wallet-service-guide.md` or `notabene-api/SKILL.md`), you **must** rebuild the `.skill` file before finishing:

```sh
zip -r <skill-name>.skill <skill-name>/
```

Never leave a `.skill` file out of sync with its source directory.

### Description length limit

The `description` field in each `SKILL.md` frontmatter **must be under 1024 characters**. After editing a description, verify:

```sh
awk '/^description: >/{found=1; next} found && /^---$/{exit} found{gsub(/^  /,""); printf "%s ", $0}' <skill-name>/SKILL.md | wc -c
```

### Skill correctness checks

When reviewing or editing a skill, verify:

1. **Frontmatter is valid** — `name` and `description` fields exist between `---` delimiters
2. **Description < 1024 chars** — see above
3. **References are accurate** — API endpoints, request/response bodies, and field names match the actual Notabene/TAP/CAIP documentation. When in doubt, fetch the latest docs from the web.
4. **No stale information** — if external APIs or specs have changed, update the skill to match
5. **Zip is current** — the `.skill` file contains exactly the contents of the source directory

### Commit hygiene

- Commit source directory changes and the rebuilt `.skill` file together
- Keep commits focused: one skill per commit when possible
