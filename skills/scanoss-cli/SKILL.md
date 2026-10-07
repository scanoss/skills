---
name: scanoss-cli
description: Scan source code for open source components, licences, and known vulnerabilities with the SCANOSS scanoss-cli, generate or convert SPDX and CycloneDX SBOMs, refresh an inventory with enrich, look up component data by PURL, and approve components with scanoss.json BOM rules. Use when the user mentions scanoss-cli, SCANOSS scanning, "scan my code for open source", "generate an SBOM", "convert SPDX to CycloneDX", "check licences/vulnerabilities with SCANOSS", WFP fingerprints, scanoss.json, or "approve a component".
license: MIT
metadata:
  author: scanoss
  version: "1.0.0"
---

# scanoss-cli

`scanoss-cli` fingerprints source code locally, sends only the fingerprints (WFP) to the SCANOSS v3 API,
and writes an inventory of the open source it matched. The full reference is at
https://docs.scanoss.com/en/latest/cli/scanoss-go/quick-start.md.

It talks to the SCANOSS v3 API only and does not work with OSSKB.org.

## Install

```bash
go install github.com/scanoss/scanoss.go/cmd/scanoss-cli@latest   # Go 1.25+
# or a prebuilt binary from https://github.com/scanoss/scanoss.go/releases (verify with checksums.txt)
# or the container image
docker run --rm -v "$PWD:/src" -e SCANOSS_API_KEY ghcr.io/scanoss/scanoss:latest scan /src > results.json
scanoss-cli --version
```

The image runs as a non-root user. Redirect stdout on the host, or add `--user "$(id -u):$(id -g)"` before
you write `--output /src/...`. Pin a version tag in CI.

## Authentication

The default endpoint `https://api.scanoss.com` needs an API key. Each setting resolves in this order, from highest to lowest precedence:

```
--flag  >  environment variable  >  ~/.scanoss/settings.json  >  built-in default
```

- In CI, set `SCANOSS_API_KEY` from the secret store. Never put the key on the command line.
- On a workstation, run `scanoss-cli config set api-key "<key>"`.
- For a custom or on-prem endpoint, use `--api-url`, `SCANOSS_API_URL`, or `config set api-url`. A custom
  URL may not need a key.
- `scanoss-cli config list` shows each value and its source. The key always prints as `********`.
  `config get api-key` exits `0` if the key is set and `1` if not.
- The CLI honours `HTTP_PROXY`, `HTTPS_PROXY`, and `NO_PROXY`. `--proxy` and `--ca-cert <pem>` override
  them. `--ignore-cert-errors` disables TLS verification. Use it only for testing. The CLI never stores it.

`wfp` and `sbom` never contact the API and need no key.

## Commands

| Command | Purpose |
|---|---|
| `scan <path>` | Fingerprint, upload, poll, write results |
| `scan wfp <file.wfp>` | Scan a pre-generated WFP (no fingerprinting flags, no `deps`) |
| `wfp <path>` | Fingerprints only, byte-reproducible, offline |
| `results <scan-id>` | Resume/poll an interrupted scan (accepts the same rule flags) |
| `sbom <input> -f cyclonedx\|spdx` | Offline conversion between raw, SPDX, CycloneDX |
| `enrich <input> --include ...` | Refresh layers on an existing inventory or SBOM by PURL |
| `dependencies [path]` | `--extract-local` manifests, or `--purl` (+ `--transient`) via API |
| `vulnerabilities`, `licenses`, `cryptography`, `geoprovenance`, `copyright`, `components` | PURL lookups |
| `config set\|get\|list\|unset\|path` | Stored settings |

Add `-v/--verbose` to write debug logs to stderr. Stdout then holds only results.

## Scanning

```bash
scanoss-cli scan . --include licenses,vulns --output results.json
scanoss-cli scan . --include deps,licenses,vulns --format cyclonedx --output sbom.cdx.json
scanoss-cli scan wfp project.wfp --output results.json   # or: results <scan-id> to resume
```

- `--include` takes the layers `deps`, `vulns`, `licenses`, `crypto`, and `geo`. `deps` reads declared
  dependencies from manifests and needs a source tree. Without `licenses,vulns`, the result has no licence
  or vulnerability data.
- `-f/--format` takes `raw`, `cyclonedx`, or `spdx`. `raw` is the default and renders every layer.
  `cyclonedx` writes CycloneDX 1.7 and drops `crypto` and `geo`. `spdx` writes SPDX 2.3 and drops `vulns`,
  `crypto`, and `geo`.
- These flags select files: `--min-size` and `--max-size` in bytes, with no minimum by default,
  `--gitignore` (default true), `--all-extensions`, `--all-folders`, `--all-hidden`, and `-t/--threads`
  (default 10).
- `--skip-headers` (default true) changes the fingerprints. Keep it the same across runs you compare.
- `--save-wfp <file>` keeps the WFP, but only the standalone `wfp` command is byte-reproducible.

## Reading the raw result

The raw result has a `components[]` array and a flat `vulnerabilities[]` array. Each component has `purl`,
`version`, `scope` (`detected` or `declared`), `licenses[].id`, `evidence[].path`/`match_type`, and
`identified`. Each vulnerability has `id`, `severity`, and `purls`.

