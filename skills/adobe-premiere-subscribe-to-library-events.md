---
name: adobe-premiere-subscribe-to-library-events
description: Register an Adobe I/O Events webhook for Creative Cloud Library create/delete/update events and read the CloudEvents payload.
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
  - getLibraryElements
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

1. **Stand up a webhook endpoint that answers the challenge.** Adobe I/O Events sends a challenge to
   your URL before it will deliver anything; the registration stays inactive until you echo it
   correctly. Adobe's reference receiver is
   `https://github.com/adobeio/io-event-sample-webhook`.
2. **Register in the Adobe Developer Console.** Add the Creative Cloud Libraries event provider to
   the same project that holds your Client ID, and point the registration at the public URL from
   step 1. The URL must be internet-reachable — Adobe's own tutorial uses ngrok for local work.
3. **Subscribe to the three event types.**
   - *Creative Cloud Library Created* — including a user adding a public library to "Your Work".
   - *Creative Cloud Library Deleted*.
   - *Creative Cloud Library Updated* — fires for element add/delete/update **and** for library
     metadata changes.
4. **Handle the payload.** It is a CloudEvents envelope: `id`, `source` (the Repository id),
   `specversion`, `type`, `datacontenttype`, `dataschema`, `dataschemaversion`, `time`, `xactionid`,
   `recipient.userid`, `recipient.clientid`, and `data`.
   - The change itself lives in `data.xdmEntity['event:resources']`, keyed by link relation.
   - Each entry carries `event:action` — `created`, `updated`, `deleted` or `none` — and
     `event:embedded`, the embedded JSON of the resource.
   - A Repository Metadata resource is **always** embedded, even when it did not change. Do not read
     its presence as a change.
5. **Resolve what actually changed.** There is no element-level subscription: an *Updated* event
   tells you the library moved, not which element. Re-read
   `GET /api/v1/libraries/{libraryId}/elements` and diff against your own last-known state.
6. **Correlate your own writes.** Send `x-request-id` on every REST call; it comes back as
   `xactionid` on the event, which is how you tell your change apart from the human editor's.

## Failure modes

- Registration never activates — your endpoint did not answer the challenge.
- Duplicate work on *Updated* storms — debounce per `source` (Repository id) and diff, do not act
  per event.
