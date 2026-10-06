---
name: crypto-finder
description: Find cryptographic algorithms, protocols, certificates, and key material in source code and third-party dependencies with SCANOSS Crypto Finder, and export a CycloneDX 1.7 CBOM (Cryptography Bill of Materials). Use when the user asks to "find the crypto in my code", "build a CBOM", "cryptographic inventory", "post-quantum readiness", "which RSA/ECDSA/MD5/SHA-1 do we use", "scan dependencies for cryptography", runs crypto-finder, or wants to fail CI on cryptography findings.
license: MIT
metadata:
  author: scanoss
  version: "1.0.0"
---

# Crypto Finder

`crypto-finder` detects cryptography in a codebase using SCANOSS-maintained detection rules, and
writes findings as JSON or a CycloneDX 1.7 CBOM. Full reference:
https://docs.scanoss.com/en/latest/cli/crypto-finder/quick-start.md

## Install

Prefer Docker: the standard image bundles the scanning engine.

| Image | Use |
|---|---|
| `ghcr.io/scanoss/crypto-finder:latest` | Recommended. Binary plus scanning engines |
| `ghcr.io/scanoss/crypto-finder:latest-slim` | Binary only; bring your own scanning engine |
| `ghcr.io/scanoss/crypto-finder:latest-deps` | Adds Go, Java (JDK 8/11/17/21, Maven, Gradle), Rust, Python toolchains for `--scan-dependencies` |

Native install (Go 1.25+), which also needs the scanning engine listed in
https://docs.scanoss.com/en/latest/cli/crypto-finder/installation.md :

```bash
go install github.com/scanoss/crypto-finder/cmd/crypto-finder@latest
crypto-finder version
```

Pin an image version tag in CI rather than `latest`.

## Configure

The API key is used to download the SCANOSS rulesets (cached 7 days) and published dependency findings.
Local rules only (`--no-remote-rules --rules-dir <dir>`) need no key.

```bash
export SCANOSS_API_KEY=...                    # CI: inject from the secret store
crypto-finder configure --api-key "<key>"     # workstation: ~/.scanoss/crypto-finder/config.json
crypto-finder configure --api-url https://scanoss.internal.example.com   # or SCANOSS_API_URL
```

Precedence: flags > env vars (`SCANOSS_API_KEY`, `SCANOSS_API_URL`) > config file > project `scanoss.json` > defaults.
The config file is separate from scanoss-cli's. No proxy/CA flags: it honours `HTTP_PROXY`/`HTTPS_PROXY`/`NO_PROXY`
and the OS trust store (`SSL_CERT_FILE` replaces the bundle). Skip paths with `settings.skip.patterns.scanning` in
`scanoss.json`. Reference: https://docs.scanoss.com/en/latest/cli/crypto-finder/configuration.md

## Scan

```bash
crypto-finder scan /path/to/code                                   # findings JSON to stdout
crypto-finder scan --output findings.json /path/to/code
crypto-finder scan --format cyclonedx --output cbom.json /path/to/code
crypto-finder scan --exclude "out/" --error-format json --output findings.json .
crypto-finder scan --rules-dir ./custom-rules /path/to/code         # remote + local rules
crypto-finder scan --no-cache /path/to/code                         # force fresh ruleset download
```

Useful flags: `--languages java,python` (override detection), `--timeout 30m` (default 10m),
`--include-tests`, `--exclude <glob>` (repeatable), `--no-default-exclusions`, `--strict` (fail instead of
using a stale rules cache), `--error-format json` (one JSON error with a stable `code` on stderr),
`-v`/`-q`. Run `crypto-finder scan --help` for the full list.

With Docker, mount the code read-only and an output folder:

```bash
docker run --rm \
  -v "$PWD:/workspace/code:ro" -v "$PWD/out:/workspace/output" -e SCANOSS_API_KEY \
  ghcr.io/scanoss/crypto-finder:latest \
  scan --exclude "out/" --output /workspace/output/findings.json /workspace/code
```

