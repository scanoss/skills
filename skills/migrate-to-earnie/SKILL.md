---
name: migrate-to-earnie
description: Migrate a repository from the legacy SCANOSS toolchain to Earnie. Detect the SCANOSS pre-commit hook, scanoss-py scans and .scanoss/results.json, the SCANOSS code-scan GitHub Action (scanoss/gha-code-scan, scanoss/actions-scan, scanoss/code-scan-action), its Policy Check status checks, and scanoss.json, then propose and apply the Earnie equivalents phase by phase and state the known gaps. Use when the user says "migrate to Earnie", "replace scanoss-py with Earnie", "move off the SCANOSS GitHub Action", "switch our SCANOSS pre-commit hook to Earnie", or asks what changes when adopting Earnie.
license: MIT
metadata:
  author: scanoss
  version: "1.0.0"
---

# Migrate to Earnie

The migration guide at https://docs.scanoss.com/en/latest/earnie/migrating/from-scanoss-tools.md is the
source of truth. Fetch it before you act. This skill gives the procedure, and the guide holds the detail.

Follow these rules:

- **There is no side-by-side mode.** Phases 1-3 leave the existing gate in force. Do not remove any legacy
  check, workflow, or hook before phase 4, which switches the gate, or phase 5, which moves developer tooling.
- The user sets up the workspace, connects repositories, triages, writes policies, and configures branch
  protection in the Earnie web app or the Git provider's settings. Tell the user what to do there. Do not
  pretend to do it yourself.
- Do not promise parity. Read the known gaps to the user before phase 1.

## Step 1: Detect the legacy setup

```bash
grep -rn --include='*.y*ml' -E 'scanoss/(gha-code-scan|actions-scan|code-scan-action)' .github/ 2>/dev/null
grep -n -E 'scanoss/pre-commit-hooks|scanoss-check-undeclared-code' .pre-commit-config.yaml 2>/dev/null
grep -rIn -E '\bscanoss-py\b' --exclude-dir=.git . | grep -v '^./node_modules' | head -50
ls scanoss.json .scanoss/ 2>/dev/null; grep -n '\.scanoss' .gitignore 2>/dev/null
grep -rIn -E 'Policy Check: (Copyleft|Undeclared|Dependency Track)' --exclude-dir=.git . | head
grep -rIn -E 'export dt|dependency-?track' --exclude-dir=.git -i . | head
```

Report a short inventory. Say which of these exist, in which files, and what each does today, for example
"the action runs in delta mode on pull requests and fails on any policy violation".

## Step 2: Map each piece

| Found | Earnie equivalent |
|---|---|
| `scanoss-py scan` in a pre-commit hook | Earnie pre-commit hook, which runs `earnie scan staged --format hook` and creates a Self-check |
| Code Compare triage written into `scanoss.json` | Triage in the Review Workspace, which records who decided, when, and why |
| `scanoss.json` skip patterns and BOM entries | Earnie reads them on every scan and never writes them. Scan Configuration adds more settings |
| SCANOSS action running scanoss-py on CI runners | Earnie's repository connection scans every push and PR. A CI step can also run `earnie scan` |
| Checks `Policy Check: Copyleft` / `Undeclared` / `Dependency Track` | One check named **Earnie** |
| `scanoss-py convert` | SBOM snapshots in CycloneDX 1.7 or SPDX 2.3 Lite, and `earnie export` |
| SBOM upload and external policies/exceptions | Earnie policies attached to projects. A Policy Approval is the only exception |

## Step 3: Walk the five phases

Each phase has one exit criterion.

1. **Connect and scan.** The user sets up the workspace and connects the Git provider. Warn the user that
   if the action only ran in delta mode, the first full scan shows a backlog. That is not a regression.
   Exit when every project has a completed full-coverage scan of its default branch and the integrations
   read Healthy.
