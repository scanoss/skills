---
name: scanoss-api
description: Call the public SCANOSS v3 HTTP API directly with curl or your own client. Look up vulnerabilities, licences, cryptography, versions, dependencies, geoprovenance, and copyright for components by PURL (single GET or batch POST), handle status/info_code responses, and run the WFP fingerprint scan flow (synchronous, or chunked upload plus polling). Use when the user wants to "call the SCANOSS API", "query api.scanoss.com", "look up a PURL", "get vulnerabilities/licences for a package via API", build a SCANOSS client or integration, or debug x-api-key, 401, PARTIAL_SUCCESS, or /v3/wfp/scan errors.
license: MIT
metadata:
  author: scanoss
  version: "1.0.0"
---

# SCANOSS v3 API

This is the public HTTP API that scanoss-cli calls. If the user only needs results in a pipeline,
`scanoss-cli` already does all of this, and the `scanoss-cli` skill covers it. The full reference is at
https://docs.scanoss.com/en/latest/developer-tools/api-overview.md.

## Basics

The base URL is `https://api.scanoss.com`, and every endpoint is under `/v3/`. An on-prem deployment
exposes the same paths under its own base URL.

Send the key in the `x-api-key` header on every request. Header names are case-insensitive. A missing or
invalid key returns HTTP `401`. Keys for the hosted API are commercial, so contact sales@scanoss.com for one.
Keep the key in `SCANOSS_API_KEY`. Never hard-code or commit it. See
https://docs.scanoss.com/en/latest/developer-tools/authentication.md.

## Endpoints

| Area | Endpoints |
|---|---|
| Vulnerabilities | `/v3/vulnerabilities/vulnerabilities`, `/v3/vulnerabilities/cpes` |
| Licences | `/v3/licenses`, `/v3/license/attribution`, `/v3/license/evidence` |
| Cryptography | `/v3/cryptography/algorithms`, `/v3/cryptography/algorithms/range`, `/v3/cryptography/hints` |
| Components | `/v3/components/search`, `/v3/components/versions`, `/v3/components/status`, `/v3/components/releases` |
| Dependencies | `/v3/dependencies/dependencies`, `/v3/dependencies/transitive` |
| Geoprovenance | `/v3/geoprovenance/countries`, `/v3/geoprovenance/origin` |
| Copyright | `/v3/copyright/evidence`, `/v3/copyright/holders` |
| Scanning | `/v3/wfp/scan`, `/v3/wfp/scan/{id}` |

Per-endpoint schemas are in the API Reference section of https://docs.scanoss.com/llms.txt.

## Identify components by PURL

The API keys its data by Package URL, such as `pkg:github/scanoss/engine` or `pkg:npm/lodash`, plus an
optional `requirement`. The requirement is a version, or a range on range endpoints. Most lookups accept
two forms:

- `GET` for one component, with `purl` and `requirement` as query parameters.
- `POST` for many components, with a JSON body that holds a non-empty `components` array. This is the batch form.

```bash
# One component
curl -sS "https://api.scanoss.com/v3/components/status" \
  --get --data-urlencode "purl=pkg:github/scanoss/engine" \
  -H "x-api-key: $SCANOSS_API_KEY"

curl -sS "https://api.scanoss.com/v3/licenses" \
  --get --data-urlencode "purl=pkg:github/scanoss/engine" --data-urlencode "requirement=v5.4.5" \
  -H "x-api-key: $SCANOSS_API_KEY"

curl -sS "https://api.scanoss.com/v3/components/versions" \
  --get --data-urlencode "purl=pkg:github/scanoss/engine" --data-urlencode "limit=50" \
  -H "x-api-key: $SCANOSS_API_KEY"

# Batch
curl -sS https://api.scanoss.com/v3/vulnerabilities/vulnerabilities \
  -H "x-api-key: $SCANOSS_API_KEY" -H "Content-Type: application/json" \
  -d '{"components":[{"purl":"pkg:github/madler/zlib","requirement":"1.2.11"},{"purl":"pkg:npm/lodash"}]}'

curl -sS https://api.scanoss.com/v3/cryptography/algorithms \
  -H "x-api-key: $SCANOSS_API_KEY" -H "Content-Type: application/json" \
  -d '{"components":[{"purl":"pkg:github/scanoss/engine","requirement":"5.4.7"}]}'

# Transitive dependencies (one ecosystem per request: composer, crates, gem, maven, npm)
curl -sS https://api.scanoss.com/v3/dependencies/transitive \
  -H "x-api-key: $SCANOSS_API_KEY" -H "Content-Type: application/json" \
  -d '{"depth":2,"limit":50,"components":[{"purl":"pkg:npm/express","requirement":"4.18.2"}]}'
```

Split long lists into chunks and send them in parallel. scanoss-cli uses 10 PURLs per chunk.

On endpoints that support it, mainly `/v3/license/evidence`, a batch body can carry a `search` object. It
takes the same options as the `GET` query parameters:

- `max_results_per_purl`. `0` means no limit.
- `show_related_results`. The default is `false`.
- `on_version_not_found`. The values are `strict` (the default), `show_latest`, `show_closest_lt`,
  `show_closest_gt`, and `show_any`. Range endpoints do not accept it.
- `encoded`. The values are `none`, `url_encoded`, and `base64_encoded`.

## Response conventions

A batch returns HTTP `200` even when some items fail. Read two levels:

- `status.status` is `SUCCESS`, `PARTIAL_SUCCESS`, or `ERROR`. An `ERROR` comes with `status.message`.
- Each item that did not resolve cleanly has an `info_code` and `info_message`. The codes are
  `REQUIREMENT_NOT_MET`, `INVALID_PURL`, `COMPONENT_NOT_FOUND`, `VERSION_NOT_FOUND`, `RESOLVE_FAILED`,
  `RELEASE_NOTES_UNAVAILABLE`, and `WARNING`. `REQUIREMENT_NOT_MET` is informational. The API used the
  nearest version, and the item still has results.

Treat `SUCCESS` and `PARTIAL_SUCCESS` as usable, and decide retries per item. Never retry a whole batch
because one item is `COMPONENT_NOT_FOUND`. A single `GET` derives the same `status.status` from its item.

When the whole request fails, the API returns an HTTP error with
`{"status":"error","error":{"code":"...","message":"..."},"timestamp":"..."}`. Parse `error.code`, not
`message`. The codes are `INVALID_BODY`, `INVALID_QUERY`, `EMPTY_BATCH`, `INVALID_PURL`,
`COMPONENT_NOT_FOUND`, `VERSION_NOT_FOUND`, `TIMEOUT`, and `INTERNAL_ERROR`.

| HTTP status | Meaning |
|---|---|
| `400` | Bad input |
| `401` | Key problem |
| `404` | Not found |
| `410` | Scan session expired |
| `413` | Too large |
| `500` | Server error |
| `504` | Deadline exceeded |

## Scanning: the WFP flow

The API never receives source code, only WFP fingerprints. Generate them with `scanoss-cli wfp` or the Go
SDK. Do not write your own fingerprinter.

```bash
scanoss-cli wfp ./my-project --output project.wfp
```

**Small input, synchronous.** Send no `X-Scan-Id`. The request timeout and upload size limit this form.

```bash
curl -sS -X POST https://api.scanoss.com/v3/wfp/scan \
  -H "x-api-key: $SCANOSS_API_KEY" -H "Content-Type: application/octet-stream" \
  --data-binary @project.wfp
```

**Large input, uploaded in blocks and then polled.**

1. Create the scan ID on the client. It must be a UUID version 7 in canonical lowercase. Any other value
   returns `400 INVALID_SCAN_ID`.
2. Upload each byte range, for example 1 MiB, with the same `X-Scan-Id`, a zero-based inclusive
   `Content-Range: bytes <start>-<end>/<total>`, and `Content-Type: application/octet-stream`.
   Each block returns `202` with `status: uploading` and `received_bytes`. You can send blocks in any
   order or in parallel. Resending an identical block returns `202` and does no harm. The scan starts
   when the full range has arrived.

   ```bash
   curl -sS -X POST https://api.scanoss.com/v3/wfp/scan \
     -H "x-api-key: $SCANOSS_API_KEY" -H "X-Scan-Id: $SCAN_ID" \
     -H "Content-Range: bytes 0-1048575/3145728" \
     -H "Content-Type: application/octet-stream" --data-binary @block-0.bin
   ```
3. Poll for the result. scanoss-cli polls every 2 s.

   ```bash
   curl -sS "https://api.scanoss.com/v3/wfp/scan/$SCAN_ID" -H "x-api-key: $SCANOSS_API_KEY"
   ```

   The `status` moves from `uploading` to `queued`, then to `scanning`, and ends as `completed`, `failed`,
   or `expired`. While scanning, the body also has `phase`, `phase_done`, and `phase_total`. A `failed`
   scan reports an `error` string in the body, not an HTTP error.

The upload errors are `400 MISSING_RANGE`, `400 INVALID_RANGE`, `409 RANGE_CONFLICT`, and
`413 PAYLOAD_TOO_LARGE`. `409 RANGE_CONFLICT` means the block had different bytes, or the upload had
already finished. The polling errors are `404 SCAN_NOT_FOUND` and `410 SCAN_EXPIRED`.

The `result` has two parts:

- `files[]`, where each file has a `path`, a `match_type` of `file`, `snippet`, or `none`, and `matches[]`
  keyed by `url_hash`. Each match has a `confidence` of `LOW`, `MEDIUM`, or `HIGH`. Snippet matches add
  `match_percentage` and line ranges.
- `components`, keyed by `url_hash`, with `component`, `vendor`, `version`, `url`, `release_date`, and
  `purls`.

To add licences or vulnerabilities, send the matched PURLs to the lookup endpoints above.
