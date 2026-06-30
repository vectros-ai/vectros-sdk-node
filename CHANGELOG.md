# Changelog — Vectros API & SDKs

The Vectros API and its official SDKs — **TypeScript** (`@vectros-ai/sdk`),
**Python** (`vectros`), and **Java** (`ai.vectros:vectros-sdk`) — share one version
line and this changelog. The SDKs are generated from the OpenAPI surface, so a
release moves the API and all three SDKs together. This file is the canonical
source (maintained alongside `sdk-version.txt`); the publish pipeline copies it
into each SDK package + mirror **and the `vectros-api-spec` repo**.

This project adheres to [Semantic Versioning](https://semver.org).

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