2. **Bring over identifications.** There is nothing to import, because Earnie reads the committed
   `scanoss.json`.
   - `bom.remove` settles as **Mark original**, `bom.replace` as **Replace component**, and
     `bom.include`/`bom.identify` as **Confirm match**.
   - The `settings.file_snippet` header skipping and ranking threshold apply unless the project sets its own.
   - **Earnie does not apply `bom.ignore` or `bom.exclude`.** Those matches become findings to decide.
   - **Confirm match is not acceptance.** A confirmed GPL component still trips a copyleft policy. Only
     Mark original takes a finding out of the gate. Accepting a real component needs a Policy Approval.

   Exit when no open finding is one the team considers already decided.
3. **Rebuild policies.** This is manual, and there is no importer. Start from the built-in templates. The
   guide has the table. **One copyleft rule becomes two.** Strong copyleft requires approval and weak
   copyleft warns. Enable both, or weak copyleft stops gating without any warning. The legacy action failed
   on every violation state, so mapping a state to Warn loosens the gate. Agree each mapping with the
   policy owner and write it down. Compare with `earnie policy list --project <slug>` and
   `earnie policy violations <policy-slug> --project <slug>`. Exit when Earnie's gate matches the old
   verdicts, or when the team has accepted every difference in writing.
4. **Switch the gate, in one pull request per repository.** Prepare the PR that deletes the SCANOSS action
   workflow and any SBOM upload step. In the same change window, the user goes to branch protection, adds
   **Earnie** as required, and removes the three `Policy Check: *` checks. A required check whose workflow
   is gone waits forever, and keeping both gates every change twice. Check the PR filters for drafts, bots,
   base branches, and paths, because a skipped PR never gets the required check. Exit when one PR has merged
   with Earnie as the only required policy check and no workflow uses the action.
5. **Move developer tooling.** Install the CLI with `brew install scanoss/dist/earnie` or the install
   script, and run `earnie auth login`. Replace the hook in `.pre-commit-config.yaml`:

   ```yaml
   repos:
     - repo: https://github.com/scanoss/earnie-pre-commit
       rev: vX.Y.Z # pin to a release tag
       hooks:
         - id: earnie
   ```

   Then run `pre-commit install --install-hooks` and `pre-commit run earnie`. Without the pre-commit
   framework, run `earnie hook install`. Remove `.scanoss/` from the repository and from `.gitignore`, and
   remove scanoss-py and Code Compare from the developer instructions. For coding agents, you can also run
   `earnie mcp setup`, which the `earnie` skill covers. Exit when a test commit produces a Self-check in
   Earnie.

Apply repository edits as small commits per phase, and show the user each diff.

## Behaviour changes to tell the user

- Results live in Earnie, not in `.scanoss/results.json`. The hook prints a summary and a link.
- By default the hook fails open on technical errors, with a 60-second limit. A project can make it fail
  closed.
- Require approval lets the commit through locally, and the PR check decides.
- By default, Earnie posts inline PR comments only on added lines, only for Medium severity and above, and
  at most ten per review.
- An approval updates the PR check without a push. Earnie does not re-scan.
- The CLI sends source to the Earnie environment. There is no offline mode and no local policy evaluation.
- Earnie sends no email or chat notifications for policy violations.
- To keep feeding an external SBOM portfolio, run `earnie export --bom-format cyclonedx --output sbom.json`
  and add your own upload step.

## Known gaps

Do not promise parity.

- CWE-based policies cannot be rebuilt yet. A policy on CWE fails as **Evaluation failed**.
- There is no equivalent for CPE, SWID tag ID, component hash, or internal-component conditions.
- Package URL conditions do not support regular expressions. Version conditions compare text only.
  "Versions behind" is a single count, not a major/minor/patch distance.
- Policy migration is manual. There is no importer.
- Approvals are per finding. A component matched in many files needs one approval per flagged finding.
- There is no sub-path or dependency-scope setting. Narrow a monorepo project with skip patterns.
- Earnie does not apply `bom.ignore`.
- On some Git providers, enforcing a block depends on the provider plan. Without it, the connection only
  advises with comments and status. Check the guide's note for the user's provider.
