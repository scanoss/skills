# SCANOSS agent skills

These skills teach AI coding agents to use SCANOSS tools. With them, an agent can scan code for open
source, licences, vulnerabilities, and cryptography, build CI checks from plain shell steps, and operate Earnie.

Skills follow the [Agent Skills](https://agentskills.io/specification) format. Each one is short and links
to the matching page on [docs.scanoss.com](https://docs.scanoss.com), which stays the full reference.
Every docs page is also served as Markdown if you append `.md` to its URL, and the full index is at
https://docs.scanoss.com/llms.txt.

## Install

All skills:

```bash
npx skills add scanoss/skills
```

One skill:

```bash
npx skills add scanoss/skills --skill scanoss-cli
```

List what is available without installing:

```bash
npx skills add scanoss/skills --list
```

## Skills

| Skill | What it does |
|---|---|
| [`scanoss-cli`](skills/scanoss-cli/SKILL.md) | Scan with scanoss-cli, produce SPDX/CycloneDX SBOMs, enrich inventories, look up PURLs, and write `scanoss.json` BOM rules. |
| [`crypto-finder`](skills/crypto-finder/SKILL.md) | Find cryptography in code and dependencies with Crypto Finder and export a CycloneDX 1.7 CBOM. |
| [`scanoss-api`](skills/scanoss-api/SKILL.md) | Call the SCANOSS v3 API directly: PURL lookups, batch responses, and the WFP scan flow. |
| [`scanoss-recipes`](skills/scanoss-recipes/SKILL.md) | Choose and adapt a SCANOSS recipe (PR scans, licence and vulnerability gates, release SBOMs, and more) to your CI and repository. |
| [`earnie`](skills/earnie/SKILL.md) | Operate Earnie: CLI, verdicts, pre-commit hook, MCP setup, Self-checks, merge gate, and the REST API. |
| [`migrate-to-earnie`](skills/migrate-to-earnie/SKILL.md) | Move a repository from the legacy SCANOSS toolchain to Earnie, phase by phase, with the known gaps. |

## Requirements

- `scanoss-cli`, `crypto-finder`, `scanoss-api`, `scanoss-recipes`: a SCANOSS API key for the hosted API,
  provided as `SCANOSS_API_KEY` (see [API Keys & Authentication](https://docs.scanoss.com/en/latest/developer-tools/authentication)).
- `earnie`, `migrate-to-earnie`: access to an Earnie organisation and an Earnie API key (`EARNIE_API_KEY`).

## Issues

Found a skill that is wrong, out of date, or triggers at the wrong time? Open an issue at
https://github.com/scanoss/skills/issues with the skill name, what you asked the agent, and what it did.
For product questions, see https://docs.scanoss.com.

## License

[MIT](LICENSE)
