# Vectros AI SDK for Node.js

TypeScript SDK for the [Vectros AI Platform](https://vectros.ai) — hybrid search, document ingestion, structured records, and inference for your application.

## Installation

```bash
npm install @vectros-ai/sdk
```

Requires Node.js 18+.

## Quick Start

```typescript
import { VectrosClient } from "@vectros-ai/sdk";

const client = new VectrosClient({
  apiKey: "sk_live_...",
  environment: "https://api.vectros.ai",
});

// Hybrid search across your indexed content
const results = await client.search.search({
  query: "patient anxiety treatment history",
  mode: "HYBRID",
  limit: 10,
});

// Ingest a document
const doc = await client.documents.createDocument({
  title: "Patient Assessment",
  text: "...",
  indexMode: "HYBRID",
});

// Write a structured record
const record = await client.records.createRecord({
  recordType: "intake_form",
  payload: { name: "Jane Doe", dob: "1990-01-01" },
});
```

## Authentication

The SDK accepts two credential types in the same `Authorization: Bearer <token>` header:

| Type | Prefix | Lifetime | Use From |
|------|--------|----------|----------|
| **API key** | `sk_live_*`, `sk_test_*` | Long-lived | **Server only** — exposes full tenant access |
| **Scoped token** | `st_*` | Short-lived (≤24h) | **Server or browser** — narrowed scope, auto-expiring |

Mint scoped tokens on your backend via `POST /v1/auth/token` using your API key, then hand them to whichever runtime needs them. The SDK does not care which type you pass — both go in the same `token` field.

### Server-side usage (API key)

```ts
import { VectrosClient } from "@vectros-ai/sdk";

const client = new VectrosClient({
  environment: "https://api.vectros.ai",
  token: process.env.VECTROS_API_KEY!, // sk_live_... or sk_test_...
});
```

### Browser-side usage (scoped token)

**Never** put an API key in a browser. Instead:

1. Expose a `POST /api/vectros/mint-token` endpoint on **your own backend** that calls Vectros' mint endpoint with your API key and returns the resulting scoped token to the browser.
2. In the browser, pass a token *supplier* (an async function) to the SDK. The SDK calls it before every request, so refresh logic lives in one place.

```ts
import { VectrosClient } from "@vectros-ai/sdk";

const tokenStore = new ScopedTokenStore(async () => {
  const res = await fetch("/api/vectros/mint-token", { credentials: "include" });
  if (!res.ok) throw new Error("Failed to mint Vectros token");
  return res.json() as Promise<{ token: string; expiresAt: number }>;
});

const client = new VectrosClient({
  environment: "https://api.vectros.ai",
  token: () => tokenStore.getValid(),
});

// Use normally — refresh is transparent.
await client.search.query({ query: "hello" });
```

### Reference `ScopedTokenStore`

Drop this into your frontend codebase. It caches the token in memory, refreshes proactively before expiry, and de-dupes concurrent refreshes so a burst of parallel calls only triggers one mint.

```ts
type MintResult = { token: string; expiresAt: number }; // expiresAt = epoch milliseconds

export class ScopedTokenStore {
  private current?: MintResult;
  private inflight?: Promise<string>;
  private readonly refreshBufferMs = 30_000;

  constructor(private readonly mintFn: () => Promise<MintResult>) {}

  async getValid(): Promise<string> {
    if (this.current && Date.now() < this.current.expiresAt - this.refreshBufferMs) {
      return this.current.token;
    }
    if (this.inflight) return this.inflight;

    this.inflight = (async () => {
      try {
        this.current = await this.mintFn();
        return this.current.token;
      } finally {
        this.inflight = undefined;
      }
    })();
    return this.inflight;
  }

  /** Force the next call to re-mint. Use after a 401 to recover from clock skew or revocation. */
  forceRefresh(): void {
    this.current = undefined;
  }
}
```

### Your backend mint endpoint

Pseudocode — adapt to your framework. The point is: your backend is the chokepoint where you validate the end-user's session and decide what scope to grant them before issuing a Vectros token.

```ts
// POST /api/vectros/mint-token
app.post("/api/vectros/mint-token", async (req, res) => {
  const user = await authenticateSession(req); // your own session check
  if (!user) return res.status(401).end();

  const vectrosRes = await fetch("https://api.vectros.ai/v1/auth/token", {
    method: "POST",
    headers: {
      Authorization: `Bearer ${process.env.VECTROS_API_KEY}`,
      "Content-Type": "application/json",
    },
    body: JSON.stringify({
      userId: user.id,
      expiresInSeconds: 3600, // up to 86400 (24h)
      scope: {
        allowedActions: ["records:crud", "schemas:r"],
        identity: { userId: user.id, orgId: user.orgId },
        dataScope: { orgId: [user.orgId] },
      },
    }),
  });

  if (!vectrosRes.ok) return res.status(502).end();

  const { token, expiresAt } = await vectrosRes.json();
  // Vectros returns expiresAt in epoch SECONDS — convert to ms for the browser store.
  res.json({ token, expiresAt: expiresAt * 1000 });
});
```

### Optional: 401 retry interceptor

The proactive refresh buffer handles the common case, but clock skew or mid-flight revocations can still produce a 401. Pass a custom `fetch` to retry once on 401 after forcing a refresh:

```ts
const client = new VectrosClient({
  environment: "https://api.vectros.ai",
  token: () => tokenStore.getValid(),
  fetch: async (input, init) => {
    let res = await fetch(input, init);
    if (res.status === 401) {
      tokenStore.forceRefresh();
      // Rebuild the request with the fresh Authorization header.
      const freshToken = await tokenStore.getValid();
      const headers = new Headers(init?.headers);
      headers.set("Authorization", `Bearer ${freshToken}`);
      res = await fetch(input, { ...init, headers });
    }
    return res;
  },
});
```

### Notes

- **Entity IDs must pre-exist:** Any IDs referenced in the mint request — `userId`, plus anything inside `scope.identity` or `scope.dataScope` (e.g., `orgId`, `clientId`) — must point to real records in your Vectros tenant. The mint endpoint returns `400` for unknown IDs. Provision users, orgs, and clients via the Identity endpoints before minting tokens that reference them.
- **The SDK never mints tokens itself.** Minting requires the API key, which must never reach the browser. Your backend is always in the loop.
- **CORS:** ensure `api.vectros.ai` allows the `Authorization` header from your browser origins.
- **Scope is set at mint time and immutable.** To change a user's permissions, mint a new token with the new scope. The old token remains valid until its `exp`.
- **Revocation latency:** API Gateway caches authorizer results for up to 5 minutes. A revoked credential may continue to work for that window.


## Resources

- [API Reference](https://developers.vectros.ai)
- [Developer Portal](https://developers.vectros.ai)