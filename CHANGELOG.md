# Changelog — Vectros API & SDKs

The Vectros API and its official SDKs — **TypeScript** (`@vectros-ai/sdk`),
**Python** (`vectros`), and **Java** (`ai.vectros:vectros-sdk`) — share one version
line and this changelog. The SDKs are generated from the OpenAPI surface, so a
release moves the API and all three SDKs together. This file is the canonical
source (maintained alongside `sdk-version.txt`); the publish pipeline copies it
into each SDK package + mirror **and the `vectros-api-spec` repo**.

This project adheres to [Semantic Versioning](https://semver.org).

## 0.34.0 — 2026-07-10

Custom ownership scopes: organize records, documents, and folders under your own
namespaces — teams, projects, departments, anything — and confine tokens, lists,
and search to them, exactly like the built-in org and client scopes.

### Added

- **`scopes` on create — namespaced ownership values.** `POST /v1/records`, `POST /v1/documents`,
  `POST /v1/documents/upload` (file uploads), and `POST /v1/folders` accept a `scopes` array of
  `<namespace>:<value>` entries (e.g.
  `group:eng-team`, `project:apollo`). A namespace is 2–32 characters, starts with a lowercase
  letter, and may contain lowercase letters, digits, `_`, and `-`; `org` and `client` are the
  built-ins, and a handful of reserved words (`user`, `self`, `tenant`, `context`, `scope`) are
  rejected. An item carries up to **two** scope values alongside its owning user. The values you
  select must come from the credential's own identity, and the resulting ownership must still be
  readable under the credential's `data_scope` (400 otherwise). Omitting `scopes` keeps today's
  behavior — the credential's full identity is stamped. An empty array (`[]`) creates a
  **private, user-owned item** with no shared scopes; it requires a user-carrying identity.
- **`scopes` read-back everywhere.** Record, document, and folder responses — and their webhook
  envelopes — now include the canonical `scopes` list. The existing `orgId`/`clientId` response
  fields are unchanged.
- **`scope` filter on lists, search, and RAG.** Record and document lists accept
  `?scope=<namespace>:<value>`; search and RAG requests accept the same value in a `scope` field.
  For the built-ins, `scope=org:<id>` / `scope=client:<id>` are equivalent to the existing
  `orgId`/`clientId` filters (sending both forms with different values is a 400).
- **`scope:<namespace>` keys in `data_scope`.** Scoped tokens, scoped keys, and access profiles
  can now confine a credential by any custom namespace — per-namespace value lists with the same
  matching rules as the built-ins, including the explicit `null` opt-in for scope-less items.
  Access-profile `identityOverrides` accept the same namespaced keys (at most two scope
  namespaces per identity).
- **Record expiry with `expiresAt` (auto-delete).** `POST`/`PUT /v1/records` accept an optional
  `expiresAt` (ISO-8601 UTC timestamp); the record is automatically deleted at (or shortly after)
  that time — removed from search and storage, the same as an explicit delete. It requires the
  record's schema to opt in with `capabilities.ttlEligible: true` (otherwise the request is
  rejected), and must be at least 10 minutes in the future. Omit it to leave a record's expiry
  unchanged; records have no expiry by default. Record responses echo `expiresAt` when set. In this
  release an expiry can be set or extended but not removed once set.
- **`payloadPartial` on list/lookup responses.** When a large record's or document's payload is
  stored externally, a list or lookup response returns only the indexed projection and now sets
  `payloadPartial: true` so you can tell the payload in hand is incomplete (distinct from
  `payloadExternalized`, which is also true on a by-id read that returned the full payload). Fetch
  the full payload with a by-id `GET` or `includePayload=true`, and use `PATCH` (not `PUT`) to
  update such a record without clearing the omitted fields.

### Changed

- **Canonical scope keys on read-back.** `orgId` and `clientId` remain accepted wherever you
  author a `data_scope` or identity, as before — but stored and minted credentials now report
  those dimensions canonically as `scope:org` / `scope:client` (the `userId` key is unchanged).
  Treat the authored form as write-only sugar; read the canonical form.
- **Record `status` is now a lifecycle: `ACTIVE` or `ARCHIVED`.** `POST`/`PUT`/`PATCH /v1/records`
  accept only `ACTIVE` (the default) or `ARCHIVED` for `status` (any other value is a 400).
  `ARCHIVED` retracts a record from search and recall (`POST /v1/search` and RAG) while keeping it
  stored, retrievable by id, findable by structured-field lookup, and returned by `GET /v1/records` —
  set `status` back to `ACTIVE` to re-index and restore. This matches how documents' lifecycle already
  works, and is the recommended way to retire a record from search without deleting it. (Previously
  `status` was a free-form label with no effect on search.)
