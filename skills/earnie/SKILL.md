---
name: earnie
description: Operate Earnie, SCANOSS's governance platform for open source, cryptography, and AI provenance - install and sign in to the earnie CLI, run scans and read verdicts and exit codes, install the pre-commit hook, connect a coding agent over MCP with earnie mcp setup, run and read Self-checks, understand the merge gate, export SBOMs, and call the Earnie REST API with a Bearer API key and the right scopes. Use when the user mentions Earnie, earnie CLI, "earnie scan", "is this project blocked", "why did the Earnie check fail", "set up Earnie MCP", "earnie mcp doctor", Self-check, Policy Approval, EARNIE_API_KEY, or "call the Earnie API".
license: MIT
metadata:
  author: scanoss
  version: "1.0.0"
---

# Earnie

This skill teaches how to **operate** Earnie. It does not define the user's policies. If the repository's
instructions file (for example `AGENTS.md` or `CLAUDE.md`) contains a block between
`<!-- earnie:policies:start` and `<!-- earnie:policies:end -->`, that managed block is the project's policy
and wins. Never edit inside the markers or the markers themselves; Earnie rewrites the block when policies
change, and damaged markers stop the refresh. Docs: https://docs.scanoss.com/en/latest/earnie/introduction.md

## The model: one policy, three moments

| Moment | Where | What Earnie does |
|---|---|---|
| Write-time | Coding agents over MCP, the CLI, hooks | Prevent: policy context plus quick **Self-checks** on changed files. No findings created. |
| Merge-time | Pull/merge requests and CI | Enforce: scans the change, posts a gate with comments. |
| Anytime | Web app | Prove and maintain: full scans, posture, triage trail, SBOM evidence. |

Terms: a **project** is **scanned**, the scan yields **findings**, findings are checked against
**policies**, producing a **verdict**: Pass, Warn, Require approval, or Block. **Not evaluated** is not Pass.
Domains: `oss`, `crypto`, `ai` (availability depends on the organisation).

## CLI: install and sign in

```bash
brew install scanoss/dist/earnie
# or
curl -fsSL https://github.com/scanoss/earnie-cli/releases/latest/download/install.sh | sh
earnie version
earnie auth login --api-url https://your-company.earnie.dev                     # interactive
printf %s "$EARNIE_API_KEY" | earnie --api-url https://your-company.earnie.dev auth login --api-key-stdin   # CI
```

Container image: `ghcr.io/scanoss/earnie-cli:X.Y.Z`. Never pass the key as an argument in CI.
Project resolution: `--project`, then `EARNIE_PROJECT`, then the Git remote. There is no default project;
on self-hosted Git remotes set `EARNIE_PROJECT`. Reference:
https://docs.scanoss.com/en/latest/earnie/using-earnie/earnie-cli.md

## Scans and verdicts

```bash
earnie scan path .                      # or: earnie scan staged | earnie scan diff --base origin/main
earnie scan path . --scanners oss,crypto --format json --details
earnie scan path . --no-wait            # returns the scan ID; then:
earnie scan wait <id>  |  earnie scan status <id>  |  earnie verdict <id>  |  earnie scan audit <id>
earnie policy list  |  earnie policy show <slug>  |  earnie policy violations <slug>   # read-only
earnie export --include oss --bom-format cyclonedx --output sbom.json
```

Scan exit codes (gate the build on these directly):

| Exit | Meaning |
|---|---|
| `0` | pass, warn, or none (no policy in force) |
| `1` | block |
| `2` | require approval (returns immediately with the approval link) |
| `3` | scan, evaluation, or gate failed / unknown |
| `4` / `5` | invalid or expired key / missing scope or restricted project |
| `6` / `7` | retries exhausted / local timeout (scan keeps running server-side) |
| `8` | invalid usage or configuration |
| `9` | `earnie mcp doctor` or `setup --check` found a problem |

`--format json` prints one document (`schema_version: "1"`) with the gate, policy revisions, a findings
summary (`--include-findings` for all), and a typed `error` (`code`, `retryable`, `request_id`). Policies
change only in the web app. In a pipeline, use the Earnie CLI action or CI template pinned to a CLI
release ("Running in a Pipeline" in the CLI reference), with the key in a masked secret.

## Pre-commit hook

`earnie hook install` adds a hook running `earnie scan staged --format hook`. It reads the Git index, not
working files. Block stops the commit (exit `1`); Pass, Warn, No policy applied, and Require are allowed.
Technical errors are fail-open by default; `EARNIE_HOOK_FAIL_CLOSED=1` makes them stop your own commits.
The pull-request check stays the authority.

## Connecting a coding agent (MCP)

