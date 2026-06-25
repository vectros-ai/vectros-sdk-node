# Changelog — Vectros SDK

The Vectros SDK is published for **TypeScript** (`@vectros-ai/sdk`), **Python**
(`vectros`), and **Java** (`ai.vectros:vectros-sdk`). All three are generated from
one OpenAPI surface and share this changelog — a release changes the API surface
for every language at once. This file is the canonical source (maintained here
alongside `sdk-version.txt`); the publish pipeline copies it into each language's
package and public mirror.

This project adheres to [Semantic Versioning](https://semver.org).

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
