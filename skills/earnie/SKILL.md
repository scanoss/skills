---
name: earnie
description: Operate Earnie, SCANOSS's governance platform for open source, cryptography, and AI provenance. Install and sign in to the earnie CLI, run scans and read verdicts and exit codes, install the pre-commit hook, connect a coding agent over MCP with earnie mcp setup, run and read Self-checks, understand the merge gate, export SBOMs, and call the Earnie REST API with a Bearer API key and the right scopes. Use when the user mentions Earnie, earnie CLI, "earnie scan", "is this project blocked", "why did the Earnie check fail", "set up Earnie MCP", "earnie mcp doctor", Self-check, Policy Approval, EARNIE_API_KEY, or "call the Earnie API".
license: MIT
metadata:
  author: scanoss
  version: "1.0.0"
---

# Earnie

This skill teaches how to **operate** Earnie. It does not define the user's policies. The repository's
instructions file, for example `AGENTS.md` or `CLAUDE.md`, may contain a block between
`<!-- earnie:policies:start` and `<!-- earnie:policies:end -->`. That managed block is the project's policy,
and it overrides this skill. Never edit the markers or the text between them. Earnie rewrites the block when
policies change, and damaged markers stop the refresh. The docs start at
https://docs.scanoss.com/en/latest/earnie/introduction.md.

## The model: one policy, everywhere code changes

| Where | Covers | What Earnie does |
|---|---|---|
| Development | Coding agents over MCP, the CLI, hooks | Gives the agent policy context and runs quick **Self-checks** on changed files. Creates no findings. |
| Pull requests & CI | Pull/merge requests and CI | Scans the change and posts a gate result with comments. |
| Governance | Web app | Runs full scans and keeps the posture, triage history, and SBOM evidence. |

Earnie scans a **project**, and the scan produces **findings**. Earnie checks the findings against
**policies**, and the result is a **verdict**: Pass, Warn, Require approval, or Block. **Not evaluated** is
not Pass. The domains are `oss`, `crypto`, and `ai`. Which ones are available depends on the organisation.

## CLI: install and sign in

```bash
brew install scanoss/dist/earnie
# or
curl -fsSL https://github.com/scanoss/earnie-cli/releases/latest/download/install.sh | sh
earnie version
earnie auth login --api-url https://your-company.earnie.dev                     # interactive
printf %s "$EARNIE_API_KEY" | earnie --api-url https://your-company.earnie.dev auth login --api-key-stdin   # CI
```

The container image is `ghcr.io/scanoss/earnie-cli:X.Y.Z`. Never pass the key as an argument in CI.
The CLI resolves the project from `--project`, then `EARNIE_PROJECT`, then the Git remote. There is no default
project. On self-hosted Git remotes, set `EARNIE_PROJECT`. See
https://docs.scanoss.com/en/latest/earnie/using-earnie/earnie-cli.md.

## Scans and verdicts

```bash
earnie scan path .                      # or: earnie scan staged | earnie scan diff --base origin/main
earnie scan path . --scanners oss,crypto --format json --details
earnie scan path . --no-wait            # returns the scan ID; then:
earnie scan wait <id>  |  earnie scan status <id>  |  earnie verdict <id>  |  earnie scan audit <id>
earnie policy list  |  earnie policy show <slug>  |  earnie policy violations <slug>   # read-only
earnie export --include oss --bom-format cyclonedx --output sbom.json
```

Gate the build directly on the scan exit codes:

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

`--format json` prints one document (`schema_version: "1"`). It holds the gate, the policy revisions, a
findings summary, and a typed `error` with `code`, `retryable`, and `request_id`. Add `--include-findings` to
get every finding. Policies change only in the web app. In a pipeline, use the Earnie CLI action or CI
template pinned to a CLI release, and keep the key in a masked secret. The CLI reference covers this under
"Running in a Pipeline".

## Pre-commit hook

`earnie hook install` adds a hook that runs `earnie scan staged --format hook`. The hook reads the Git
index, not the working files. Block stops the commit with exit `1`. Pass, Warn, No policy applied, and
Require let the commit through. By default the hook fails open on technical errors. Set
`EARNIE_HOOK_FAIL_CLOSED=1` to make those errors stop your own commits. The pull-request check has the
final say.

## Connecting a coding agent (MCP)

1. Ask an Admin or Operator for a key with the **Coding agent (MCP)** preset, scoped to the needed projects.
2. Run `earnie auth login --api-url <url>`. Then run `earnie mcp setup` from the repository. Add
   `--client detected` to run it non-interactively, or `--dry-run` to preview the changes.
3. Restart the client, then run `earnie mcp doctor`. It exits `0` when healthy and `9` on a problem.

