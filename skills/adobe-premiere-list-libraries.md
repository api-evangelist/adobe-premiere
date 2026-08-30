---
name: adobe-premiere-list-libraries
description: Page through a user's Creative Cloud Libraries and read one library's details before touching its contents.
api: Adobe Creative Cloud Libraries API
api_id: adobe-premiere
base_url: https://cc-libraries.adobe.io
spec: openapi/adobe-premiere-cc-libraries-api-openapi.json
generated: '2026-08-30'
method: generated
source: >-
  Grounded in the Adobe-published Creative Cloud Libraries OpenAPI harvested verbatim from
  https://raw.githubusercontent.com/AdobeDocs/cc-libraries-api-spec/main/openapi.json on 2026-08-30.
  Every operationId below is present in that document.
operations:
  - getLibraries
  - getLibrary
---

## Before you start

Every call needs **two** credentials, not one:

- `x-api-key: <Client ID>` — the Client ID of an Adobe Developer Console project that has the
  Creative Cloud Libraries API added. A wrong or missing key returns **403 Forbidden**.
- `authorization: Bearer <IMS access token>` — an Adobe IMS OAuth 2.0 token from
  `https://ims-na1.adobelogin.com/ims/token/v3`. A wrong or expired token returns **401 Unauthorized**.

Scopes: `creative_sdk`, `openid`, `profile`, `AdobeID`.
Optional on every request: `x-request-id` — set it, and the same value comes back as `xactionid` on
any Adobe I/O Event the call triggers, which is the only way to correlate a write with its event.

Errors are plain `application/json` with an `ErrorResponse` body — **not** RFC 9457 problem+json.
There is **no** `Idempotency-Key` header: a repeated POST creates a second object. Use `if-match` /
`if-none-match` for concurrency; a failed precondition returns **412**.

## Steps

1. **List the libraries.** `GET /api/v1/libraries`
   - `limit` is capped at **10** and defaults to 10. `start` is 0-based and is **required whenever
     you send `limit`** — omitting it is a 400.
   - `orderBy` defaults to `-modified_date`. Prefix a vector with `-` to reverse it; comma-separate
     to sort on several (`name,-modified_date`).
   - Read `total_count` from the `LibrariesInfo` envelope and walk `start` forward in steps of
     `limit` until you have `total_count` rows. Do not assume one page is everything.
2. **Pick the library.** Match on `name`, or on `id` if you already hold one. Note `details.etag` —
   you will need it if you later write with `if-match`.
3. **Read the library.** `GET /api/v1/libraries/{libraryId}` returns `DbLibraryInfo`, including
   `elements_count`, `removed_elements_count` (what is sitting in the archive), `storage_used`,
   `ownership`, `collaboration` and the `_links` block.

## Failure modes

- **401** — refresh the IMS token and retry once.
- **403** — the `x-api-key` is not entitled to this API. Do not retry; fix the Console project.
- **404** — the libraryId does not exist for this user, or it belongs to someone else.
- **429** — back off. Adobe publishes no `Retry-After` and no rate-limit headers, so use jittered
  exponential back-off and do not tighten your loop on success.
