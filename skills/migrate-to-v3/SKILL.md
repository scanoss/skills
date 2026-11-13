---
name: migrate-to-v3
description: Migrate a repository or CI pipeline from the SCANOSS v2 API and scanoss-py to the v3 API and scanoss-cli. Detect scanoss-py installs and images, the SCANOSS GitHub Action and other scanoss-py integrations, SCANOSS_SCAN_URL and /scan/direct endpoints, api.osskb.org, v2 flags, inspect gates, and scripts that parse scanoss-py JSON, then map each to its scanoss-cli equivalent or to Earnie, apply the changes phase by phase, and state the known gaps. Use when the user says "migrate to v3", "replace scanoss-py with scanoss-cli", "move off /scan/direct", "upgrade from the v2 API", "switch our pipeline to scanoss-cli", or asks what changes when moving to the v3 API.
license: MIT
metadata:
  author: scanoss
  version: "1.0.0"
---

# Migrate to v3

The migration guide at https://docs.scanoss.com/en/latest/developer-tools/migrating/overview.md is the
source of truth, and https://docs.scanoss.com/en/latest/developer-tools/migrating/from-scanoss-py.md holds
the full mapping. Fetch both before you act. This skill gives the procedure, and the guide holds the detail.

Follow these rules:

- **There is no v2 sunset date.** v3 is recommended, and v2 keeps working for now. Do not invent a deadline.
- **This skill covers repositories and CI pipelines.** If the repository points at a self-hosted v2 server,
  say so and send the user to https://docs.scanoss.com/en/latest/developer-tools/migrating/self-hosted.md.
  Server migration is infrastructure work, and this skill does not do it.
- **Some scanoss-py features have no scanoss-cli equivalent, and they will not be added.** For those, say
  "no equivalent; use Earnie" and point to the `migrate-to-earnie` skill and
  https://docs.scanoss.com/en/latest/earnie/introduction.md. Never say a gap is coming soon.
- **SCANOSS integrations stay on scanoss-py and v2, and will not be ported.** This covers the GitHub Action
  (`scanoss/gha-code-scan`), the GitLab, Azure DevOps and Jenkins setups built on scanoss-py images, the
  SCANOSS pre-commit hooks, SBOM Workbench, scanoss.cc, and the scanoss.js SDK. Replace each one with Earnie
  or with a docs recipe that runs scanoss-cli (the `scanoss-recipes` skill). Do not wait for a v3 version.
- Do not promise identical results. Read the behaviour changes and the known gaps to the user before phase 1.

## Step 1: Detect the v2 setup

```bash
grep -rIn -E '\bscanoss-py\b|pip3? install .*\bscanoss\b|ghcr\.io/scanoss/scanoss-py' --exclude-dir=.git --exclude-dir=node_modules . | head -50
grep -rn --include='*.y*ml' -E 'scanoss/(gha-code-scan|actions-scan|code-scan-action)' .github/ 2>/dev/null
grep -rIn -E 'SCANOSS_(SCAN|GRPC)_URL|/scan/direct|api\.osskb\.org|:(4443|5443)\b' --exclude-dir=.git . | head
grep -rIn --exclude-dir=.git -E -- '--apiurl|--key\b|-k [^ ]|--pac\b|--skip-(size|extension|folder|snippets)\b|--stdin|--files\b|--hpsm|--flags\b|--timeout|--retry' . | grep -i scanoss | head
grep -rIn -E 'inspect (copyleft|undeclared|license-summary|component-summary|dt|gitlab)|(-f|--format) (csv|spdxlite|plain|glc-codequality)' --exclude-dir=.git . | head
grep -rIn -E '\b(container-scan|folder-scan|folder-hash|file_count|delta|export dt)\b' --exclude-dir=.git . | grep -i scanoss | head
grep -rIn -E 'results\.json|\.scanoss/' --exclude-dir=.git . | grep -E 'jq|json\.load|JSON\.parse' | head
grep -n -E 'scanoss/pre-commit-hooks|scanoss-check-undeclared-code' .pre-commit-config.yaml 2>/dev/null
grep -rIn -E 'scanoss\.js|"scanoss":|sbom-workbench|scanoss\.cc' --exclude-dir=.git --exclude-dir=node_modules . | head
ls scanoss.json 2>/dev/null
```

