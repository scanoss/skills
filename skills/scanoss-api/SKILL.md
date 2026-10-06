---
name: scanoss-api
description: Call the public SCANOSS v3 HTTP API directly with curl or your own client - look up vulnerabilities, licences, cryptography, versions, dependencies, geoprovenance, and copyright for components by PURL (single GET or batch POST), handle status/info_code responses, and run the WFP fingerprint scan flow (synchronous, or chunked upload plus polling). Use when the user wants to "call the SCANOSS API", "query api.scanoss.com", "look up a PURL", "get vulnerabilities/licences for a package via API", build a SCANOSS client or integration, or debug x-api-key, 401, PARTIAL_SUCCESS, or /v3/wfp/scan errors.
license: MIT
metadata:
  author: scanoss
  version: "1.0.0"
---

# SCANOSS v3 API

The public HTTP API behind scanoss-cli. If the user only needs results in a pipeline, `scanoss-cli`
already implements all of this (see the `scanoss-cli` skill). Full reference:
https://docs.scanoss.com/en/latest/developer-tools/api-overview.md

## Basics

- Base URL: `https://api.scanoss.com`. Every endpoint is under `/v3/`. An on-prem deployment exposes the
  same paths under its own base URL.
- Auth: send the key in the `x-api-key` header on every request (header names are case-insensitive).
  Missing or invalid key: HTTP `401`. Keys for the hosted API are commercial: sales@scanoss.com.
- Keep the key in `SCANOSS_API_KEY`; never hard-code it or commit it.
- Reference: https://docs.scanoss.com/en/latest/developer-tools/authentication.md

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

Data is keyed by Package URL (`pkg:github/scanoss/engine`, `pkg:npm/lodash`) plus an optional
`requirement` (a version, or a range on range endpoints). Most lookups accept:

- `GET` for one component: `purl` and `requirement` as query parameters.
- `POST` for many: a JSON body with a non-empty `components` array (the batch form).

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

For long lists, split into chunks and send them in parallel (scanoss-cli uses 10 PURLs per chunk).

A batch body may carry a `search` object (same options as `GET` query parameters) on endpoints that
support it, mainly `/v3/license/evidence`: `max_results_per_purl` (0 = no limit), `show_related_results`
(false), `on_version_not_found` (`strict` default, `show_latest`, `show_closest_lt`, `show_closest_gt`,
`show_any`; not for range endpoints), `encoded` (`none`, `url_encoded`, `base64_encoded`).

## Response conventions

A batch returns HTTP `200` even when some items fail. Read two levels:

- `status.status`: `SUCCESS`, `PARTIAL_SUCCESS`, or `ERROR` (with `status.message`).
- Per item `info_code` / `info_message` when it did not resolve cleanly:
  `REQUIREMENT_NOT_MET` (informational, nearest version used, still has results), `INVALID_PURL`,
  `COMPONENT_NOT_FOUND`, `VERSION_NOT_FOUND`, `RESOLVE_FAILED`, `RELEASE_NOTES_UNAVAILABLE`, `WARNING`.

Treat `SUCCESS` and `PARTIAL_SUCCESS` as usable; decide retries per item. Never retry a whole batch
because one item is `COMPONENT_NOT_FOUND`. A single `GET` derives the same `status.status` from its item.

Whole-request failures return an HTTP error with
`{"status":"error","error":{"code":"...","message":"..."},"timestamp":"..."}`. Parse `error.code`
(`INVALID_BODY`, `INVALID_QUERY`, `EMPTY_BATCH`, `INVALID_PURL`, `COMPONENT_NOT_FOUND`, `VERSION_NOT_FOUND`,
`TIMEOUT`, `INTERNAL_ERROR`), not `message`. Statuses: `400` bad input, `401` key, `404` not found,
`410` scan session expired, `413` too large, `500` server, `504` deadline exceeded.

## Scanning: the WFP flow

The API never receives source code, only WFP fingerprints. Generate them with `scanoss-cli wfp` (or the
Go SDK); do not write your own fingerprinter.

```bash
scanoss-cli wfp ./my-project --output project.wfp
```

**Small input, synchronous** (no `X-Scan-Id`; limited by request timeout and upload size):

```bash
curl -sS -X POST https://api.scanoss.com/v3/wfp/scan \
  -H "x-api-key: $SCANOSS_API_KEY" -H "Content-Type: application/octet-stream" \
  --data-binary @project.wfp
```

**Large input, upload in blocks then poll:**

1. Create the scan ID client-side: a UUID version 7, canonical lowercase. Anything else is `400 INVALID_SCAN_ID`.
2. Upload each byte range (e.g. 1 MiB) with the same `X-Scan-Id`, a zero-based inclusive
   `Content-Range: bytes <start>-<end>/<total>`, and `Content-Type: application/octet-stream`.
   Each block returns `202` (`status: uploading`, `received_bytes`). Blocks may go in any order or in
   parallel; resending an identical block is a harmless `202`. The scan starts when the range is complete.

   ```bash
   curl -sS -X POST https://api.scanoss.com/v3/wfp/scan \
     -H "x-api-key: $SCANOSS_API_KEY" -H "X-Scan-Id: $SCAN_ID" \
     -H "Content-Range: bytes 0-1048575/3145728" \
     -H "Content-Type: application/octet-stream" --data-binary @block-0.bin
   ```
3. Poll (scanoss-cli polls every 2 s):

   ```bash
   curl -sS "https://api.scanoss.com/v3/wfp/scan/$SCAN_ID" -H "x-api-key: $SCANOSS_API_KEY"
   ```

   `status`: `uploading` > `queued` > `scanning` (with `phase`, `phase_done`, `phase_total`) > `completed`,
   `failed`, or `expired`. A `failed` scan reports an `error` string in the body, not an HTTP error.

Upload errors: `400 MISSING_RANGE`, `400 INVALID_RANGE`, `409 RANGE_CONFLICT` (different bytes, or a
finished upload), `413 PAYLOAD_TOO_LARGE`. Polling: `404 SCAN_NOT_FOUND`, `410 SCAN_EXPIRED`.

The `result` holds `files[]` (`path`, `match_type`: `file`/`snippet`/`none`, `matches[]` keyed by
`url_hash` with `confidence` `LOW`/`MEDIUM`/`HIGH`; snippets add `match_percentage` and line ranges) and
`components` keyed by `url_hash` (`component`, `vendor`, `version`, `url`, `release_date`, `purls`).
To add licences or vulnerabilities, send the matched PURLs to the lookup endpoints above.
