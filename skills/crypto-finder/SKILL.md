---
name: crypto-finder
description: Find cryptographic algorithms, protocols, certificates, and key material in source code and third-party dependencies with SCANOSS Crypto Finder, and export a CycloneDX 1.7 CBOM (Cryptography Bill of Materials). Use when the user asks to "find the crypto in my code", "build a CBOM", "cryptographic inventory", "post-quantum readiness", "which RSA/ECDSA/MD5/SHA-1 do we use", "scan dependencies for cryptography", runs crypto-finder, or wants to fail CI on cryptography findings.
license: MIT
metadata:
  author: scanoss
  version: "1.0.0"
---

# Crypto Finder

`crypto-finder` detects cryptography in a codebase with detection rules that SCANOSS maintains. It
writes findings as JSON or as a CycloneDX 1.7 CBOM. The full reference is at
https://docs.scanoss.com/en/latest/cli/crypto-finder/quick-start.md

## Install

Prefer Docker, because the standard image bundles the scanning engine.

| Image | Use |
|---|---|
| `ghcr.io/scanoss/crypto-finder:latest` | Recommended. Binary plus scanning engines |
| `ghcr.io/scanoss/crypto-finder:latest-slim` | Binary only; bring your own scanning engine |
| `ghcr.io/scanoss/crypto-finder:latest-deps` | Adds Go, Java (JDK 8/11/17/21, Maven, Gradle), Rust, Python toolchains for `--scan-dependencies` |

To install natively, use Go 1.25+. A native install also needs the scanning engine listed in
https://docs.scanoss.com/en/latest/cli/crypto-finder/installation.md.

```bash
go install github.com/scanoss/crypto-finder/cmd/crypto-finder@latest
crypto-finder version
```

Pin an image version tag in CI rather than `latest`.

## Configure

Crypto Finder uses the API key to download the SCANOSS rulesets, which it caches for 7 days, and published
dependency findings. A scan with local rules only (`--no-remote-rules --rules-dir <dir>`) needs no key.

```bash
export SCANOSS_API_KEY=...                    # CI: inject from the secret store
crypto-finder configure --api-key "<key>"     # workstation: ~/.scanoss/crypto-finder/config.json
crypto-finder configure --api-url https://scanoss.internal.example.com   # or SCANOSS_API_URL
```

Flags take precedence over environment variables (`SCANOSS_API_KEY`, `SCANOSS_API_URL`). Environment variables
take precedence over the config file, then the project `scanoss.json`, then the defaults. The config file is
separate from the scanoss-cli config. There are no proxy or CA flags. Crypto Finder honours `HTTP_PROXY`,
`HTTPS_PROXY`, and `NO_PROXY`, and uses the OS trust store. `SSL_CERT_FILE` replaces the bundle. To skip paths,
set `settings.skip.patterns.scanning` in `scanoss.json`. See
https://docs.scanoss.com/en/latest/cli/crypto-finder/configuration.md.

## Scan

```bash
crypto-finder scan /path/to/code                                   # findings JSON to stdout
crypto-finder scan --output findings.json /path/to/code
crypto-finder scan --format cyclonedx --output cbom.json /path/to/code
crypto-finder scan --exclude "out/" --error-format json --output findings.json .
crypto-finder scan --rules-dir ./custom-rules /path/to/code         # remote + local rules
crypto-finder scan --no-cache /path/to/code                         # force fresh ruleset download
```

Other useful flags:

- `--languages java,python` overrides language detection.
- `--timeout 30m` sets the timeout. The default is 10m.
- `--include-tests` scans test files.
- `--exclude <glob>` skips matching paths. You can repeat it.
- `--no-default-exclusions` turns off the built-in exclusions.
- `--strict` fails the scan instead of using a stale rules cache.
- `--error-format json` writes one JSON error with a stable `code` to stderr.
- `-v` makes output verbose and `-q` makes it quiet.