Report a short inventory. Say which of these exist, in which files, and what each does today, for example
"the release job runs `scanoss-py scan -f cyclonedx` and a Python script reads the file-keyed results".

Classify each scan URL:

- No URL, `api.osskb.org`, or `api.scanoss.com`: SaaS. Continue with this skill.
- Any other host, often with `/scan/direct` or a port such as `4443` or `5443`: a self-hosted v2 server.
  scanoss-cli speaks only `/v3`, so it cannot use that server until the server moves to v3. Stop for that
  pipeline and point to the self-hosted guide.

## Step 2: Map each piece

| Found | v3 equivalent |
|---|---|
| `pip install scanoss`, `scanoss-py` | `scanoss-cli`: a release binary, `go install github.com/scanoss/scanoss.go/cmd/scanoss-cli@latest`, or the image `ghcr.io/scanoss/scanoss` |
| `ghcr.io/scanoss/scanoss-py` image | `ghcr.io/scanoss/scanoss`. It is distroless with no shell, so run the CLI as the entrypoint, not inside `sh -c` |
| `SCANOSS_SCAN_URL`, `SCANOSS_GRPC_URL` | `SCANOSS_API_URL`, host only. Drop `/scan/direct` and any other path |
| `api.osskb.org` or no key | `https://api.scanoss.com` (the default), which needs `SCANOSS_API_KEY`. scanoss-cli does not work with OSSKB.org |
| `SCANOSS_API_KEY` | Unchanged |
| `--apiurl`, `--key` / `-k` | `--api-url`, `--api-key` |
| `--proxy`, `--ca-cert`, `--ignore-cert-errors` | Unchanged. `SCANOSS_PROXY` and `SCANOSS_CA_CERT` also work |
| `--pac` | No equivalent. Set `--proxy` or `HTTPS_PROXY` |
| `--threads` / `-T` (default 5) | `--threads` / `-t` (default 10) |
| `--skip-size` | `--min-size`, plus `--max-size` if needed |
| `--skip-extension`, `--skip-folder` | `settings.skip.patterns.scanning` in `scanoss.json` |
| `scan --wfp file.wfp` | `scan wfp file.wfp` |
| `wfp`, `scan -D` | `wfp`, `scan --include deps` |
| `convert` | `sbom <input> -f cyclonedx\|spdx` |
| `-f plain` / `cyclonedx` / `spdxlite` | `-f raw` / `cyclonedx` / `spdx` |
| `-f csv`, `glc-codequality` | No equivalent. Build it from `raw` with your own script, or use Earnie |
| `component vulns` / `licenses` / `provenance` / `search` / `versions` | `vulnerabilities` / `licenses` / `geoprovenance` / `components search` / `components versions` |
| `crypto` | `cryptography` |
| `dependencies` | `dependencies --extract-local`, or `--purl` for an API lookup |
| `results <file>` (filter a results file) | No equivalent. scanoss-cli `results <scan-id>` resumes an interrupted scan |
| `inspect copyleft` / `undeclared` / summaries | No equivalent; use Earnie, or the licence gate recipe |
| `inspect dt`, `export dt` (Dependency-Track) | No equivalent; use Earnie |
| `container-scan`, `folder-scan`, `folder-hash`, `delta`, `file_count`, `utils` | No equivalent; use Earnie |
| `--stdin`, `--files`, `--skip-snippets`, `--hpsm`, `--flags`, `--timeout`, `--retry` | No equivalent |
| SCANOSS GitHub Action, scanoss-py CI images, pre-commit hooks | Earnie, or the PR scan, licence gate, and pre-commit recipes |
| SBOM Workbench, scanoss.cc, scanoss.js | Earnie for triage and SBOMs, or scanoss-cli recipes |

`scanoss.json` stays. scanoss-cli reads `bom.identify`/`include`, `bom.ignore`/`exclude`, `bom.remove`,
`bom.replace`, `settings.skip.*`, and `settings.file_snippet.skip_headers`, `skip_headers_limit`, and
`ranking_threshold`.

## Step 3: Apply in phases

Each phase has one exit criterion. Work on one pipeline at a time, and keep the old job until phase 5.

