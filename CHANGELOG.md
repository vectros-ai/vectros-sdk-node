# Changelog — Vectros SDK

The Vectros SDK is published for **TypeScript** (`@vectros-ai/sdk`), **Python**
(`vectros`), and **Java** (`ai.vectros:vectros-sdk`). All three are generated from
one OpenAPI surface and share this changelog — a release changes the API surface
for every language at once. This file is the canonical source (maintained here
alongside `sdk-version.txt`); the publish pipeline copies it into each language's
package and public mirror.

This project adheres to [Semantic Versioning](https://semver.org).

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