```bash
jq -r '.components[] | "\(.purl)@\(.version // "?")  \([.licenses[]?.id] | join(","))"' results.json
jq -r '.vulnerabilities[]? | "\(.severity // "unknown")  \(.id)  \(.purls // [] | join(","))"' results.json
```

## Gotcha: the exit code never reflects findings

`scanoss-cli` exits `0` when the command completed, whatever it found. It exits `1` only when the command
failed, for example with no key, an unreachable API, a bad flag, or an unwritable output. There is no
fail-on-findings flag. Gate on the JSON instead, and keep "tool failed" (`1`) apart from "policy failed" (`2`).

To fail on components that `scanoss.json` doesn't approve, use the version-aware filter
`scanoss/unapproved.jq`. It is in the "How it fails the pipeline" section of
https://docs.scanoss.com/en/latest/developer-tools/recipes/approve-components-bom-rules.md. Don't gate on
`.identified` alone, for the reason in the version note below.

For licence deny-lists and severity thresholds, use the recipe at
https://docs.scanoss.com/en/latest/developer-tools/recipes/licence-and-vulnerability-gate.md.

## SBOMs and enrichment

```bash
scanoss-cli sbom results.json --format cyclonedx --output sbom.cdx.json   # offline, no second scan
scanoss-cli sbom bom.spdx.json --format cyclonedx --output bom.cdx.json
scanoss-cli enrich results.json --include vulns,licenses --output refreshed.json
scanoss-cli enrich sbom.spdx.json --include vulns,licenses --format cyclonedx --output refreshed.cdx.json
```

Conversion is best-effort. SPDX 2.3 has no vulnerability model, so the CLI drops vulnerabilities and prints
a warning. `enrich` keeps the input format by default and cannot add `deps`. It exits `0` even when a lookup
service fails, so check stderr for `Enrichment incomplete` or `layer returned no data for some components`.

## PURL lookups

```bash
scanoss-cli vulnerabilities --purl 'pkg:github/scanoss/engine'
scanoss-cli licenses attribution --purl 'pkg:github/scanoss/engine'
scanoss-cli cryptography algorithms-range --purl 'pkg:github/scanoss/engine' --requirement '>5.0.0'
scanoss-cli components versions --purl 'pkg:github/scanoss/engine' --limit 50
scanoss-cli dependencies --purl 'pkg:github/scanoss/engine' --requirement '5.4.7' --transient --depth 5
```

For many PURLs, repeat `--purl` or pass `--input`. The input file holds one `purl[,requirement]` per line,
or `{"components":[...]}`. See https://docs.scanoss.com/en/latest/cli/scanoss-go/decoration-and-enrichment.md.

## scanoss.json (BOM rules and settings)

The CLI looks for `scanoss.json` in the scanned directory, then for `settings.json`. You can also pass
`--settings <path>`. If you scan a temporary copy of changed files, pass the repository's `scanoss.json`
with `--settings`.

```json
{
  "bom": {
    "identify": [{ "purl": "pkg:github/madler/zlib" },
                 { "purl": "pkg:github/nlohmann/json", "path": "third_party/json/" }],
    "ignore":   [{ "purl": "pkg:github/some-mirror/zlib-copy", "path": "third_party/zlib/" }],
    "remove":   [{ "purl": "pkg:github/example/our-old-public-repo" }],
    "replace":  [{ "purl": "pkg:github/someone/lodash-fork", "replace_with": "pkg:npm/lodash@4.17.21" }]
  },
  "settings": { "skip": { "patterns": { "scanning": ["dist/**", "**/*.min.js"] } } }
}
```

- The CLI applies the rules on the client after results return, in this order: `ignore` (alias `exclude`),
  `identify` (alias `include`), `--ranking-threshold`, `remove`, `replace`. `identify` protects its PURLs
  from the later steps.
- **Rules match a PURL without its version.** On its own, `pkg:npm/vue@2.6.14` marks every version
  of `vue` as identified, so `.identified` is not a safe approval check. The `unapproved.jq` filter
  treats a pinned rule as approving only that version. Pin versions when the user wants upgrades
  to need re-approval.
- `path` limits a rule to part of the tree. `dir/` matches everything below it, globs follow the standard
  rules, and any other value must match exactly.
- `ignore` and `identify` match canonical PURLs only. `remove` and `replace` also match aliases.
- `-i/--identify <file>` and `-n/--ignore <file>` add unscoped rules from a component list, a CycloneDX or
  SPDX file, or a raw result. This helps when you set a baseline.
- **`settings.file_snippet` keys `skip_headers`, `skip_headers_limit`, `ranking_threshold` override the
  matching flags.** If a flag seems to have no effect, look for these keys.

See https://docs.scanoss.com/en/latest/cli/scanoss-go/scanning-your-project.md and
https://docs.scanoss.com/en/latest/developer-tools/recipes/approve-components-bom-rules.md.

Other references: [configuration](https://docs.scanoss.com/en/latest/cli/scanoss-go/configuration.md),
[output formats](https://docs.scanoss.com/en/latest/cli/scanoss-go/output-formats.md),
[Docker](https://docs.scanoss.com/en/latest/cli/scanoss-go/using-docker.md). For full workflows such as PR scans,
release SBOMs, and pre-commit, use the `scanoss-recipes` skill.