1. **Install.** Add scanoss-cli to the job, or swap the image. Pin a release version, not `latest`. Store
   `SCANOSS_API_KEY` in the CI secret store and never pass it on the command line. Exit when
   `scanoss-cli --version` runs in the job and `scanoss-cli config get api-key` exits `0`.
2. **Environment and flags.** Rename the environment variables and flags from Step 2. Remove the path from
   the URL. Move skip lists into `scanoss.json`. Remove every flag that has no equivalent, and tell the user
   which ones went. Exit when `scanoss-cli scan . --output results.json` completes without errors.
3. **Output parsing.** Rewrite every script that reads scanoss-py JSON. That output was keyed by file path.
   The default `raw` output is grouped by component (schema 2.1), and each component lists per-file
   `evidence`. Fetch the scan output section of the CLI reference at
   https://docs.scanoss.com/en/latest/cli/scanoss-go/quick-start.md before you write the parser. If a
   consumer only needs an SBOM, switch it to `-f cyclonedx` or `-f spdx` instead. Exit when every consumer
   reads the new file and a run produces the same report shape as before.
4. **Gates.** Replace `inspect` gates and Action policy checks. If the team wants central policies, a merge
   gate, and triage, recommend Earnie and hand over to the `migrate-to-earnie` skill. Otherwise adapt the
   PR scan or licence and vulnerability gate recipe with the `scanoss-recipes` skill. scanoss-cli exits `1`
   on any error, so a gate script decides pass or fail itself. Exit when a known-bad change fails the new
   gate and a clean change passes.
5. **Verify and switch.** Run the old and new scans on the same commit, for example on a branch. Compare
   the matched components, not the files. Explain each difference with the behaviour changes below, and
   tune `scanoss.json` or flags where the team wants the old result. Then remove the scanoss-py job, image,
   and environment variables. Exit when the team has accepted every difference and no pipeline calls
   scanoss-py or a v2 URL.

Apply repository edits as small commits per phase, and show the user each diff.

## Behaviour changes to tell the user

- **An API key is required.** scanoss-py fell back to the free OSSKB service without a key. scanoss-cli
  defaults to `https://api.scanoss.com`, which needs `SCANOSS_API_KEY`.
- **Header skipping is on by default.** scanoss-cli drops licence headers, comments, and imports at the top
  of each file before fingerprinting. scanoss-py did not. Matches can differ. Turn it off with
  `--skip-headers=false`. A `skip_headers` value in `scanoss.json` overrides the flag.
- **Components per match are configurable.** Unset, the server's default applies. Set
  `--max-components N`, or `--max-components all` to get every candidate as scanoss-py reported them.
- **More files are skipped by default.** `.gitignore` is honoured (turn off with `--gitignore=false`), and
  model weights such as `.safetensors`, `.gguf`, `.onnx`, `.pt`, and `.bin` are skipped.
- **Dependencies come from local manifest and lockfile parsing**, not from scancode. Where a lockfile
  resolves a manifest, the output carries the dependency graph without dev dependencies.
- **BOM rules and `ranking_threshold` run on the client**, after results come back. `bom.identify` protects
  its components from every filter.
- **Uploads are chunked and asynchronous.** The CLI uploads the fingerprints in blocks, prints a scan ID,
  and polls. If a job is interrupted, `scanoss-cli results <scan-id>` resumes it.
- The default thread count is 10, not 5.

## Known gaps

Do not promise parity. These will not be built. Earnie is the alternative.

- No `inspect`: no copyleft, undeclared, licence summary, or component summary reports, and no policy exit
  codes.
- No container scanning, folder hashing (`folder-scan`, `folder-hash`), or `delta`.
- No Dependency-Track export or inspection.
- No `csv`, `spdxlite`, `plain`, or GitLab code quality output. `spdx` and `cyclonedx` replace the SBOM
  formats.
- No `--stdin`, `--files`, `--skip-snippets`, HPSM, engine `--flags`, `--pac`, `--timeout`, or `--retry`.
- No `file_count` or `utils` commands.
- No v3 versions of the GitHub Action, other CI integrations, pre-commit hooks, SBOM Workbench, scanoss.cc,
  or scanoss.js.
- scanoss-cli cannot scan against a self-hosted v2 server or OSSKB.org.