Run `crypto-finder scan --help` for the full list.

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

This limits detection to the listed files. Crypto Finder still reads the rest of the tree, so reachability
stays correct. Do not use `--exclude` for this, because it also prunes the call graph. This option needs the
default scanning engine.

## CBOM output (CycloneDX 1.7)

```bash
crypto-finder convert findings.json --output cbom.json
crypto-finder scan /path/to/code | crypto-finder convert --output cbom.json
```

`convert` validates the output against the CycloneDX 1.7 schema. Each `cryptographic-asset` component has
`cryptoProperties` and `evidence.occurrences`. The `assetType` in `cryptoProperties` is one of `algorithm`,
`certificate`, `protocol`, or `related-crypto-material`. Only findings with complete metadata reach the CBOM,
so it can list fewer assets than the findings file. Keep the findings JSON for detailed analysis.

The findings JSON has a `.findings[]` array. Each entry has `file_path`, `language`, and
`cryptographic_assets[]`. Each asset has `start_line`, `match`, `rules[]`, and
`metadata.assetType`/`algorithmFamily`/`algorithmPrimitive`/`algorithmMode`. Its `source` is `direct` or
`dependency`, and dependency assets add `dependency_info.module`/`version`/`purl`.

```bash
jq -r '[.findings[].cryptographic_assets[] | .metadata.algorithmFamily // "unknown"]
       | group_by(.) | map("\(length)\t\(.[0])")[]' findings.json
```

See https://docs.scanoss.com/en/latest/cli/crypto-finder/output-formats.md.

## Dependency scanning

```bash
docker run --rm \
  -v "$PWD:/workspace/code:ro" -v "$PWD/out:/workspace/output" -e SCANOSS_API_KEY \
  ghcr.io/scanoss/crypto-finder:latest-deps \
  scan --scan-dependencies --exclude "out/" --output /workspace/output/findings.json /workspace/code
```

- Supported ecosystems are Go, Java (Maven/Gradle), Python (pip), Rust (Cargo), and Node (npm).
  `--dep-ecosystem` forces one.
- The scan needs the toolchains, from the `latest-deps` image or installed locally. Install a Python
  project's requirements before the scan.
- Dependency findings have `"source": "dependency"` and `dependency_info`.
- `--export-callgraph callgraph.json` writes reachability, which answers whether your code reaches the crypto.
  Without `--scan-dependencies`, findings report `not_applicable` unless you add
  `--export-callgraph-project-reachability`. Leave that flag off when you scan a library on its own.
- The scan skips npm dev dependencies unless you pass `--include-dev-dependencies`.

The analysis is static and does not follow reflection or interface dispatch. The reference and full list
of limitations are at
https://docs.scanoss.com/en/latest/cli/crypto-finder/dependency-scanning.md

## Failing CI

| Exit | Meaning |
|---|---|
| `0` | Scan completed |
| `1` | Scan failed, **or** findings were detected and `--fail-on-findings` was set |

`--fail-on-findings` fails on **any** cryptography. Use it only for components that must contain none.
With `--error-format json`, its failure code is `findings_detected`, which differs from tool errors such as
`rules_load_failed` or `scanner_timeout`. For an inventory, filter the JSON instead and exit `2` when a
finding breaks your policy. This example fails on broken algorithms:

```bash
jq -e '[.findings[].cryptographic_assets[]
        | select((.metadata.algorithmFamily // "") | test("^(MD4|MD5|SHA-?1|DES|3DES|RC4)$"; "i"))]
       | length == 0' findings.json > /dev/null || exit 2
```

You define the algorithm lists. Crypto Finder does not ship them. The full post-quantum inventory
script is in https://docs.scanoss.com/en/latest/developer-tools/recipes/crypto-inventory-cbom.md, and the
`scanoss-recipes` skill covers it too. Docker and CI examples are in
https://docs.scanoss.com/en/latest/cli/crypto-finder/docker-usage.md.