Setup writes four things:

- the connection to the user-level client config, with server `earnie` at `<earnie address>/mcp/v1` and
  header `Authorization: Bearer <key>`;
- the managed policy block to the repository instructions file;
- end-of-turn review hooks;
- a session-start hook that refreshes the block.

`earnie mcp uninstall` removes the repository changes. The optional header `X-Earnie-Project` sets a default
project. See https://docs.scanoss.com/en/latest/earnie/mcp/connecting-a-client.md.

### Working as a connected agent

- Call `earnie_policy_context` once per session, and again when the project changes. Follow what it returns.
- After you add or change a dependency, copy code from elsewhere, or touch cryptography, call
  `earnie_review_code`. Pass either the changed `files` or a unified `diff`, never both. The limits are 200
  files, 512 KiB per file, and 1 MiB in total. Use `earnie scan diff` for larger changes.
- `status: "running"` with a `selfcheck_id` is not a verdict. Call again with only the `selfcheck_id`.
- Read state with `earnie_project_posture`, `earnie_search_findings`, and `earnie_get_finding`. Export with
  `earnie_export_bom`, which returns existing snapshots only.
- Call `earnie_triage_finding` only when the user told you what to decide on a specific finding, always
  with their reason. It takes effect immediately.
- `EARNIE_SKIP_REVIEW=1` records a visible, user-requested skip. Only set it when the user asks.

Tools and scopes are listed at https://docs.scanoss.com/en/latest/earnie/mcp/tools.md. Errors are covered
at https://docs.scanoss.com/en/latest/earnie/mcp/troubleshooting.md.

## Reading a Self-check

A Self-check reviews only the files it receives. It creates no findings and never changes posture. Earnie
keeps it for 30 days. Coverage is always Partial, so **a Pass means nothing reviewed violated a policy, not
that the change is clear**. Check the **Not evaluated** list. It names policies whose scanner did not run,
missing enrichment, and inputs that were not present. The status (Running, Completed, or Failed) tracks the
lifecycle, not the outcome. See https://docs.scanoss.com/en/latest/earnie/using-earnie/self-checks.md.

## When the merge gate blocks

The findings a change introduced decide the Earnie check on its pull request. There are three ways forward:

1. **Fix it.** Upgrade or replace the component, and the next scan re-evaluates.
2. **Correct the finding.** If the finding is wrong, correct it in the Review Workspace. For example, use
   Mark original for your own code.
3. **Request a Policy Approval.** Do this when the risk is real and accepted. Only an Admin approves, and a
   pending request changes nothing. Agents and API keys can never approve.

A **neutral** check ("Base branch moved" or "No baseline") means Earnie could not compare the change. It is
not a pass. Push again or re-run the check. See https://docs.scanoss.com/en/latest/earnie/using-earnie/merge-gate.md.

## Calling the Earnie API

The base URL is the organisation's Earnie address, and all paths start with `/v1/`. Send the header
`Authorization: Bearer $EARNIE_API_KEY`. Keys expire after at most 365 days, and Earnie shows each key once.

```bash
curl https://your-company.earnie.dev/v1/projects -H "Authorization: Bearer $EARNIE_API_KEY"
```

`GET /v1/projects:resolve` finds a project `id` from its slug or repository. Endpoints page by offset
(`limit`/`offset`, `items`/`total`) or by cursor (`cursor`/`next_cursor`). Errors are `{"code","message"}`.
Write scripts against `code`, not the message.

- `401` means a key problem: `unauthenticated`, `invalid_key`, `key_expired`, or `key_revoked`.
- `403` means a missing scope or a project out of scope: `project_out_of_scope` or `project_scope_required`.
- `404` also covers items in projects the key cannot reach.
- `429` follows a burst of failed authentication. Honour `Retry-After`.

Choose the smallest preset and one key per consumer:

| Preset | Scopes |
|---|---|
| CI pipeline | `scans:read`, `scans:write`, `findings:read`, `policies:read` |
| Coding agent (MCP) | adds `projects:read`; `findings:write` only with "Allow agents to triage findings" |
| Read-only reporter | `projects:read`, `scans:read`, `findings:read`, `policies:read` |

The other scopes are `organization:read`, `projects:write`, `policies:write`, `scanconfig:read`/`write`,
`integrations:read`, and `audit:read`. No key can approve policy requests, manage members, SSO, or
organisation settings, connect integrations, or manage API keys. See
https://docs.scanoss.com/en/latest/earnie/api/using-the-api.md and
https://docs.scanoss.com/en/latest/earnie/administration/api-keys.md.

To move a repository off scanoss-py, the legacy SCANOSS action, or the SCANOSS pre-commit hook, use the
`migrate-to-earnie` skill.