1. Ask an Admin or Operator for a key with the **Coding agent (MCP)** preset, scoped to the needed projects.
2. `earnie auth login --api-url <url>`, then from the repository: `earnie mcp setup`
   (non-interactive: `--client detected`; preview: `--dry-run`).
3. Restart the client, then `earnie mcp doctor` (exit `0` healthy, `9` problem).

Setup writes the connection (server `earnie` at `<earnie address>/mcp/v1`, header
`Authorization: Bearer <key>`) to the user-level client config, the managed policy block to the repo
instructions file, end-of-turn review hooks, and a session-start hook that refreshes the block.
`earnie mcp uninstall` removes repository changes. Optional header `X-Earnie-Project` sets a default project.
Reference: https://docs.scanoss.com/en/latest/earnie/mcp/connecting-a-client.md

### Working as a connected agent

- Call `earnie_policy_context` once per session (or when the project changes) and follow it.
- After adding or changing a dependency, copying code from elsewhere, or touching cryptography, call
  `earnie_review_code` with the changed `files` or a unified `diff` (not both). Limits: 200 files,
  512 KiB per file, 1 MiB total; larger changes use `earnie scan diff`.
- `status: "running"` with a `selfcheck_id` is not a verdict: call again with only the `selfcheck_id`.
- Read state with `earnie_project_posture`, `earnie_search_findings`, `earnie_get_finding`; export with
  `earnie_export_bom` (existing snapshots only).
- Call `earnie_triage_finding` only when the user told you what to decide on a specific finding, always
  with their reason. It takes effect immediately.
- `EARNIE_SKIP_REVIEW=1` records a visible, user-requested skip. Only set it when the user asks.

Tools and scopes: https://docs.scanoss.com/en/latest/earnie/mcp/tools.md. Errors:
https://docs.scanoss.com/en/latest/earnie/mcp/troubleshooting.md

## Reading a Self-check

A Self-check reviews only the files it was given. It creates no findings and never changes posture; it is
kept 30 days. Coverage is always Partial, so **a Pass means nothing reviewed violated a policy, not that
the change is clear**. Check the **Not evaluated** list (policy whose scanner did not run, missing
enrichment, inputs not present). Status (Running/Completed/Failed) is lifecycle, not outcome.
https://docs.scanoss.com/en/latest/earnie/using-earnie/self-checks.md

## When the merge gate blocks

The Earnie check on a pull request is decided by findings the change introduced. Three routes:
1. **Fix it** (upgrade or replace the component); the next scan re-evaluates.
2. **Correct the finding** in the Review Workspace if it is wrong (e.g. Mark original for your own code).
3. **Request a Policy Approval** when the risk is real and accepted. Only an Admin approves; a pending
   request changes nothing. Agents and API keys can never approve.

A **neutral** check ("Base branch moved" or "No baseline") means not comparable, not pass: push or re-run.
https://docs.scanoss.com/en/latest/earnie/using-earnie/merge-gate.md

## Calling the Earnie API

- Base: the organisation's Earnie address; all paths start with `/v1/`.
- Header: `Authorization: Bearer $EARNIE_API_KEY`. Keys expire (365 days max) and are shown once.

```bash
curl https://your-company.earnie.dev/v1/projects -H "Authorization: Bearer $EARNIE_API_KEY"
```

`GET /v1/projects:resolve` finds a project `id` from its slug or repository. Paging is offset
(`limit`/`offset`, `items`/`total`) or cursor (`cursor`/`next_cursor`). Errors are `{"code","message"}`;
script on `code`. `401` key problem (`unauthenticated`, `invalid_key`, `key_expired`, `key_revoked`),
`403` missing scope or project out of scope (`project_out_of_scope`, `project_scope_required`), `404`
also for items in projects the key cannot reach, `429` after failed auth bursts (honour `Retry-After`).

Choose the smallest preset and one key per consumer:

| Preset | Scopes |
|---|---|
| CI pipeline | `scans:read`, `scans:write`, `findings:read`, `policies:read` |
| Coding agent (MCP) | adds `projects:read`; `findings:write` only with "Allow agents to triage findings" |
| Read-only reporter | `projects:read`, `scans:read`, `findings:read`, `policies:read` |

Other scopes: `organization:read`, `projects:write`, `policies:write`, `scanconfig:read`/`write`,
`integrations:read`, `audit:read`. Never available to a key: approving policy requests, members, SSO,
organisation settings, connecting integrations, managing API keys.
https://docs.scanoss.com/en/latest/earnie/api/using-the-api.md and
https://docs.scanoss.com/en/latest/earnie/administration/api-keys.md

Moving a repo off scanoss-py, the legacy SCANOSS action, or the SCANOSS pre-commit hook? Use the
`migrate-to-earnie` skill.
