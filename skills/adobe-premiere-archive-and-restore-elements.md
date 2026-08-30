---
name: adobe-premiere-archive-and-restore-elements
description: Remove library elements reversibly, restore them from the archive, and — only when certain — purge them permanently.
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
  - getLibraryElements
  - archiveLibraryElement
  - archiveLibraryElements
  - unArchiveLibraryElements
  - deleteLibraryElement
  - deleteLibraryElements
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

## The one thing to understand first

`DELETE` on an **element** does not destroy it. It moves it to the library's **archive**, and that is
reversible. `DELETE` on the **archive** is what actually destroys it, and that is not.

| Intent | Call | Reversible |
|---|---|---|
| Remove one element | `DELETE /api/v1/libraries/{libraryId}/elements/{elementId}` (`archiveLibraryElement`) | yes — restore from archive |
| Remove many elements | `DELETE /api/v1/libraries/{libraryId}/elements` (`archiveLibraryElements`) | yes |
| Put them back | `POST /api/v1/libraries/{libraryId}/archive` (`unArchiveLibraryElements`) | n/a |
| Destroy one archived element | `DELETE /api/v1/libraries/{libraryId}/archive/{elementId}` (`deleteLibraryElement`) | **no** |
| Destroy all archived elements | `DELETE /api/v1/libraries/{libraryId}/archive` (`deleteLibraryElements`) | **no** |

Adobe does **not** publish a retention window for the archive. Treat the reversal as available but
untimed: restore promptly, and never rely on the archive as long-term storage.

## Steps

1. **List what you are about to remove.** `GET /api/v1/libraries/{libraryId}/elements` — page with
   `start`/`limit` and record every `id` and `name`. This list is your undo manifest.
2. **Archive.** Call `archiveLibraryElement` per element, or `archiveLibraryElements` for a batch.
3. **Confirm.** `GET /api/v1/libraries/{libraryId}` — `removed_elements_count` should have gone up by
   what you archived and `elements_count` down by the same.
4. **Restore, if you got it wrong.** `POST /api/v1/libraries/{libraryId}/archive` with the ids from
   step 1.
5. **Purge only on an explicit instruction.** The `/archive` DELETEs are terminal. Never call them to
   "tidy up" and never call them as a retry of step 2.

## Failure modes

- **412** — an `if-match` precondition failed; someone else changed the library. Re-read and redecide.
- **404** on restore — the element was already purged, or never archived.
- **429** — back off; there are no rate-limit headers to pace against.
