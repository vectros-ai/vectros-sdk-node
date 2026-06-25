# Vectros SDK for TypeScript / Node.js

[![npm](https://img.shields.io/npm/v/@vectros-ai/sdk)](https://www.npmjs.com/package/@vectros-ai/sdk)
[![license](https://img.shields.io/npm/l/@vectros-ai/sdk)](https://www.apache.org/licenses/LICENSE-2.0)

The official TypeScript client for the [Vectros API](https://vectros.ai) —
hybrid search, document ingestion, structured records, and grounded inference
for your application.

## Installation

```bash
npm install @vectros-ai/sdk
```

Requires Node.js 18+. Ships with TypeScript types.

## Quick start

```typescript
import { VectrosClient } from "@vectros-ai/sdk";

const client = new VectrosClient({
  environment: "https://api.vectros.ai",
  token: process.env.VECTROS_API_KEY!, // sk_live_... or sk_test_...
});

// Hybrid (keyword + semantic) search over your indexed content
const results = await client.search.content({
  query: "patient intake form diabetes",
});

// Ingest a document — extracted, chunked, and indexed for search + RAG
const doc = await client.documents.ingestDocument({
  title: "Patient Intake Form — Jane Doe",
});

// Write a structured record against one of your schemas
const record = await client.records.createRecord({
  typeName: "intake_form",
  schemaId: "6ba7b810-9dad-11d1-80b4-00c04fd430c8",
  payload: { first_name: "Jane", email: "jane@example.com" },
});
```

## Authentication

The SDK sends whatever credential you pass in the `Authorization: Bearer <token>`
header. Two credential types are accepted:

| Type | Prefix | Lifetime | Use from |
|------|--------|----------|----------|
| **API key** | `sk_live_*` / `sk_test_*` | Long-lived | **Server only** — full tenant access |
| **Scoped token** | `st_*` | Short-lived | **Server or browser** — narrowed scope, auto-expiring |

Put an API key only on your server. For browser code, mint a short-lived scoped
token on your backend and pass it (or a token-supplier function) to the client —
never ship an API key to the browser. The `token` option accepts either a string
or an async supplier, so refresh logic lives in one place:

```typescript
const client = new VectrosClient({
  environment: "https://api.vectros.ai",
  token: async () => getFreshScopedToken(), // your backend mint call
});
```

See the [authentication guide](https://docs.vectros.ai) for the full
browser token-minting pattern.

## What you can do

- **Hybrid search & RAG** — `client.search`, `client.inference` — vector + keyword
  search and grounded document Q&A over your indexed corpus.
- **Documents & folders** — `client.documents`, `client.folders` — ingest,
  organize, retrieve, and look documents up by field.
- **Structured records** — `client.records` — create, read, update (full and
  partial), delete, and look records up by indexed field.
- **Schemas** — `client.schemas` — define and evolve record/document schemas.
- **Identity & access** — `client.identity`, `client.auth` — manage clients,
  organizations, and users; mint and revoke scoped credentials.

## Documentation

- **Guides & reference:** [docs.vectros.ai](https://docs.vectros.ai)
- **Product:** [vectros.ai](https://vectros.ai)

## License

[Apache License 2.0](./LICENSE).