- **Document search filters are now limited to structured fields.** A document's free-text payload
  values are no longer indexed as exact-match search filters; only schema-declared `filterable` fields
  and short scalar values (≤256 bytes) become `?filters=` targets. Free text remains fully full-text
  searchable (via the document body, and via schema-declared `searchable` fields). This prevents a
  large free-text field from silently breaking a document's indexing.
- **`PUT` is guarded against silently clearing externally-stored fields.** For a large record or
  document whose payload is stored externally, a list/lookup response returns only the indexed
  projection — so a `PUT` built from one would omit (and therefore clear) the stored fields it never
  saw. Such a `PUT` is now rejected with a clear error naming the fields that would be cleared. Send
  those fields explicitly, use `PATCH` (which preserves omitted fields), or pass `?allowClear=true`
  to confirm a full replacement. `PATCH` semantics are unchanged.

### Fixed

- **Re-indexing no longer drops documents whose extracted text was discarded.** A
  `storeText: false` file document that was re-indexed after its text had been discarded
  previously fell out of search results silently; the index now re-extracts from your original
  upload transparently. The retention contract is unchanged — the text remains unretrievable via
  `/text` and `/ask`.
- **Authoring a `data_scope` that names both an alias and its canonical form** (e.g. `orgId`
  and `scope:org` with different values) now returns a clear 400 instead of a server error.
- **A document with too much filterable metadata now fails fast at ingest instead of silently.**
  Previously a document whose filterable fields exceeded the search engine's per-item limit could be
  accepted but never become searchable, with no signal. Ingest now rejects it with a clear 400 naming
  the oversized fields, so you can shorten them, mark them non-filterable, or move the text into the
  document body.
- **A corrected re-ingest now actually re-applies its metadata.** Re-ingesting a document or record
  (or editing a filterable field) previously re-indexed against the values captured at first ingest, so
  a fix to a filter field never took effect; it now re-reads the current values on every re-index.

## 0.33.1 — 2026-07-09

Scope-authoring hygiene: mistaken grants now fail loudly instead of quietly
authorizing nothing.

### Changed

- **Scope `allowed_actions` must use the compact `resource:ops` form.** When you author an access
  profile, role, or scoped token, each `allowed_actions` entry must be `*`, the compact
  `resource:ops[:type]` form (e.g. `records:cru`, `documents:r`, `search:r`, `keys:crd`,
  `profiles:cru`), or the literal `create_own_scoped_key`. Bare verbs (`read`, `write`, `delete`)
  and the older `admin:keys` / `admin:profiles` shorthand are **no longer accepted**: they matched
  no resource at runtime, so a credential authored with them was silently granted nothing. They are
  now rejected when you save, so the mistake surfaces immediately. The `s` letter inside the compact
  form still grants sensitive-field reveal for a record type (e.g. `records:rs:patient`).
- **Scope `data_scope` honors the public `userId` ownership key.** A `data_scope` may key on
  `userId`, `orgId`, or `clientId`. Previously a filter keyed on `userId` was silently ignored — a
  credential confined that way matched no records, including its own; it now scopes correctly to the
  named user's records. An unrecognized ownership key is rejected when you save, instead of being
  stored inert.

## 0.33.0 — 2026-07-05

Control whether a file's extracted text is retained, and rely on retention
behavior that finally matches its documentation.

### Added

- **`storeText` on the file-upload request — choose text retention at upload time.** File uploads
  (`POST /v1/documents/upload`) now accept `storeText`. The default (`true`) retains the text
  extracted from your file: it stays retrievable via `GET /v1/documents/{id}/text` and powers
  `POST /v1/documents/{id}/ask`. Set `false` to discard the extracted text once indexing
  completes — search results and the original-file download are unaffected, but `/text` returns
  404 and `/ask` returns 409 for that document. The choice is fixed at ingest: it cannot be
  changed later, and a re-upload keeps the original choice. Previously the flag existed only on
  document responses and could not be set on uploads at all.

### Changed

