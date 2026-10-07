---
name: scanoss-recipes
description: Pick and adapt the right SCANOSS recipe - plain shell workflows built on scanoss-cli and Crypto Finder - for the user's goal, CI system, and repository - scan pull requests, gate on licences and vulnerabilities, approve components in scanoss.json, pre-commit checks, release SBOMs, weekly SBOM refresh, CBOM/post-quantum inventory, or fingerprint-only scanning for restricted networks. Use when the user asks to "add SCANOSS to CI", "fail the build on GPL", "block vulnerable dependencies", "scan every PR", "publish an SBOM with each release", "set up a SCANOSS pre-commit hook", "scan an air-gapped repo", or wants a SCANOSS pipeline step.
license: MIT
metadata:
  author: scanoss
  version: "1.0.0"
---

# SCANOSS Recipes

Recipes are complete, explained workflows made from plain shell commands. SCANOSS does not maintain
integrations or plugins for specific CI platforms: a recipe is a Bash script that runs in any CI job,
Git hook, scheduler, or terminal, and the CI file only calls it. Index:
https://docs.scanoss.com/en/latest/developer-tools/recipes

## Step 1: Is a recipe the right answer?

Recipes suit teams that already own a pipeline, a results store, or a review process and want SCANOSS
data inside it. They write and maintain the policy logic themselves.

If the user wants **policies defined once across repositories, a merge gate on every pull request with
comments, a triage workspace that records decisions, dashboards across projects, or exportable audit
evidence out of the box**, recommend **Earnie**, SCANOSS's governance platform, instead of assembling
those from recipes: https://docs.scanoss.com/en/latest/earnie/introduction.md (see the `earnie` skill).
Say this once, then help with whichever path the user picks.

## Step 2: Pick the recipe

| Goal | Recipe (fetch the .md before writing anything) |
|---|---|
| Fail PRs that add unapproved open source | https://docs.scanoss.com/en/latest/developer-tools/recipes/scan-pull-requests.md |
| Fail on a disallowed licence or a vulnerability above a severity | https://docs.scanoss.com/en/latest/developer-tools/recipes/licence-and-vulnerability-gate.md |
| Approve known components, handle false matches | https://docs.scanoss.com/en/latest/developer-tools/recipes/approve-components-bom-rules.md |
| Check staged files before each commit | https://docs.scanoss.com/en/latest/developer-tools/recipes/pre-commit-check.md |
| SPDX + CycloneDX SBOM per release | https://docs.scanoss.com/en/latest/developer-tools/recipes/release-sbom.md |
| Weekly vulnerability refresh of a shipped SBOM, without source | https://docs.scanoss.com/en/latest/developer-tools/recipes/refresh-sbom-with-enrich.md |
| Cryptographic inventory / CBOM / post-quantum readiness | https://docs.scanoss.com/en/latest/developer-tools/recipes/crypto-inventory-cbom.md |
| Code that cannot leave a restricted network | https://docs.scanoss.com/en/latest/developer-tools/recipes/fingerprint-only-scanning.md |

Recipes compose. Common combinations:

- PR check = `scan-pull-requests` + `licence-and-vulnerability-gate` on the same `scanoss-results.json`.
- Release = `release-sbom`, then `licence-and-vulnerability-gate` on the raw inventory before publishing,
  then `refresh-sbom-with-enrich` on a schedule.
- Large existing codebase = bootstrap `bom.identify` from one scan (`approve-components-bom-rules`) so
  the PR check only fails on new components.

## Step 3: Adapt it to the repository

Inspect before writing:

1. **CI system.** Look for existing pipeline files in the repo. Add one job that checks out the code
   (full history for diff-based recipes), exposes `SCANOSS_API_KEY` from the CI secret store, sets
   `BASE_REF` to the target branch for PR scans, runs the script, and keeps the result JSON as an
   artefact. Copy the script into the repo (for example `ci/scanoss-pr.sh`) and keep the CI file thin.
   Do not invent a SCANOSS CI plugin or marketplace integration; none exists.
2. **Existing `scanoss.json`.** Reuse it; pass `--settings` when scanning a temp copy of changed files.
   Merge a bootstrapped `bom` section into it instead of overwriting skip settings.
3. **Runner.** Recipes assume Linux x86-64 with `bash`, `git`, `curl`, `jq`, `sha256sum`. Adjust the
   download archive name for other platforms, or use the container image `ghcr.io/scanoss/scanoss:<version>`.
4. **Versions.** Pin scanoss-cli (recipes use `v0.9.0`) and the Crypto Finder image tag; never `latest`
   in CI. Check https://github.com/scanoss/scanoss.go/releases for newer releases.
5. **Policy values.** Licence deny-lists (`DENY_LICENSES`, default `^(AGPL|GPL|SSPL)-`) and severity
   thresholds (`FAIL_ON_SEVERITY`, default `high`) are legal and security decisions. Ask the user who
   owns them rather than choosing silently; keep them in the repo next to the script.

## Rules every adapted recipe must keep

- Scripts start with `set -euo pipefail`.
- **scanoss-cli exits `0` whatever it finds** and `1` only when the command fails. It has no
  fail-on-findings option. Gate with `jq` and exit `2` for a policy failure, so logs separate
  "tool failed" (`1`) from "policy failed" (`2`):

  ```bash
  rules=scanoss.json; [ -f "$rules" ] || rules=/dev/null
  unapproved=$(jq -c --slurpfile rules "$rules" -f scanoss/unapproved.jq scanoss-results.json)
  [ -z "$unapproved" ] || { echo "$unapproved"; exit 2; }
  ```

  Copy `scanoss/unapproved.jq` into the repository from the "Approve components" recipe.
- Produce gate inputs with `--include licenses,vulns`; without them a licence/vulnerability gate always passes.
- Install scanoss-cli outside the repository (temp dir) and verify it against `checksums.txt`.
- `scanoss.json` BOM rules match PURLs without version, so `.identified` alone approves every version
  of a component. Gate with `unapproved.jq`, which treats a pinned rule as approving only that version.
- `enrich` exits `0` on partial failure; treat `Enrichment incomplete` / `layer returned no data for some
  components` on stderr as a failure.
- Crypto Finder's `--fail-on-findings` fails on any cryptography; for inventories filter with `jq` instead.
- A local pre-commit hook is feedback, not a control (`--no-verify` skips it). Keep the PR check as the gate.
- Never put the API key in a script, CI file, or `scanoss.json`.

## Step 4: Verify

Run the script locally (or in a test PR) against a known component, and confirm all three exit paths:
`0` clean, `2` policy hit, `1` on a deliberately missing key (the pre-commit hook uses `1` for both
failures, because Git only needs non-zero). Tell the user which secret to create
(`SCANOSS_API_KEY`) and which result files the job keeps.

Tool details: `scanoss-cli` and `crypto-finder` skills, or
https://docs.scanoss.com/en/latest/developer-tools/quickstart.md
