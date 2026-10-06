---
name: migrate-to-earnie
description: Migrate a repository from the legacy SCANOSS toolchain to Earnie - detect the SCANOSS pre-commit hook, scanoss-py scans and .scanoss/results.json, the SCANOSS code-scan GitHub Action (scanoss/gha-code-scan, scanoss/actions-scan, scanoss/code-scan-action), its Policy Check status checks, and scanoss.json, then propose and apply the Earnie equivalents phase by phase and state the known gaps honestly. Use when the user says "migrate to Earnie", "replace scanoss-py with Earnie", "move off the SCANOSS GitHub Action", "switch our SCANOSS pre-commit hook to Earnie", or asks what changes when adopting Earnie.
license: MIT
metadata:
  author: scanoss
  version: "1.0.0"
---

# Migrate to Earnie

Source of truth: https://docs.scanoss.com/en/latest/earnie/migrating/from-scanoss-tools.md. Fetch it
before acting; this skill is the procedure, the guide holds the detail.

Ground rules:
- **No side-by-side mode.** Phases 1-3 leave the existing gate in force. Do not remove any legacy check,
  workflow, or hook before phase 4 (gate) or phase 5 (developer tooling).
- Workspace setup, repository connection, triage, policies, and branch protection happen in the Earnie
  web app or the Git provider's settings. Tell the user what to do there; do not pretend to do it.
- Do not promise parity. Read the gaps section to the user before phase 1.

## Step 1: Detect the legacy setup

```bash
grep -rn --include='*.y*ml' -E 'scanoss/(gha-code-scan|actions-scan|code-scan-action)' .github/ 2>/dev/null
grep -n -E 'scanoss/pre-commit-hooks|scanoss-check-undeclared-code' .pre-commit-config.yaml 2>/dev/null
grep -rIn -E '\bscanoss-py\b' --exclude-dir=.git . | grep -v '^./node_modules' | head -50
ls scanoss.json .scanoss/ 2>/dev/null; grep -n '\.scanoss' .gitignore 2>/dev/null
grep -rIn -E 'Policy Check: (Copyleft|Undeclared|Dependency Track)' --exclude-dir=.git . | head
grep -rIn -E 'export dt|dependency-?track' --exclude-dir=.git -i . | head
```

Report a short inventory: which of these exist, in which files, and what each does today (e.g. "the
action runs in delta mode on pull requests and fails on any policy violation").

## Step 2: Map each piece

| Found | Earnie equivalent |
|---|---|
| `scanoss-py scan` in a pre-commit hook | Earnie pre-commit hook, which runs `earnie scan staged --format hook` and creates a Self-check |
| Code Compare triage written into `scanoss.json` | Triage in the Review Workspace (who, when, why recorded) |
| `scanoss.json` skip patterns and BOM entries | Read by Earnie on every scan (never written); plus Scan Configuration |
| SCANOSS action running scanoss-py on CI runners | Earnie's repository connection scans every push and PR; or `earnie scan` in a CI step |
| Checks `Policy Check: Copyleft` / `Undeclared` / `Dependency Track` | One check named **Earnie** |
| `scanoss-py convert` | SBOM snapshots (CycloneDX 1.7, SPDX 2.3 Lite), `earnie export` |
| SBOM upload and external policies/exceptions | Earnie policies attached to projects, Policy Approvals as the only exception |

## Step 3: Walk the five phases (one exit criterion each)

1. **Connect and scan.** User sets up the workspace and connects the Git provider. Exit: every project
   has a completed full-coverage scan of its default branch; integrations read Healthy. Warn the user that
   the first full scan surfaces a backlog if the action only ran in delta mode. That is not a regression.
2. **Bring over identifications.** Nothing to import: Earnie reads the committed `scanoss.json`.
   - `bom.remove` settles as **Mark original**; `bom.replace` as **Replace component**;
     `bom.include`/`bom.identify` as **Confirm match**.
   - `settings.file_snippet` header skipping and ranking threshold apply unless the project sets its own.
   - **`bom.ignore` / `bom.exclude` are not applied**: those matches become findings to decide.
   - **Confirm match is not acceptance.** A confirmed GPL component still trips a copyleft policy. Only
     Mark original leaves the gate; accepting a real component needs a Policy Approval.
   Exit: no findings the team considers already decided remain open.
3. **Rebuild policies.** Manual; no importer. Start from the built-in templates (guide has the table).
   **One copyleft rule becomes two** (strong: require approval; weak: warn); enable both or the weak
   ones silently stop gating. The legacy action failed on every violation state, so mapping a state to
   Warn loosens the gate: agree each mapping with the policy owner and write it down. Compare with
   `earnie policy list --project <slug>` and `earnie policy violations <policy-slug> --project <slug>`.
   Exit: Earnie's gate matches the old verdicts, or every difference is accepted in writing.
4. **Switch the gate, in one pull request per repository.** The agent prepares the PR that deletes the
   SCANOSS action workflow (and any SBOM upload step). In the same change window the user, in branch
   protection, adds **Earnie** as required and removes the three `Policy Check: *` checks. A required
   check whose workflow is gone waits forever; keeping both gates everything twice. Check PR filters
   (drafts, bots, base branches, paths): a skipped PR never gets the required check.
   Exit: one PR merged with Earnie as the only required policy check; no workflow uses the action.
5. **Move developer tooling.** Install the CLI (`brew install scanoss/dist/earnie` or the install
   script), `earnie auth login`. Replace the hook in `.pre-commit-config.yaml`:

   ```yaml
   repos:
     - repo: https://github.com/scanoss/earnie-pre-commit
       rev: vX.Y.Z # pin to a release tag
       hooks:
         - id: earnie
   ```

   Then `pre-commit install --install-hooks` and `pre-commit run earnie`. Without the framework:
   `earnie hook install`. Remove `.scanoss/` from the repo and `.gitignore`, and remove scanoss-py and
   Code Compare from developer instructions. Optionally `earnie mcp setup` for coding agents (see the
   `earnie` skill). Exit: a test commit produces a Self-check in Earnie.

Apply repository edits as small, reviewable commits per phase, and show the user each diff.

## Behaviour changes to tell the user

- Results live in Earnie, not `.scanoss/results.json`; the hook prints a summary and a link.
- The hook is fail-open on technical errors by default (60-second limit); a project can make it fail-closed.
- Require approval allows the commit locally; the PR check decides.
- Inline PR comments only on added lines, Medium severity and up, at most ten per review by default.
- An approval updates the PR check without a push; nothing is re-scanned.
- The CLI sends source to the Earnie environment: no offline mode, no local policy evaluation.
- No email or chat notifications for policy violations.
- To keep feeding an external SBOM portfolio: `earnie export --bom-format cyclonedx --output sbom.json`
  plus your own upload step.

## Known gaps (do not promise parity)

- CWE-based policies cannot be rebuilt yet; a policy on CWE fails as **Evaluation failed**.
- No equivalent for CPE, SWID tag ID, component hash, or internal-component conditions.
- Package URL conditions have no regular expressions; version conditions are text comparisons only;
  "versions behind" is a single count, not major/minor/patch distance.
- Policy migration is manual; there is no importer.
- Approvals are per finding: a component matched in many files needs one per flagged finding.
- No sub-path or dependency-scope setting; narrow a monorepo project with skip patterns.
- `bom.ignore` is not applied.
- On some Git providers, enforcing a block depends on the provider plan; otherwise the connection is
  advisory (comments and status only). Check the guide's note for the user's provider.