- **Text-ingested documents always retain their body, and it is always metered.** `storeText` is
  no longer part of the text-ingest request (`POST /v1/documents`): the ingested body is the
  document, is always retrievable via `/text`, and now always counts toward text-storage usage.
  Requests that still send the field are accepted and the value ignored. Previously a
  `storeText: false` text document stored fully retrievable text without being metered for it.
- **`storeText` is immutable after ingest.** `PATCH /v1/documents/{id}` requests naming
  `storeText` are now rejected with 400 (previously accepted). Retention is decided when the
  document is created.
- **`/ask` availability is now determined by actual text presence.** Documents whose text exists
  can always be interrogated (including text documents created under the old opt-in flag), and
  documents whose extracted text was discarded return an actionable 409. Previously the guard
  keyed on the stored flag and rejected some documents whose text was fully available.

### Fixed

- The `/text` endpoint, `/ask` 409, and document-response `storeText` descriptions now describe
  the implemented behavior exactly, including how documents created before this release report
  the flag.

## 0.32.0 — 2026-07-03

Soft-retract documents, and re-index a file document by re-uploading it.

### Added

- **Document lifecycle `status` — archive and restore without deleting.** Documents now carry a
  caller-settable `status` of `ACTIVE` (the default) or `ARCHIVED`. Setting it to `ARCHIVED`
  soft-retracts a document — it is pulled from search and recall but kept and fully retrievable — and
  setting it back to `ACTIVE` restores it. This is the documents analog of the record lifecycle
  `status`, so you can model a review/retirement workflow (or a reversible delete) without destroying
  content. Requires the `documents:u` scope.
- **File-backed documents re-index when you re-upload them.** Re-initiating a file upload with the same
  `externalId` re-issues a presigned URL to the existing document; uploading new bytes now re-extracts
  the text and re-indexes the document, so search reflects the new content. Previously a re-upload
  replaced the stored file but its searchable content did not change.

### Changed

- **Document responses split lifecycle state from processing state.** The document response `status`
  field now reports the lifecycle state (`ACTIVE`/`ARCHIVED`, above); the extraction/indexing state
  (`PENDING_UPLOAD`, `EXTRACTING`, `PENDING_INDEX`, `INDEXED`, `SKIPPED`, `STORED`, `FAILED`) moves to a
  new `indexStatus` field — matching how records already separate the two. If you read a document's
  `status` expecting `INDEXED`, read `indexStatus` instead.
- **Re-uploading over an existing document requires the `documents:u` scope.** A re-upload overwrites
  (and now re-indexes) the document's body, so it is treated as an update: creating a new document still
  requires `documents:c`, but re-uploading over an existing one requires `documents:u`. A create-only
  credential can still create new documents but can no longer overwrite an existing document's file.

## 0.31.0 — 2026-06-29

Tell a created record from a returned one, and overwrite by `externalId` on purpose.

### Added

- **`created` flag on every create response.** Creating a record, document, schema, user,
  organization, client, app context, folder, role, or access profile now returns a `created`
  boolean: `true` when a new entity was created, `false` when one with the same identity key
  (`externalId`, `typeName`, `contextId`, slug, `roleId`, or `principalId`) already existed and was
  returned. Previously the two outcomes were indistinguishable, so a re-create over changed content
  looked like it succeeded even though the submitted fields were not applied. The HTTP status now
  mirrors the flag (see Changed).
- **`?upsert=true` on every create endpoint — explicit overwrite-by-identity-key.** By default a
  create is still idempotent (an existing entity is returned unchanged). Pass `?upsert=true` to
  instead apply the submitted fields to the existing entity and bump its version — the first-class
  way to sync changed content without a separate fetch-then-update. The immutable identity and
  ownership fields (`externalId`, `schemaId`/`typeName`, owner) are never changed, and a re-applied
  upsert whose content already matches is a no-op (no version bump). Upsert requires the resource's
  **update** scope (e.g. `records:u:<type>`) in addition to its create scope, so a create-only
  credential cannot overwrite through the flag.

### Changed

- **Idempotent create now returns `200 OK`, not `201 Created`.** When a create returns an existing
  entity (same identity key) it now responds `200`; a genuinely new entity still responds `201`.
  This makes created-vs-returned visible at the HTTP layer, consistent across all create endpoints
  (it matches what app-context create already did). Clients that treat any `2xx` as success are
  unaffected; a client that strictly asserted `201` on a re-applied create will now see `200` —
  which is the created-vs-existing signal, not an error. Read the `created` field for a
  transport-independent check.

## 0.30.0 — 2026-06-26

