---
name: adobe-premiere-add-element-to-library
description: Upload an asset and attach it to a Creative Cloud Library as a new element, in the order Adobe requires.
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
  - postLibraryComponent
  - postLibraryElements
  - getLibraryElement
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

1. **Resolve the target library.** `GET /api/v1/libraries` (see `adobe-premiere-list-libraries`) and
   take its `id`.
2. **Upload the asset FIRST.** `POST /api/v1/libraries/{libraryId}/representations/content`
   (`postLibraryComponent`). Adobe's own description is explicit: *when creating a new asset-based
   element (images, thumbnails, video), upload the asset(s) BEFORE creating the element.* Keep the
   whole response — it is what you put in the element's `representations` array.
3. **Create the element.** `POST /api/v1/libraries/{libraryId}/elements` (`postLibraryElements`).
   - Every element needs **at least one** representation, asset-based or literal.
   - Pass the upload responses from step 2 as objects in `representations`.
   - The same operation also copies and moves existing elements between libraries — check which mode
     you intend before sending.
4. **Verify.** `GET /api/v1/libraries/{libraryId}/elements/{elementId}` and confirm `element_type`,
   `representations` and `details.tags` read back the way you sent them.

## Failure modes

- **413 Payload Too Large** — you inlined the asset instead of uploading it in step 2.
- **415 Unsupported Media Type** — wrong content type on the representation upload.
- **400** — an element with no representation, or a malformed `representations` array.
- No idempotency key exists. If step 3 times out, **read before you retry** (step 4 against the
  library's element list) or you will create a duplicate element.