Persist the ruleset cache across runs with a volume on `/root/.scanoss/crypto-finder/cache`.

### Only the files a change touched

```bash
git diff --name-only origin/main...HEAD > changed.txt
crypto-finder scan --detect-paths-from changed.txt --output findings.json .
```

Scopes detection only; the rest of the tree is still read, so reachability stays correct. Do not
emulate this with `--exclude`, which also prunes the call graph. Needs the default scanning engine.

## CBOM output (CycloneDX 1.7)

```bash
crypto-finder convert findings.json --output cbom.json
crypto-finder scan /path/to/code | crypto-finder convert --output cbom.json
```

`convert` validates against the CycloneDX 1.7 schema. Each `cryptographic-asset` component carries
`cryptoProperties` (`assetType`: `algorithm`, `certificate`, `protocol`, `related-crypto-material`) and
`evidence.occurrences`. Only findings with complete metadata reach the CBOM, so it can list fewer assets
than the findings file. Keep the findings JSON for deep analysis.

Findings JSON shape: `.findings[]` with `file_path`, `language`, and `cryptographic_assets[]` (`start_line`,
`match`, `rules[]`, `metadata.assetType`/`algorithmFamily`/`algorithmPrimitive`/`algorithmMode`, `source`:
`direct` or `dependency`, `dependency_info.module`/`version`/`purl`).

```bash
jq -r '[.findings[].cryptographic_assets[] | .metadata.algorithmFamily // "unknown"]
       | group_by(.) | map("\(length)\t\(.[0])")[]' findings.json
```

Reference: https://docs.scanoss.com/en/latest/cli/crypto-finder/output-formats.md

## Dependency scanning

```bash
docker run --rm \
  -v "$PWD:/workspace/code:ro" -v "$PWD/out:/workspace/output" -e SCANOSS_API_KEY \
  ghcr.io/scanoss/crypto-finder:latest-deps \
  scan --scan-dependencies --exclude "out/" --output /workspace/output/findings.json /workspace/code
```

- Ecosystems: Go, Java (Maven/Gradle), Python (pip), Rust (Cargo), Node (npm). `--dep-ecosystem` forces one.
- Needs the toolchains (the `latest-deps` image or local ones). Python projects need their requirements
  installed before the scan.
- Dependency findings have `"source": "dependency"` and `dependency_info`.
- `--export-callgraph callgraph.json` writes reachability ("does my code reach this crypto?").
  Without `--scan-dependencies`, findings report `not_applicable` unless you add
  `--export-callgraph-project-reachability` (leave it off when scanning a library on its own).
- npm dev dependencies are skipped unless `--include-dev-dependencies`.

Reference and limitations (static analysis, no reflection/interface dispatch):
https://docs.scanoss.com/en/latest/cli/crypto-finder/dependency-scanning.md

## Failing CI

| Exit | Meaning |
|---|---|
| `0` | Scan completed |
| `1` | Scan failed, **or** findings were detected and `--fail-on-findings` was set |

`--fail-on-findings` fails on **any** cryptography. Use it only for components that must contain none;
with `--error-format json` its failure code is `findings_detected`, distinct from tool errors such as
`rules_load_failed` or `scanner_timeout`. For an inventory, filter the JSON instead and exit `2` on a
policy hit, for example broken algorithms:

```bash
jq -e '[.findings[].cryptographic_assets[]
        | select((.metadata.algorithmFamily // "") | test("^(MD4|MD5|SHA-?1|DES|3DES|RC4)$"; "i"))]
       | length == 0' findings.json > /dev/null || exit 2
```

The algorithm lists are your policy, not a Crypto Finder feature. The full post-quantum inventory
script is in https://docs.scanoss.com/en/latest/developer-tools/recipes/crypto-inventory-cbom.md
(see also the `scanoss-recipes` skill). Docker and CI examples:
https://docs.scanoss.com/en/latest/cli/crypto-finder/docker-usage.md
