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
and writes an inventory of the open source it matched. Full reference:
https://docs.scanoss.com/en/latest/cli/scanoss-go/quick-start.md

It talks to the SCANOSS v3 API only and does not work with OSSKB.org.

## Install

```bash
go install github.com/scanoss/scanoss.go/cmd/scanoss-cli@latest   # Go 1.25+
# or a prebuilt binary from https://github.com/scanoss/scanoss.go/releases (verify with checksums.txt)
# or the container image
docker run --rm -v "$PWD:/src" -e SCANOSS_API_KEY ghcr.io/scanoss/scanoss:latest scan /src > results.json
scanoss-cli --version
```

The image runs as non-root: redirect stdout on the host, or add `--user "$(id -u):$(id -g)"` before
writing `--output /src/...`. Pin a version tag in CI.

## Authentication

The default endpoint `https://api.scanoss.com` needs an API key. Every setting resolves as:

```
--flag  >  environment variable  >  ~/.scanoss/settings.json  >  built-in default
```

- CI: set `SCANOSS_API_KEY` from the secret store. Never put the key on the command line.
- Workstation: `scanoss-cli config set api-key "<key>"`.
- Custom/on-prem endpoint: `--api-url`, `SCANOSS_API_URL`, or `config set api-url`. A custom URL may run keyless.
- `scanoss-cli config list` shows each value and its source; the key always prints as `********`.
  `config get api-key` exits `0` if set, `1` if not.
- Proxy/CA: `HTTP_PROXY`/`HTTPS_PROXY`/`NO_PROXY` are honoured; `--proxy` and `--ca-cert <pem>` override.
  `--ignore-cert-errors` disables TLS verification (testing only, never stored).

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

Add `-v/--verbose` for debug logs on stderr; stdout stays clean for results.

## Scanning

```bash
scanoss-cli scan . --include licenses,vulns --output results.json
scanoss-cli scan . --include deps,licenses,vulns --format cyclonedx --output sbom.cdx.json
scanoss-cli scan wfp project.wfp --output results.json   # or: results <scan-id> to resume
```

- `--include` layers: `deps` (declared deps from manifests; needs a source tree), `vulns`, `licenses`,
  `crypto`, `geo`. Without `licenses,vulns` the result has no licence or vulnerability data.
- `-f/--format`: `raw` (default, renders every layer), `cyclonedx` (CycloneDX 1.7; drops `crypto`/`geo`),
  `spdx` (SPDX 2.3; drops `vulns`/`crypto`/`geo`).
- File selection: `--min-size`/`--max-size` (bytes; no minimum by default), `--gitignore` (default true),
  `--all-extensions`, `--all-folders`, `--all-hidden`, `-t/--threads` (10).
- `--skip-headers` (default true) changes fingerprints; keep it consistent across runs you compare.
- `--save-wfp <file>` keeps the WFP, but only the standalone `wfp` command is byte-reproducible.

## Reading the raw result

`components[]` (each with `purl`, `version`, `scope`: `detected` or `declared`, `licenses[].id`,
`evidence[].path`/`match_type`, `identified`) and a flat `vulnerabilities[]` (`id`, `severity`, `purls`).

```bash
jq -r '.components[] | "\(.purl)@\(.version // "?")  \([.licenses[]?.id] | join(","))"' results.json
jq -r '.vulnerabilities[]? | "\(.severity // "unknown")  \(.id)  \(.purls // [] | join(","))"' results.json
```

## Gotcha: the exit code never reflects findings

`scanoss-cli` exits `0` when the command completed, whatever it found, and `1` only when the command
failed (no key, API unreachable, bad flag, unwritable output). There is no fail-on-findings flag.
Gate on the JSON instead, and keep "tool failed" (`1`) apart from "policy failed" (`2`):

To fail on components that `scanoss.json` doesn't approve, use the version-aware filter
`scanoss/unapproved.jq` from
https://docs.scanoss.com/en/latest/developer-tools/recipes/approve-components-bom-rules.md
(section "How it fails the pipeline"). Don't gate on `.identified` alone: see the version note below.

For licence deny-lists and severity thresholds use the recipe
https://docs.scanoss.com/en/latest/developer-tools/recipes/licence-and-vulnerability-gate.md

## SBOMs and enrichment

```bash
scanoss-cli sbom results.json --format cyclonedx --output sbom.cdx.json   # offline, no second scan
scanoss-cli sbom bom.spdx.json --format cyclonedx --output bom.cdx.json
scanoss-cli enrich results.json --include vulns,licenses --output refreshed.json
scanoss-cli enrich sbom.spdx.json --include vulns,licenses --format cyclonedx --output refreshed.cdx.json
```

Conversion is best-effort: SPDX 2.3 has no vulnerability model, so vulns are dropped with a warning.
`enrich` keeps the input format by default, cannot add `deps`, and exits `0` even when a lookup service
fails: check stderr for `Enrichment incomplete` or `layer returned no data for some components`.

## PURL lookups

```bash
scanoss-cli vulnerabilities --purl 'pkg:github/scanoss/engine'
scanoss-cli licenses attribution --purl 'pkg:github/scanoss/engine'
scanoss-cli cryptography algorithms-range --purl 'pkg:github/scanoss/engine' --requirement '>5.0.0'
scanoss-cli components versions --purl 'pkg:github/scanoss/engine' --limit 50
scanoss-cli dependencies --purl 'pkg:github/scanoss/engine' --requirement '5.4.7' --transient --depth 5
```

Many PURLs: repeat `--purl` or pass `--input` (newline `purl[,requirement]` or `{"components":[...]}`).
Reference: https://docs.scanoss.com/en/latest/cli/scanoss-go/decoration-and-enrichment.md

## scanoss.json (BOM rules and settings)

Auto-detected in the scanned directory (then `settings.json`), or pass `--settings <path>`. Scanning a
temporary copy of changed files? Pass the repository's `scanoss.json` with `--settings`.

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

- Applied client-side after results return, in order: `ignore` (alias `exclude`), `identify` (alias
  `include`), `--ranking-threshold`, `remove`, `replace`. `identify` protects its PURLs from the rest.
- **Rules match a PURL without its version.** On its own, `pkg:npm/vue@2.6.14` marks every version
  of `vue` as identified, so `.identified` is not a safe approval check. The `unapproved.jq` filter
  treats a pinned rule as approving only that version. Pin versions when the user wants upgrades
  to need re-approval.
- `path` scopes a rule: `dir/` matches everything below, globs use standard rules, anything else is exact.
- `ignore`/`identify` match canonical PURLs only; `remove`/`replace` also match aliases.
- `-i/--identify <file>` and `-n/--ignore <file>` add unscoped rules from a component list, CycloneDX,
  SPDX, or raw result (useful for a baseline).
- **`settings.file_snippet` keys `skip_headers`, `skip_headers_limit`, `ranking_threshold` override the
  matching flags.** If a flag seems to have no effect, look for these keys.

Reference: https://docs.scanoss.com/en/latest/cli/scanoss-go/scanning-your-project.md and
https://docs.scanoss.com/en/latest/developer-tools/recipes/approve-components-bom-rules.md

More: [configuration](https://docs.scanoss.com/en/latest/cli/scanoss-go/configuration.md),
[output formats](https://docs.scanoss.com/en/latest/cli/scanoss-go/output-formats.md),
[Docker](https://docs.scanoss.com/en/latest/cli/scanoss-go/using-docker.md). For full workflows (PR scans,
release SBOMs, pre-commit) use the `scanoss-recipes` skill.