Search by document type, clearer rate-limit signals, and usage-report read metering.

### Added

- **Document-type search facet.** `typeName` on `POST /v1/search` (and the RAG search block)
  now applies to **documents as well as records** — scope a relevance search to a single
  schema type across either content type. On its own it narrows both documents and records to
  that type; combine it with `contentTypes` to narrow within one (for example
  `contentTypes: ["documents"]` with `typeName: "runbook"` searches only runbook documents).
- **Rate-limit response headers.** A `429` (rate-limited) response now carries `Retry-After`
  (seconds until the limit window resets) plus `X-RateLimit-Limit` and `X-RateLimit-Remaining`,
  so clients can pace and retry intelligently instead of blind-retrying. The `429` body now
  carries an `errorCode` of `"RATE_LIMITED"`.
- **Rate-limit documentation.** `429` is now documented across the rate-limited write and
  search operations in the API reference, and a new rate-limits guide covers the per-plan
  limits, what counts toward them, and recommended backoff.
- **Usage report — read metering.** The usage report (`GET /v1/usage`) gains a `reads` section
  (a per-call axis and a data-out/egress axis, each with your plan's allowance and any overage)
  plus `reads` and `dataOut` lines in the credit breakdown.

### Changed

- **`typeName` is no longer record-only.** Previously `typeName` implicitly restricted a search
  to records; it now filters whichever content types are in scope. A search that sent `typeName`
  **without** `contentTypes` and relied on getting records only will now also return matching
  documents — pass `contentTypes: ["records"]` to keep the prior behavior.

### Notes

- **Existing documents need reindexing for the `typeName` facet.** Documents ingested before
  this release do not carry the type index key and will not match a `typeName` filter until they
  are reindexed (re-ingested or updated). New and updated documents pick it up automatically;
  records are unaffected.

## 0.29.9 — 2026-06-25

A record or document with no indexable text now reports a clear `SKIPPED`
index status instead of `FAILED`.

### Added

- **`SKIPPED` index status** — a record (`indexStatus`) or document (`status`)
  saved with no searchable text now resolves to `SKIPPED`: it is stored and
  retrievable but was not added to the search index, and this is not an error.
  Populate a searchable field (or re-ingest with extractable text) to index it.
  Previously such items reported `FAILED`.

## 0.29.8 — 2026-06-24

Documentation — **no API or wire changes**. Each SDK now ships a curated,
hand-maintained README in place of generated or outdated content.

### Changed

- **READMEs** for the TypeScript, Python, and Java SDKs are now consistent,
  customer-focused, and verified against the generated client surface,
  correcting earlier TypeScript examples that used outdated method names
  (hybrid search is `client.search.content(...)`; document ingestion is
  `client.documents.ingestDocument(...)`).

## 0.29.7 — 2026-06-24

Licensing correction — **no API or wire changes**. The SDKs are now released
under **Apache License 2.0**, unified with the rest of the Vectros open-source
artifacts, and every package and source mirror now ships the full `LICENSE`
file. (0.29.6 declared MIT but shipped no `LICENSE` file; that version remains
available under MIT on the package registries.)

### Changed

- **License → Apache-2.0** for the TypeScript (`@vectros-ai/sdk`), Python
  (`vectros`), and Java (`ai.vectros:vectros-sdk`) SDKs. The `LICENSE` file is
  now included in each published package and GitHub source mirror.

## 0.29.6 — 2026-06-21

Initial public release of the Vectros SDK — a typed client for the Vectros
API, available for **TypeScript**, **Python**, and **Java**, with the
same surface and wire contract verified across all three.

### Added

- **Structured records** — create, read, update (full and partial `PATCH`),
  delete, and look records up by indexed field, with range and prefix lookups
  and caller-controlled ordering.
- **Documents & folders** — ingest, organize, retrieve, and look up documents by
  field; folder listing and a filter grammar for search.
- **Schemas** — define and evolve record/document schemas, with version history.
- **Hybrid search & in-perimeter RAG** — vector + keyword search and grounded
  document Q&A over your indexed corpus.
- **Inference** — region-aware model invocation with per-tenant residency
  controls.
- **Identity & access** — manage clients, organizations, and users; mint and
  revoke scoped credentials; define least-privilege access profiles and roles.
- **Consistent response envelopes** — paginated list endpoints return a uniform
  `{ data, nextCursor }` shape with cursor-based paging.
- **Audit & version history** read surfaces for records, documents, and schemas.
