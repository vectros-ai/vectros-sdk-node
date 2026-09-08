# Changelog — Vectros API & SDKs

The Vectros API and its official SDKs — **TypeScript** (`@vectros-ai/sdk`),
**Python** (`vectros`), and **Java** (`ai.vectros:vectros-sdk`) — share one version
line and this changelog. The SDKs are generated from the OpenAPI surface, so a
release moves the API and all three SDKs together. This file is the canonical
source (maintained alongside `sdk-version.txt`); the publish pipeline copies it
into each SDK package + mirror **and the `vectros-api-spec` repo**.

This project adheres to [Semantic Versioning](https://semver.org).

## 0.43.0 — 2026-09-07

### Added

- **New `POST /v1/scripts/execute` — run a stored script synchronously, as one atomic transaction.**
  Send `{ scriptRef: { name, version | "latest" }, input: { ... } }` and the script you pushed to
  `POST /v1/scripts` runs under your own credential, in the same sandbox and with the same
  `vectros.records` / `vectros.documents` / `vectros.folders` host functions a trigger rule uses. Every
  write it makes is committed as **one transaction after the script finishes** — all or nothing — and
  the response `{ result, execution: { id, durationMs } }` is sent only after the commit, so everything
  in `result` is committed state. The motivating case: create a folder and file documents into it in
  one call instead of several independently-failing round trips. `result` is whatever the script
  returned, opaque to the platform, capped at 256 KB serialised.

  Requires the new **`scripts:x`** scope — the permission to *execute*, separate from `scripts:c` —
  either bare (every script) or qualified as `scripts:x:<name>` (that script, every version). Inside
  the script every read and write is enforced by your data scopes exactly as the equivalent REST call
  would be.

  Your script sees `input.event` (`"execute"`), `input.userId`, and `input.params` (your `input`
  object) — the same grammar a trigger firing uses, so one script can serve both. Budget: 15 s of wall
  clock, a bounded number of statements and host calls, and 100 storage rows per transaction (a folder
  is 3 rows; a plain text document or record 1 row plus one per range-indexed lookup field and one per
  reference) — the error names `total` and `limit`, and nothing is committed. Failures map the trigger
  taxonomy onto HTTP: the script's own thrown message is a `400 SCRIPT_ERROR`, a concurrent
  modification a `409` naming the ids to re-read, a timeout a `504`. The request takes the write-path
  limits once, before the script runs; reads and writes inside it are metered as such, and execution
  time beyond what they include is charged as script execution time (see `GET /v1/usage` below).

- **Optional `Idempotency-Key` header on `POST /v1/scripts/execute`.** A request repeated under the
  same key within 24 hours receives the first attempt's response without executing again; a duplicate
  still in flight is `409 IDEMPOTENCY_IN_PROGRESS`, the same key on a different request
  `422 IDEMPOTENCY_KEY_REUSED`.

  **Exactly two outcomes are replayable**, and this is the part to write your retry loop around: a
  success, and the undetermined outcome below. Every other response — a `400`, a `403`, a `504`
  timeout, a refusal at the concurrency ceiling — records nothing, so the key stays spendable and a
  retry under it **runs the script again**. That is deliberate: those are the outcomes where retrying
  is what you want. If you need a failed attempt to stay failed, use a new key rather than the same
  one. When an execution's outcome cannot be determined (the script did not
  finish inside the budget while its writes may have committed) the response is
  `500 EXECUTION_OUTCOME_UNKNOWN` carrying `execution.id`, and a keyed retry receives that same answer
  — inspect what the script would have written, then use a new key for a new attempt. **With a key
  present the serialised `result` is capped at 16 KB** rather than 256 KB. This is a platform header, available on this
  endpoint only.

- **`GET /v1/usage` now reports script execution time and its charge.** A new `execution` section
  carries `executions`, `totalMillis`, `includedMillis`, `billableMillis`, `credits` and
  `creditsMilli`; `credits.breakdown` gains `scriptExecution` / `scriptExecutionMilli` so the
  per-category breakdown continues to sum to `credits.used`. Script execution bills at **1 credit per
  100 seconds**, and each billable operation your script performs includes execution time at no charge
  — 350 ms for a simple record write, 1000 ms for a document ingest, and 150 ms for a read charged
  as overage (a read inside your plan's allowance earns nothing) — with a 200 ms per-execution
  platform baseline. You are charged only for time beyond what your operations
  included, so a script that does real work and returns promptly pays nothing. An indexed, foldered or
  oversized write earns a different amount because it is priced differently;
  `execution.includedMillis` reports the exact total actually applied against your execution time —
  including the full duration of any execution we do not charge for at all — so the arithmetic stays
  checkable without reconstructing it. Firings refused before your script runs — rate limits, credit
  ceiling, concurrency limits, a missing script — are never charged and never metered. The section
  covers both trigger firings and synchronous `POST /v1/scripts/execute` calls.

- **New `/v1/scripts` endpoints** for storing versioned script source. `POST /v1/scripts` pushes a
  new, immutable version of a named script (`scriptVersion` auto-increments per `name`, starting at
  1); `GET /v1/scripts/{id}` retrieves one version by its id; `GET /v1/scripts` lists your scripts,
  optionally filtered by `?name=` to list every version of one script; `DELETE /v1/scripts/{id}`
  removes one version. Requires the new `scripts:c`/`scripts:r`/`scripts:d` scopes — there is no `u`
  scope, because a script version is immutable once pushed. Your SDK will still show an
  `updateScript` method for `PUT /v1/scripts/{id}`: that route exists only to refuse, and returns
  `400` for every caller regardless of scope. Push a new version instead.

  `source` is capped at 300,000 UTF-8 bytes and `declaredInputContract` at 50,000. One caveat worth
  stating rather than discovering: the request-size limit is applied to the encoded request body,
  while `source`'s cap counts UTF-8 bytes — so a script that is mostly non-ASCII, sent by a client
  that escapes non-ASCII characters, can be refused below its own documented byte cap. Ordinary ASCII
  source has ample headroom.

  **Deliberately narrow access:** scripts carry no ownership dimension, so an ordinary role grant with
  a bare, non-empty `data_scope` clause (the routine shape of a blueprint-issued role) does not reach
  scripts at all — reaching them requires a clause with no `data_scope`, the explicit null-sentinel
  opt-in in the clause's value list, or a root credential. This is intentional: script authoring is a
  design-time, admin-shaped surface, not one meant for ordinary confined partner roles.

- **New `/v1/triggers` endpoints** for declaring async trigger rules, **and trigger rules now fire.**
  `POST /v1/triggers` declares a new rule ("when a schema record fires this event, invoke this script
  version under this grant"), or with `?upsert=true` reconciles an existing one of the same name to
  the submitted shape; `GET /v1/triggers/{id}` retrieves one by id; `GET /v1/triggers` lists your
  rules; `PUT /v1/triggers/{id}` reconciles an existing rule (unlike scripts, a trigger rule is
  upsert-based, not immutable-per-version — a blueprint re-apply updates it in place);
  `DELETE /v1/triggers/{id}` removes one. Requires the new `triggers:c`/`r`/`u`/`d` scopes.

  A record write on a triggers-enabled schema dispatches the rule's script, which runs asynchronously
  in a sandbox under the rule's own grant — so a rule's `scopes`/`roleIds` are live authority, not a
  declaration awaiting a future release.

  A rule's `scriptRef.version` is a pinned decimal version or the literal `"latest"` (resolved at fire
  time); its grant is XOR — either `roleIds` (role composition) or an inline
  `scopes`/`dataScope`/`capabilities` clause — mirroring the shape an access profile already uses. An
  optional `manifest` narrows which host-object verbs (e.g. `records.update`) the triggered script may
  call, and **is enforced at execution**: a verb outside it is refused even where the grant would
  permit it. A trigger's optional `principalId` (the identity the script executes under) must resolve
  to an existing user of `type: "SERVICE"` with an AccessProfile already provisioned in the same
  tenant and context — rejected with a `400` otherwise. `principalId` is immutable once set (`400` on an attempt
  to change it), alongside `name` and `provisionedBy`; to run a trigger as a different service
  principal, delete it and re-create it.

  A trigger also carries an optional `provisionedBy`: the name of the blueprint that manages it.
  `vectros bootstrap --blueprint <file>` sets this, and deletes only the triggers carrying its own blueprint's name when you
  remove one from that blueprint — so a trigger belonging to another blueprint, or created directly
  through this API, is never removed by someone else's apply. Leave it unset for a trigger you manage
  yourself.

  Because a rule's grant is executed, declaring or reconciling one is subject to the same
  scope-monotonicity rule every other authority-granting surface applies: a scoped credential cannot
  declare a trigger whose grant exceeds its own, on `POST` or on `PUT`, and an unresolvable `roleIds`
  entry is rejected with a `400` rather than stored. A root API key is unaffected. Same deliberately
  narrow, ownerless-resource access posture as `/v1/scripts` above.

  Three further consequences to plan for:

  - **`${{ self.* }}`, `${{ under.self.* }}` and `${{ member.* }}` are rejected (`400`) in a trigger's
    grant** — inline or via a composed role. They resolve against the trigger principal's identity,
    not the author's, so the reach they describe is not bounded by the author's own and the platform
    cannot admit them here. They remain valid everywhere else they were before (roles, access
    profiles, blueprint fragments). Use `${{ input.* }}` to confine a trigger's grant to the record
    that fired it, or name literal values.
  - **A trigger firing that fails for a retryable reason is redelivered** rather than dropped. A
    transient condition — a platform write freeze, a rate limit, a concurrency limit — no longer loses
    the firing permanently. Deterministic failures (an unresolvable script, a manifest violation, an
    authorization denial) are still consumed, since redelivering them cannot change the answer.
  - **`vectros bootstrap --blueprint` can now fail where it previously succeeded**: if two blueprints declare a
    trigger of the same name in one app context, the second apply is rejected with a `400` instead of
    silently taking the row over. Rename one of them, or move it to its own context.

- **A trigger script receives the record that fired it — and a rule declares which fields, in a new
  required `fields` list.** What a trigger script sees as `input`: `event` (`CREATE`, `UPDATE` or
  `DELETE`), `schemaId`, `recordId` (the id `vectros.records.get` accepts), `userId` and
  `scope.<namespace>` (the record's stamped ownership — the values the rule's own `${{ input.* }}`
  grant resolved against), **`record`** (the rule's declared `fields` from the row as written — or,
  on `DELETE`, from the deleted row, which is the only way a script can see anything of it) and, on
  `UPDATE` only, **`previous`** (the same fields from the row before the write, so a script can act on
  what actually changed). A field the record does not carry is absent (`undefined`), never null.

  `fields` is **required** on `POST /v1/triggers`: a rule states what it needs, and `[]` projects
  nothing beyond the record's identity. The generated SDKs type it as optional, because the same body
  serves `PUT /v1/triggers/{id}` where omitting it means "unchanged" — so omitting it on a create is a
  runtime `400 "fields is required"`, not a compile error. Every entry must name a field the schema keeps **inline** —
  declared `inline: true` (new, below), `filterable`, or as a lookup field — and not `sensitive`;
  anything else is a `400` listing the projectable fields. This is a bound by construction rather than
  a runtime surprise: a record's payload moves out of the row above 4 KB (or always, on a
  `LARGE_PAYLOAD` schema), and only the schema's inline fields ride the change event a trigger fires
  from — so a field that is not inline could reach a script for small records and silently vanish for
  large ones. Restricting `fields` to the inline set makes the projection identical whatever the row's
  size. **A rule that declares `fields` must be able to read every record it fires on**: its grant
  must hold a `records:r` clause covering the schema's type — a grant with no read at all is rejected
  with a `400` — and that clause may not carry a literal `data_scope` constraint (a
  `${{ input.userId }}` / `${{ input.scope.<namespace> }}` placeholder is fine — it is the firing
  record's own value) or the rule is rejected with a `400`; and if at execution the rule's grant cannot
  read the firing record (a composed role narrowed after declaration, say), `record` and `previous`
  are delivered as empty objects — the image never discloses what the script's own read would be
  refused. On `PUT /v1/triggers/{id}`, omitting `fields` leaves the stored declaration unchanged, like
  `manifest`. Together, `record` and `previous` are limited to **224 KB serialised**; a firing whose
  projected input exceeds that is not run and is recorded as a new **`INPUT_TOO_LARGE`** failure
  category naming the size and the limit — narrow the rule's `fields` to what the script reads.

  One thing to plan for: a record last written before its schema inlined a field, or before the rule
  was declared, may not yet carry that field on its row, so `previous` on its first `UPDATE` (or
  `record` on its `DELETE`) can lack it until the record is written again; `record` on `CREATE`/`UPDATE`
  is always complete.

- **Schema fields accept a new `inline` flag.** `inline: true` keeps the field on the record row when
  the payload is stored out of line: it then appears in list and lookup projections without
  `includePayload`, and is projectable into a trigger's `input.record`. It cannot be combined with
  `sensitive`, and `PUT /v1/schemas/{id}` is rejected with a `409` if it would stop keeping a field
  inline (by dropping `inline`, `filterable` or the lookup that carried it) while a trigger rule still
  projects that field — narrow the rule's `fields` first, matching the endpoint's existing refusal to
  turn `triggersEnabled` off under live rules.

- **Schemas opt in to firing triggers, via `capabilities.triggersEnabled`.** Set it to `true` on the
  schema named by a rule's `firingSource.schemaId` before declaring the rule, or the create is
  rejected with a `400`; records of a schema fire triggers only when that schema opts in. It defaults
  to `false` and is independent of `capabilities.auditHistory` — turning audit history off does not
  turn triggers off.

  Symmetrically, `PUT /v1/schemas/{id}` is rejected with a `409` if it would turn `triggersEnabled`
  off while trigger rules still fire off that schema, and `DELETE /v1/schemas/{id}` is refused with a
  `409` while any rule fires off it — delete those rules first, matching the endpoint's existing
  refusals for records of the type and for lineage variants. The `PUT` case matters because a schema
  update replaces `capabilities` **in full when you supply it**: omitting the block preserves it, but
  supplying a partial map (to change `auditHistory`, say) drops `triggersEnabled` along with it and
  would otherwise disable every trigger on the schema with no error.

- **Trigger scripts can now reach documents and folders, not just records.** The `vectros` host object
  gains two namespaces alongside `vectros.records.*`:
  `vectros.documents.create/get/update/delete/query/lookup` and
  `vectros.folders.create/get/update/delete/query` (folders have no `lookup` — there is no
  `/v1/folders/lookup` endpoint to mirror). Each verb is the in-process equivalent of the same
  `/v1/documents` or `/v1/folders` call, running through the identical scope, placement and validation
  path, so a script can do exactly what a scoped credential holding the same grant could do over HTTP
  — and nothing more. `update` is a partial (JSON Merge Patch) update on both, matching
  `records.update`. A trigger's `manifest` can now name these verbs (`documents.get`,
  `folders.create`, …); an entry naming a verb that does not exist — including `folders.lookup` — is
  rejected with a `400` listing the recognized set. Presigned-URL routes
  (`POST /v1/documents/upload`, `GET /v1/documents/{id}/download`) are deliberately **not** exposed to
  scripts: they hand back a credential that outlives the execution, which is a different capability
  from calling the API in-process. Purely additive — an existing trigger, script and manifest behave
  exactly as before.

- **A per-tenant ceiling on concurrent script executions.** At most **10** run concurrently per
  tenant — and trigger firings and `POST /v1/scripts/execute` draw from **separate pools of 10**, so
  a burst of synchronous calls cannot starve your triggers, or the other way round. A firing refused
  at the ceiling is recorded as the retryable category `TENANT_CONCURRENCY_LIMIT` and redelivered
  rather than dropped, and is neither charged nor metered; a synchronous call refused there fails the
  request. This is a fixed platform limit in this release, not a per-plan or per-request setting.

- **New `GET /v1/trigger-failures` endpoint** — find automation that has stopped running. Returns a
  page of trigger executions that failed: what went wrong (`category`, a closed set server-side but
  typed as a plain string in the SDKs, so match on the value and treat an unrecognised one as
  unknown rather than assuming your enum is exhaustive — among them `SCRIPT_ERROR`, `TIMEOUT`,
  `AUTHORIZATION_DENIED`, `MANIFEST_VIOLATION`, `CONCURRENT_MODIFICATION`, `RESOURCE_LIMIT_EXCEEDED`,
  `WRITE_BUFFER_CAP_EXCEEDED`, `RATE_LIMITED`, `CREDIT_LIMIT_EXCEEDED`, `TENANT_CONCURRENCY_LIMIT`
  and `INTERNAL_ERROR`), whether the platform will
  retry it, how many attempts it has made, and which of your rules, schemas and records were involved.
  Filter by `ruleId`, `category`, `retryable`, and a `from`/`to` time window; paginates like every
  other list endpoint. Requires the existing `triggers:r` scope — no new scope to grant. Until now a
  failing trigger left no partner-visible trace at all.

  **A record appears once per firing, not once per attempt.** If the platform retries a retryable
  failure, the same record's `attempts` count rises rather than a second record appearing — and **if a
  retry later succeeds, the record is removed**. So this endpoint tells you what is broken *now*; it
  is not an append-only history, and you should not build a permanent audit on it. Records also expire
  30 days after the firing first failed.

  **A refusal your script does not `catch` is categorised as that refusal**, not as an internal
  error: a `vectros.*` call your credential or manifest does not allow is recorded as
  `AUTHORIZATION_DENIED` or `MANIFEST_VIOLATION` naming the refused operation, and any other `4xx` a
  `vectros.*` call returns (a not-found, a validation failure, a version conflict) as `SCRIPT_ERROR`
  carrying that message — the same categories you get if the script catches the error and throws its
  own, and all of them bill execution time like any other script failure. `INTERNAL_ERROR` is reserved
  for a genuine platform fault: unbilled, and not retried as a script error.

  `detail` is safe to surface to your own users: it carries only your own content (a script's own
  thrown message) or public API vocabulary (verb names, scope dimensions, record ids), never platform
  internals. For `INTERNAL_ERROR` it is absent and a `correlationId` is returned instead — quote that
  to support. A `CONCURRENT_MODIFICATION` **names the specific records, documents and folders**
  another writer changed (by id, with no type word — an id is globally unique), so you know which rows
  to re-read; it deliberately does not return their current version numbers, because re-applying your
  change on top of a returned version overwrites the concurrent writer instead of reconciling with
  them. Re-read, re-apply, retry.

- **New `trigger.failed` webhook event** — subscribe a webhook to it to be told when a trigger
  execution fails, instead of polling the endpoint above. `data` carries the failure's own fields —
  the same ones a `GET /v1/trigger-failures` record carries, except its `createdAt`/`updatedAt`
  timestamps; the envelope's own `created` is the delivery's.

  **It fires once per problem, when the platform gives up** — immediately for a failure that cannot be
  retried, and only on the final attempt for one that can. A transient failure that succeeds on retry
  sends no webhook at all.

  **If you alert on this event, deduplicate on `data.id`**, which is stable for a given firing. Do
  **not** deduplicate on the envelope's own `id`: that is the delivery's id, minted fresh per
  delivery, so one firing fanning out to two subscribed webhooks produces two of them. In rare recovery cases we may re-send an event for a firing you were already told about, or
  recover a firing after telling you it failed — in which case the matching `GET /v1/trigger-failures`
  record is removed and that id will no longer resolve. Treat the event as "this failed" rather than
  as a durable handle. (The `GET` endpoint still shows the in-progress failure meanwhile, so you can
  see a retry in flight if you look; the webhook deliberately stays quiet until the outcome is
  settled.)

- **`POST /v1/records/batch` is now implemented** (closing a contract previously reserved as an HTTP
  501) — writes multiple records in one call. Each item in `items` has the same shape as a single
  `POST /v1/records` body and goes through the same schema validation, `externalId` idempotency,
  unique-field enforcement, and `records:c:<type>` scope check, applied **per item** against that
  item's own record type. `atomicity` selects how the batch commits: `best_effort` (the default, up to
  50 items) writes each item independently, so some can succeed while others fail; `all_or_nothing`
  (up to 50 items) commits every item in one transaction, and if any item fails nothing is written at
  all — reported with an `error.code` of `batch_not_committed` (a different item failed),
  `batch_refused` (the batch is too large to commit atomically; the practical maximum is below the
  item limit) or `batch_conflict` (a concurrent writer). The response is HTTP 200 whenever the batch
  was processed — including when every item failed —
  so always inspect the per-item `results` rather than the status code, matching each result to the
  item you sent via its `index`. `?upsert=true` and `?allowClear=true` are honored per item, exactly
  as on a single create.

  **In an `all_or_nothing` batch, an item may depend on a row written earlier in the same batch** — a
  child record can carry a `reference` to a parent created by an earlier item, and the batch commits.
  See the read-your-own-writes entry under *Changed* for the full rule, which applies identically to a
  script execution.

- **`BatchWriteResult.status` carries six values** — `created`, `updated`, `conflict`, `invalid`,
  `forbidden` and `not_committed` — and `succeeded` + `failed` account for every item you submitted.
  `forbidden` means your credential lacks the scope or ownership that item required, so the fix is to
  your credential rather than to the item; `not_committed` means the item itself was fine but an
  `all_or_nothing` batch was aborted by a different item, so nothing was written and resubmitting with
  that item fixed will write this one unchanged.

- **Issuer registrations (`POST /v1/auth/issuers`, `PUT /v1/auth/issuers/{issuerId}`) accept a new optional `capturedClaims`
  field** — a list of additional OIDC claim names (beyond `emailClaim`, which keeps its own dedicated
  field) to capture from that issuer's tokens on every successful token exchange, stored as your
  tenant's golden identity-provider-asserted copy for the signed-in user. Not a fixed set: name
  whatever claims your identity provider actually asserts (standard claims like `name` or
  `phone_number`, or your provider's own custom claims). Each claim is read from the verified token
  first, falling back to the registration's `userinfoUri` (if configured) only for names still missing
  after that. Purely additive — omit entirely to capture nothing beyond email (unchanged default
  behavior for every existing registration).

- **App contexts (`POST`/`PUT /v1/app-contexts`) accept a new optional `identityProjectionClaims`
  field** — declares which of your tenant's captured identity-provider claim names (see
  `capturedClaims` above) get projected, read-only, onto access profiles in this context. Filled in
  once per profile, the first time a sign-in for that principal can supply a value — usually at
  profile-creation, but for an invited member not until they actually accept and sign in for the first
  time. Once filled, a profile's projection does not update again even if this declaration or the
  underlying identity data changes later. Requires a root API key or your platform provisioning
  credential; an ordinary `app-contexts:u`-scoped token cannot set it. Projected values surface as a
  new read-only `identityProjection` object on `AccessProfileResponse`
  (`GET`/`POST`/`PUT` on `/v1/app-contexts/{contextId}/profiles/*`) — absent when your context
  declares no projection, when none of the declared names have a captured value for that principal
  yet, or before that principal's first successful sign-in. Platform-write-only: it cannot be set or
  changed via the profile's own `POST`/`PUT` body.

- **Role/AccessProfile scope clauses accept a new optional `assignable_roles` field** — a
  designated-roleId allow-list restricting WHICH named roles a clause may compose into a delegated
  AccessProfile (`roleIds` composition), independent of how much data the clause's own `data_scope`
  reaches. Absent/null means no roleId restriction (default-open), matching `data_scope`'s existing
  default-open-for-reads posture — every existing clause keeps behaving exactly as it does today; a
  tenant opts into the restriction by adding this field to a clause that grants `profiles:c`-shaped
  authority. Capped at 20 entries; purely additive — omit entirely for unchanged
  behavior. **Adoption note:** "unchanged behavior" holds only for non-adopters — once you add
  `assignable_roles` to a clause, that clause can only compose the roles the list NAMES — so plan a
  rollout that names every roleId the restricted clause still needs BEFORE adding the field, not
  after.

  One further condition, and it is the one that catches a common shape: a composed role that itself
  grants role-composing authority (`profiles`, `users`, `keys` or `triggers` with `c`, `u` or `d`, or a
  wildcard `*` — `users:c` is "can invite team members", so this is not an exotic case) must ALSO carry an
  `assignable_roles` no broader than yours. Without that, composing such a role would hand the
  delegate unrestricted composing power one hop later, which is the escalation the field exists to
  close. So those roles need updating before you adopt; an ordinary role that grants no composing
  authority of its own does not, and composing one keeps working exactly as it does today.

- **`POST /v1/auth/token`, `POST /v1/auth/token/exchange`, and `POST /v1/auth/token/assume` now return
  a new `resolvedScope` object** alongside the minted/exchanged/re-minted token —
  `{allowedActions: string[], identity: Record<string, string>}`, the same plaintext data baked into
  the token's own (compressed) `scope` claim, resolved server-side so you no longer need to decode the
  token to read it. `allowedActions` is the union of `allowed_actions` across every clause the token
  carries (`["*"]` for a wildcard-scoped credential); `identity` is the token's own `identity` claim,
  keyed by the same public dimension names (`userId`, `scope:<namespace>`) you send on mint. Purely
  additive — every existing response field is unchanged.

- **`POST /v1/search` and `POST /v1/rag` accept a new optional `scopeFilters` array** (on `/v1/rag`
  it sits inside `search`) — narrow a search or RAG retrieval by more than one ownership dimension in
  a single request, for example
  `["org:<id>", "client:<id>"]` to narrow to one specific client within one specific org. Mutually
  exclusive with the existing `scope` field: use `scope` for a single dimension, `scopeFilters` when
  you need more than one. Purely additive — omit entirely for unchanged behavior.

### Security

- **`POST /v1/admin/keys/scoped` no longer mints a key against a `suspended` access profile.** It
  returned `201` and handed back a working `ssk_*` bound to a profile that had already been suspended,
  so suspending a profile did not stop new credentials being issued against it. It now returns `409`
  naming the profile, before anything is written. **This changes an outcome you may be relying on:** a
  mint that previously succeeded against a suspended profile now fails, for every credential including
  a root `sk_*`, and a repeat `POST` of a tuple whose key already exists is refused too rather than
  returning its metadata with `200`. Reactivate the profile
  (`PUT /v1/app-contexts/{contextId}/profiles/{principalId}` with `status: active`) and retry.

  The status is `409`, not `403`, on purpose: every `403` this endpoint returns is about your
  credential being insufficient, while this one is about the referenced profile's state — if you
  handle `403` by re-issuing a broader credential, that is the wrong remedy here.

  **The token exchange is fixed in the same release, so issuance is now immediate on both paths.**
  `POST /v1/auth/token/exchange` could also mint a brand-new `st_*` against a suspended profile for up
  to five minutes, because the resolved-scope cache served a memoized scope without re-reading the
  profile's status — and since an `st_*` carries its own one-hour lifetime, that produced credentials
  valid roughly 65 minutes past the suspension. The exchange now reads the profile's status on every
  request and returns its usual uniform `403 invalid_grant`. `AccessProfileRequest`'s `status`
  description previously described both paths as returning "a uniform 403"; they are now described
  separately, because they answer differently.

  **What is NOT changed: credentials already issued.** An `ssk_*` or `st_*` handed out before you
  suspended may keep working for up to five minutes while the access-profile cache expires, and an
  `st_*` keeps its own one-hour lifetime regardless. Suspension is immediate containment against new
  credentials and eventually-consistent against existing ones; to stop a specific credential now,
  revoke that credential.

- **`POST /v1/admin/keys/scoped` no longer mints a key for a `SUSPENDED` user.** Suspending a user via
  `PUT /v1/users/{id}` does not suspend their access profiles, and the mint did not check the user's
  own status — so a suspended user could still have new keys minted for them, though `status`'s own
  documentation says `SUSPENDED` users are blocked from new operations. It now returns `409` naming
  the user. `PENDING` users are deliberately unaffected: a pending user is an outstanding invitation,
  and minting against one is a supported bootstrap step. Note this prevents ISSUANCE only — a key
  minted before the user was suspended is unaffected, since user status is not consulted when a
  credential authenticates.

  Two details worth knowing if you automate around this. The check is **case-insensitive and
  allow-listed**: `status` on a user is `ACTIVE`/`SUSPENDED`/`PENDING` in upper case, while `status`
  on an access profile is `active`/`suspended` in lower case, and any value that is not recognisably
  `ACTIVE` or `PENDING` is treated as not mintable. And the developer-portal token route
  (`GET /developer/scoped-token`) got the same rule in this release — a suspended member is refused
  there too, with that route's usual uniform `404`, since it does not distinguish refusal causes.

### Changed

- **A record or document write is refused (`400`) when the fields its schema keeps `inline` total more
  than 224 KB on that item** — the same limit as a trigger's projected input, which is what `inline`
  exists to feed. Without it the identical write reached the storage row's 400 KB ceiling as a `500`. Only
  `inline: true` fields can reach this; `filterable` fields and lookup fields keep their existing,
  smaller budgets.

- **A root API key can no longer file a document, record or folder into an AppContext other than
  `default`.** `POST`/`PUT /v1/documents`, `POST /v1/documents/upload`, `POST`/`PUT
  /v1/records` and `POST /v1/folders` previously accepted a `folderId`/`parentFolderId` naming a
  folder in any of the tenant's contexts when called with an unscoped root key; they now return the
  same uniform `400 "Folder not found"` that every other credential already received.

  **This never did what it appeared to do.** A root key's writes are always stamped with the
  `default` context, so the child landed in `default` while its folder lived elsewhere — the row's
  containment and its context partition disagreed permanently. Such a row is listed by
  `GET /v1/documents?folderId=` only for callers in `default`, and is invisible to the context that
  owns the folder. No coherent placement is lost: the accepted shape was the incoherent one.

  **The same rule now applies to the SCHEMA a row is bound to**, for the same reason and on the same
  credential. `POST`/`PUT`/`PATCH /v1/documents`, `POST /v1/documents/upload`, and the `schemaId` of a
  trigger rule's `firingSource` previously accepted a schema belonging to another AppContext when
  called with an unscoped root key; they now return `400 "Schema not found"`. A root-written row is
  stamped `default`, so binding it to another context's schema left the row pointing at a definition
  no caller confined to the row's own context could resolve. Records are unaffected on update — a
  record's `schemaId` is immutable once created — and unchanged for every scoped credential, which
  could always bind only its own context's schemas.

  **If you were relying on either**, the supported route is a scoped key or token bound to the target
  context (`POST /v1/admin/keys/scoped` with that `contextId`), which could always file into and bind
  within its own context and still can. Rows written in the old shape are unchanged by this release
  and are still readable, updatable and deletable: an update that omits `folderId` preserves the
  existing placement, one that carries the existing `schemaId` forward keeps working, and supplying a
  `default`-context folder or schema re-files the row into a coherent shape.

- **Inside one script execution or one `all_or_nothing` batch, reads and guards see that operation's
  own uncommitted writes.** Each read and each guard was previously answered from the state that
  existed before the operation started, so a script or a batch could contradict itself. Now:

  - `vectros.records.query(...)`/`lookup(...)`, `vectros.documents.query(...)`/`lookup(...)` and
    `vectros.folders.query(...)` subtract what the same execution has deleted, and show a row the same
    execution re-saved with its updated content — so a script that removes rows and then returns "what
    remains" hands back a list consistent with the commit it reports. Previously `get(id)` on such a
    row already returned 404 while a list still returned it. This is a property of every paginated
    read the execution makes, not of the three `query` functions by name.
  - Uniqueness and existence guards see rows staged earlier in the same operation. A second write of
    the same unique field value in one operation is refused rather than both writes being admitted to
    collide at commit; and a row that references or requires another row staged earlier in the same
    operation resolves against it instead of failing as missing. This is what lets an
    `all_or_nothing` batch file a child under a parent it creates in the same call.
  - A second write to the *same* row within one operation builds on the first, not on the state before
    the operation began. Two partial updates of one record in one execution previously each merged
    onto the pre-operation baseline, so the second silently discarded the first's fields.
  - An idempotent create still finds a row this operation has written to. Re-sending a create for an
    `externalId` a row already held — or a folder slug — returns that row, carrying this operation's
    own updated content, rather than colliding with the operation's own staged write. Claiming an
    identifier that only a SIBLING of this operation staged stays refused, on records, documents and
    folders alike: creating one identifier twice in a single operation is a mistake, not an
    idempotent repeat, and the two cases are deliberately answered differently.

  Two limits are deliberate and unchanged. A list does not show a row the same operation *created* —
  you hold its id from `create()`. And what your credential is *permitted* to do is always decided
  from committed state, so a script cannot widen (or narrow) its own permissions mid-execution by
  editing the records that grant them. Nothing changes for reads made outside an execution or batch.

- **A `filterable: true` field may no longer be declared with a field id the platform owns in the
  search index.** `POST`/`PUT /v1/schemas` now rejects, with a `400`, a filterable field whose
  `fieldId` collides with a platform-written search-index metadata key for a surface the schema binds
  to: `tenantId`, `owner_id`, `folderId`, `rootFolderId` and `recordType` for a schema allowing the
  `record` surface, and those plus `model_type` and `title` for one allowing `document`. Such a field
  shadowed the platform's own value in the index, silently costing recall and precision on
  type-scoped searches. **This changes an outcome you may be relying on:** an existing schema
  declaring such a field is rejected the next time you `PUT` it, and a blueprint that declares one
  fails to apply. Rename the field, or leave it non-filterable — a non-filterable field of any name is
  unaffected, as is a schema that binds only to `user`/`entity` surfaces.

- **Records no longer publish their lifecycle `status` into search-index metadata.** The key was
  information-free — an archived record is removed from the index entirely — and nothing read it, but
  it meant a record schema could not safely use `status` as a filterable field of its own. It now can:
  `status` is reserved on neither surface. **This changes an outcome you may be relying on:** a
  `POST /v1/search` `filters` clause on `status` no longer matches the platform's lifecycle value —
  for a schema that declares its own filterable `status` field it now matches that field, and for one
  that does not it matches nothing. **`POST /v1/search`'s own published `filters` example used to be
  `status`, and is now the neutral, non-colliding `category`** — if you started from that example,
  yours is the clause this changes. Documents are unaffected: they
  have always kept lifecycle off the index, on the dedicated `lifecycleStatus` field.

- **Webhook delivery history is now retained for 30 days.** A successfully delivered webhook left a
  permanent record carrying the full event payload, with nothing to reclaim it. A delivered record
  now expires **30 days after the delivery record was created** (not 30 days after it succeeded) —
  long enough to review recent deliveries in the developer portal, and a bound on how long the
  platform retains your event payloads. Undelivered and failed records keep their existing 7-day
  window, measured the same way.

- **`status` descriptions on `UserRequest` and `EntityRequest` corrected.** Both said `SUSPENDED`
  records "are retained but blocked from new operations", which promised enforcement the platform does
  not perform. Suspending a **user** stops new credentials being issued for them and does **not**
  revoke credentials they already hold — those keep working until they expire or are revoked.
  Suspending an **entity** does nothing at
  all: the field is a label you set and read back, and a suspended entity is still readable, updatable
  and referenceable. Enforce entity status in your own application if you need it to have an effect.
  No behaviour changed here — the documentation did.

### Removed

- **`PENDING` is no longer an accepted value of the `status` request field on `POST`/`PUT /v1/users`.**
  It is a server-managed state and the field's own description already said you could not set it
  directly, but it was listed among the accepted values, so the OpenAPI spec and all three SDKs
  published it as settable. Sending it now returns `400` (see *Fixed* below). **Nothing about reading
  changes:** a user is still returned with `status: "PENDING"` while an invitation is outstanding, and
  the response field still carries all three values. An invited user is still activated exactly as
  before — `status: "ACTIVE"` together with `inviteToken`, `externalSubject` and
  `emailVerifiedAttestation: true`.

  In a typed SDK this narrows a generated enum, so code that names the `PENDING` member of the
  *request* type will no longer compile; switch to the response type's enum, or to the string literal.

### Fixed

- **`status: "ARCHIVED"` now reliably keeps a DOCUMENT out of search.** Archiving is documented to
  retract an item from search and recall while keeping it stored, and it did — but a later write to a
  document could silently put it back. Writing new content to an already-archived document re-created
  its search-index entry, leaving the document reporting `ARCHIVED` while still being returned by
  `POST /v1/search` and RAG, with no error on either side. Creating a document directly with
  `status: "ARCHIVED"` indexed it for the same reason. **Records were not affected by either of
  those** — they have refused to index an archived record since 0.35. Both document paths are now
  fixed, and writing to an archived document no longer moves its `indexStatus` to `PENDING_INDEX`:
  nothing is queued for an archived document, so the field keeps the value it already had rather than
  reporting work that will never happen.

  **For documents and records alike:** re-sending the archive now re-asserts the retraction instead of
  doing nothing, so an item found archived-but-still-searchable can be repaired without a hard delete;
  and background re-index passes drop an archived item's index entry rather than refreshing it.

  **If you have an item that is already archived-but-still-searchable, it stays that way until it is
  written to.** Nothing sweeps for the existing ones — the periodic reconcile pass only revisits items
  whose indexing did not complete, and a stranded item's index entry looks perfectly healthy to it.
  Re-sending `status: "ARCHIVED"` on the item is the fix, and now works.

- **`createdAt` on a search hit is documented accurately.** The field was described as when the source
  item was created. It is when the item was added to the *search index*, which is the same moment in
  the common case but is later for anything that was re-indexed onto a new entry. The description now
  says so and points at `documents.get` / `records.get` for the source creation time. No behaviour
  changed — the value was always the index time.

- **The update paths now validate `status` the way the create paths always did.** Three endpoints
  stored whatever `status` string you sent:

  - `PUT /v1/users/{id}`
  - `PUT /v1/entities/{namespace}/{id}`
  - `POST /v1/entities/{namespace}?upsert=true`, when it matches an existing entity

  On users, `POST /v1/users` and `POST /v1/users?upsert=true` both already normalised and rejected, so
  the same request was accepted or refused depending only on which verb you used. On entities, only
  the create branch did.

  **This changes an outcome you may be relying on.** On all three endpoints:

  - a lowercase or mixed-case `status` is now normalised — send `"suspended"`, read back `"SUSPENDED"`
    (previously it was stored verbatim);
  - a value outside `ACTIVE`/`SUSPENDED` — including the empty string — now returns `400` instead of
    being stored;
  - `status: "PENDING"` on `PUT /v1/users/{id}` now returns `400`. This includes echoing a pending
    user's own status back in a read-modify-write, which is the one case where a request that used to
    succeed now fails. Omit `status` from the body to update a pending user's other fields.
    `POST /v1/users?upsert=true` already refused `PENDING`, so the two write paths now agree.

  An explicit `"status": null` still means "leave it unchanged", exactly as omitting the field does.

  **One consequence specific to entity upsert.** A re-applied upsert whose content matches the stored
  entity is a documented no-op — no version bump, no audit row, no write fee. Because the incoming
  `status` is now normalised before that comparison, an upsert that re-sends `"suspended"` against an
  entity stored as `"suspended"` no longer matches and is written once, canonicalising the row. From
  then on it matches and is a no-op again. Only entities holding a non-canonical status are affected,
  and each is affected once.

  **Activating a pending user now requires the invite token however their status is spelled.** A user
  whose stored status was a non-canonical spelling of `pending` was not recognised as pending, so
  sending `status: "ACTIVE"` for them set them active **without an invite token**, on
  `PUT /v1/users/{id}` and on `POST /v1/users?upsert=true` alike. Both now compare the stored value
  case-insensitively. If you have such a user and no longer hold their invitation, re-invite them.

  Sending a non-canonical `status` to `PUT /v1/users/{id}` had real consequences. A user stored as
  `"active"` rather than `"ACTIVE"` was refused at `POST /v1/auth/token/exchange` with a uniform `403`
  — the sign-in path compares the value exactly. Repair such a row by sending `status: "ACTIVE"`; the
  update now rewrites it to the canonical value. Sending `status: "PENDING"` was worse: it demoted a
  live user into the invitation state, with no way back through the API, because the pending-to-active
  transition requires the `inviteToken` from an invitation that a demoted user never had. That door is
  now closed at the entrance.

- **A browser SDK caller now sees the API's shaped error instead of a CORS failure.** Preflight for a
  request that never reaches the API's own handlers — an unmatched route, an expired or rejected
  token, a throttled or firewall-blocked request — was answered with an allow-list that omitted
  headers every generated SDK sends on every request, so the browser refused to send the real request
  and the developer saw a bare CORS error with no status. All three allow-lists now carry the same
  full set, so those responses arrive as the `404`, `401` or `403` they were shaped to be. Affects
  browser-hosted integrations only; server-side callers were never subject to preflight.

- **Webhook deliveries are no longer silently stranded or dropped.** Two paths lost a delivery with no
  retry, no failure record, and nothing to see: an attempt killed part-way through never incremented
  its attempt count or scheduled a retry, and a failure to enqueue the attempt at all left the
  delivery permanently pending. Both now record the failure and schedule the retry the delivery policy
  calls for. No partner-visible field or payload changed — deliveries that would previously have
  vanished now arrive, or surface as failures.

- **A refused document update no longer leaves the new text already written over the old.**
  `PUT /v1/documents/{id}` wrote the supplied `text` to storage before the checks that can reject the
  request, and reused the document's own object key, so a rejected update replaced the live body and
  then returned `400` — while the document's metadata and search index still described the previous
  text. The write now happens after every such check. A valid update behaves exactly as before.

- **Moving an `externalId` onto one another resource already holds is now rejected** — on documents
  via a re-type, and on identity entities directly. An `externalId` is unique per type for a document
  (`(tenant, context, type)`) and, for an entity, per namespace **within its placement** — a namespace
  registered to one app context is a separate space from a tenant-wide one of the same name, so the
  same `externalId` may legitimately exist in both. Nothing on either update path re-checked that
  uniqueness, so the identifier could be moved onto an occupied slot and two resources would end up
  sharing one. After that, every lookup keyed on that `externalId` resolved to an arbitrary one of the
  two: the idempotent-create echo, `GET /v1/entities/{namespace}?externalId=`, and — the sharp end —
  `POST /v1/documents?upsert=true`, which could overwrite the wrong document with no error anywhere.

  **What this is, precisely:** a check performed against the current state when your request is
  processed — not a reservation. Two requests moving onto the *same free* identifier at the same
  instant can still both succeed, exactly as two simultaneous creates of the same `externalId` always
  could. If you rely on an `externalId` being held by exactly one resource, treat this as the safety
  net it is and keep your own writes serialised.

  Two ways to reach it, both now refused with `400`:

  - **Documents.** `PATCH`/`PUT /v1/documents/{id}` changing `schemaId` to a type on which the
    document's `externalId` is already taken. (The `externalId` *value* was already immutable on
    update; its uniqueness *scope* was not, because the scope is the bound schema's type.)
  - **Identity entities.** `PUT /v1/entities/{namespace}/{id}` changing `externalId` to one another
    entity in that namespace already holds.

  **This changes an outcome you may be relying on**, but only in the colliding case. Re-typing a
  document onto a type where its `externalId` is free still works, as does binding a schema to a
  previously untyped document, re-typing a document that has no `externalId` at all, and moving an
  entity onto an unused `externalId`. Re-sending a resource's current `schemaId` or `externalId` is
  unaffected. If you hit the refusal, the identifier is genuinely in use — pick a different one, or
  delete the resource holding it first. The error names the conflict but never identifies the other
  resource.

- **`DELETE /v1/folders/{id}` no longer deletes a folder that still holds documents or records.** The
  emptiness guard saw sub-folders and file-backed documents but was blind to text-ingested documents
  and to records, so deleting such a folder returned `204` and left its documents and records readable
  and listable behind a `folderId` that no longer resolves. The endpoint now rejects with `400` unless
  the folder is empty across all three populations. **This changes an outcome you may be relying on:**
  a `DELETE` that previously succeeded on a folder holding text documents or records now fails. Empty
  the folder first — `GET /v1/documents?folderId=`, `GET /v1/records?folderId=` and
  `GET /v1/folders?parentFolderId=` enumerate what is in it.

  Two consequences worth knowing before you hit them:

  - **The guard counts everything in the folder, including items your credential cannot read.** If
    your credential is scoped to a subset of a shared tenant and someone else files a document or
    record into a folder you own, that folder now refuses to delete for you, and your own
    `GET /v1/documents?folderId=` will not show you why. This is deliberate — the alternative is
    deleting the container out from under data you were not allowed to see — but it means such a
    folder needs a credential that can reach the whole folder to empty it. The refusal discloses only
    that *something* is filed there: never what, whose, or how many.
  - **Folders already orphaned by the old behaviour are not repaired by this change**, and there is no
    endpoint that enumerates them: `GET /v1/documents?folderId=` needs the folder id, which is exactly
    what is gone. To clean them up, page `GET /v1/documents` and `GET /v1/records` and re-file
    (`PATCH` the `folderId`) any row whose `folderId` no longer resolves via `GET /v1/folders/{id}`.

- **Two schemas can no longer be created with the same `recordType`.** Two concurrent
  `POST /v1/schemas` calls naming one `recordType` could both succeed, and the duplicate then broke
  subsequent record writes against that type — for every caller, not just the two racing creates. The
  type name is now claimed atomically, so one of the two creates wins and the other is refused. A
  sequential create of a duplicate `recordType` was, and remains, refused.

## 0.42.0 — 2026-08-30

### Added

- **`GET`/`POST /v1/documents/lookup` now accept `sortFrom`/`sortTo`**, narrowing an exact-value
  document lookup by the lookup field's declared `sortBy`, on either index tier — the document
  surface's equivalent of the existing `sortFrom`/`sortTo` on `/v1/records/lookup`. Purely
  additive: both fields are optional and every existing request is unaffected. Composite
  (multi-field) lookups remain records-only; documents keep single-field lookups.
- **App contexts (`POST`/`PUT /v1/app-contexts`) accept a new optional `companyName` field**,
  distinct from the existing `name` field. `name` is your app's own identity; `companyName` is
  your own organization's display name, used to personalize platform-sent correspondence (e.g.
  sub-user invitation emails) with your own branding instead of a generic app name. Purely
  additive — optional on every request, absent on every context that predates this release.

- **New `delegate-principal-stamp` capability**, grantable in `granted_capabilities`. Without it, a
  record's `userId` on create must match the credential's own identity. With it, a `data_scope`
  clause that already authorizes the write may also explicitly name a different `userId` in your
  own tenant, so an admin role can create a membership-shaped record on behalf of a known user
  instead of only itself. Grants nothing by itself — the clause authorizing the write must still name
  `userId` (the capability itself may be granted on a different clause of the same role) — and does
  not extend to updates, which continue to reject any principal reassignment regardless of this
  capability.

- **`POST /v1/rag`'s `truncation_warning` event now reports two independent drop reasons
  separately.** It previously fired only when results were dropped for exceeding the model's
  context window; it now also fires when a result had no groundable text to include at all,
  regardless of budget. The event carries two new fields, `truncatedCount` and `noContentCount`,
  so you can tell the two apart instead of inferring the cause from `resultsUsed` alone — the two
  call for different responses (raise `maxTokens` / narrow your query vs. nothing actionable on
  your end). `reason` gains two new values, `no_groundable_content` and
  `context_window_budget_and_no_content`, alongside the existing `context_window_budget`. Purely
  additive; the event's existing fields and firing name are unchanged.

### Security

- **Registering a trusted issuer (`POST /v1/auth/issuers`) now closes a race on the
  `(issuer, audience)` uniqueness check.** Two concurrent registrations for the same pair under
  different `issuerId`s could previously both succeed, which `POST /v1/auth/token/exchange` would
  then need to arbitrate between arbitrarily. Registration now atomically claims the pair before
  writing, and token exchange refuses (rather than picks) if it ever finds more than one active
  registration sharing a pair.
- **Token exchange no longer trusts an unverified external-identity binding.** `PUT /v1/users/{id}`
  lets you record a user's `externalSubject` yourself, but — as its operation description has
  always stated — that binding is informational only and was never meant to enable sign-in;
  `POST /v1/auth/token/exchange` is supposed to compute and trust only the value it independently
  verifies from a real credential. Previously the two kinds of binding weren't actually
  distinguished internally, so a self-reported binding could, in practice, still be matched at
  exchange time. Exchange now requires the stronger, independently-verified provenance before
  honoring a match; a self-reported binding remains exactly as useful as documented — a record of
  which of your own users an account corresponds to — and can never itself grant a session. No
  action needed: every binding that predates this release already had the verified provenance.

### Changed

- **The `member-lifecycle` capability now also covers resending an invitation and attaching an
  existing member to an additional app context**, matching its existing reach over creating and
  removing one. A credential scoped to a single app context, holding `member-lifecycle` plus the
  matching unqualified `profiles:*` grant(s), can now do everything a tenant-wide
  `users:c`/`users:r`/`users:u` credential could already do on `POST /v1/users/invite` and
  `POST /v1/users/invite/resend` — but the grant needed is not the same three letters in every
  case, so grant only what the specific operation requires:
  - `profiles:c` is the pre-existing baseline for both endpoints — unchanged by this release, but
    still required: it's what gates entry to `POST /v1/users/invite` at all (including every case
    below), and `POST /v1/users/invite/resend` requires it explicitly alongside `profiles:r`/
    `profiles:u`. Omitting it is not an option for either endpoint.
  - `POST /v1/users/invite/resend`, and `POST /v1/users/invite` when the invited email already
    has a PENDING invitation (in the SAME context, or elsewhere in your tenant), all rotate a live
    credential and therefore need `profiles:r` **and** `profiles:u` together (disclosure and
    mutation, correlated) — on top of the `profiles:c` baseline above.
  - `POST /v1/users/invite` attaching an **already-ACTIVE** member to a new context mints no
    token — nothing is mutated — so it needs only `profiles:r` on top of the baseline; granting
    `profiles:u` too is unnecessary for this case specifically.
  See the operation descriptions on `POST /v1/users/invite` and `/invite/resend` for the exact
  per-branch requirement.

### Fixed

- **Listing records/documents/folders/entities by `scope`/`userId` filters no longer requires
  restating every ownership dimension your role holds.** A role authorized over more than one
  ownership dimension at once (e.g. both `scope:org` and `scope:client`) previously couldn't
  successfully list anything: the list endpoints rejected any `?scope=`/`?userId=` combination
  that didn't name every dimension a clause carried, but a single call only accepts one
  `namespace:value` pair. Listing now succeeds once your filter is backed by at least one clause
  dimension; every returned row is still filtered against your full data scope exactly as before,
  so nothing you couldn't already read is now visible.
- **Sub-user invitation emails now carry the app context's own branding, not your account's
  internal name.** The email previously named your own vendor/account organization name and
  unconditionally signed off "on Vectros," regardless of whether the app context inviting the
  recipient is white-labeled. It now uses the app context's `name` (and `companyName`, when set)
  and drops the hardcoded Vectros branding, so an invitee sees the product they're actually
  joining rather than your platform vendor relationship.
- **A duplicate `externalId` on `POST /v1/users` under concurrent requests now returns the
  documented `400 already exists` instead of an unhandled `500`.** Two near-simultaneous creates
  for the same `externalId` could previously race past the initial duplicate check and have the
  loser fail with a generic server error; it now resolves to the same clean "already exists"
  response a non-racing duplicate request already got.
- **`POST /v1/erasure-requests` now validates `contextScope` up front.** Each entry must name an
  app context that actually exists for the caller's tenant; an unknown or mistyped context id now
  returns `400` immediately instead of being silently accepted. Previously such a request was
  accepted and, due to a separate erasure-engine bug now also fixed, still erased the subject's
  data in every *real* context named alongside the bad one — masking the typo. With that bug fixed,
  an unvalidated bad context would instead have let the request report `completed` having erased
  nothing at all for that context. If you scope erasure requests to specific contexts, double-check
  the context id spelling against `GET /v1/app-contexts`.
- **`score` on a search/RAG result is now documented accurately — no wire-shape change.**
  The field's description previously implied a single, roughly 0–1 "relevance confidence"
  in every mode; in `HYBRID` mode it is actually a Reciprocal Rank Fusion value — small and
  tightly clustered (a top single-leg hit scores ~0.016, a top dual-leg hit ~0.033) — on a
  genuinely different scale from `TEXT`- or `SEMANTIC`-only mode's own native score. The
  description and example on `GET /v1/search`'s `SearchResult.score` and `POST /v1/rag`'s
  `RagSearchResult.score` now explain the per-mode scale so a `HYBRID` score of `0.02`
  isn't mistaken for "2% relevant," and a similarity-style threshold isn't set against it
  expecting it to behave like a 0–1 confidence.

## 0.41.0 — 2026-08-27

### Added

- **`PUT /v1/auth/issuers/{issuerId}` — update a registered trusted issuer's safe fields.**
  Previously a registered issuer's `subClaim`/`emailClaim`/`status`/`selfSignupPolicies` could only
  be set once, at registration — getting one wrong (or needing to change a self-signup policy after
  initial setup) meant delete-and-re-register, which was itself refused once any real user had ever
  bound through the issuer, leaving no recovery path at all. The new endpoint updates just that
  "safe" field set (fields that only affect *future* first-logins, or whether the issuer's tokens
  are currently honored). `issuer`/`jwksUri`/`audience` (the cryptographic trust anchor) and
  `contextId` (the routing pin) stay immutable via this route — supplying a differing value is
  rejected with `400`, naming the field; echoing the current value back is a no-op. As part of this,
  `status: "suspended"` is now actually reachable: setting it rejects the issuer's tokens at
  exchange time identically to an unregistered issuer (the enforcement already existed; nothing
  could set the status before this).

- **A registered issuer can now resolve an invitee's email via the IdP's OIDC userinfo endpoint,
  when the presented token doesn't carry it directly.** `POST /v1/auth/token/exchange` presents your
  IdP's access token, and access tokens generally don't carry an `email` claim under OIDC unless your
  IdP was specifically configured to add one — previously, a first-login attempt against such a token
  failed with no way to recover except that per-deployment IdP-side workaround. Set the new
  `userinfoUri` field (`POST /v1/auth/issuers`, or `PUT /v1/auth/issuers/{issuerId}`) to your IdP's
  userinfo endpoint to enable a fallback: when the configured `emailClaim` is absent from the
  presented token, Vectros calls `userinfoUri` (with that same token as the bearer credential) and
  reads `emailClaim` from its response instead. The fallback is tried only on a miss, never
  unconditionally, and is entirely opt-in — omit `userinfoUri` to leave today's behavior unchanged.

- **An access profile can now reference SEVERAL roles.** `POST`/`PUT /v1/app-contexts/{id}/profiles`
  accepts `roleIds: ["case-handler", "hr-admin"]` in place of the single `roleId`. The effective grant
  is each named role's own clauses concatenated in the order you list them — roles compose additively
  and are never merged, so every clause keeps meaning exactly what its own author wrote. Every id must
  name a role in the same app context, and no id may repeat. `roleId` still works and is now
  deprecated: sending both is a `400`, and reads return `roleIds` always plus `roleId` only when
  exactly one role composes.

- **`POST /v1/auth/token/assume` accepts more than one namespace per call.** The body may now name
  several `scope:<namespace>` keys at once — provided a SINGLE one of your roles grants all of them
  together. The combination is never assembled across two roles, because no role author would have
  vouched for it.

### Changed

- **`POST /v1/auth/token/assume` checks your entitlement LIVE, against your roles as they are right
  now** — not against a copy captured when your token was minted. Three consequences worth planning
  for:
  - **A token produced by `/assume` can no longer assume again** (`403`). Every assume starts from the
    token you exchanged for, so the identity you end up with is always one a single role explicitly
    granted rather than a combination reached by chaining calls. Keep your original token if you need
    to switch more than once, or exchange for a new one.
  - **A new `409` response** means your credential's basis changed: the access profile or a role it
    was minted from has since been edited, suspended or deleted. Re-authenticate to get a current
    token, then retry the same request. Existing `400`/`401`/`403` cases are unchanged.
  - **What survives the re-mint is now stated per clause.** `/assume` previously promised "every other
    claim preserved verbatim". It now preserves every clause whose reach does not DEPEND on a namespace
    you are changing; clauses that do are kept only if they come from a role that authorized the new
    value. A clause depends on a namespace when it filters on it OR when one of its values resolves
    against your own value for it (`${{ self.scope.<ns> }}`, `${{ under.self.scope.<ns> }}`) — the
    latter can sit under a different key, so it is not always the key you would expect. A role that does not authorize it loses its clauses touching that namespace,
    including any scoped to the value you already held. Assume into a value one role grants and you
    keep that role's reach, not the reach of roles that never vouched for it.

- **An `assumable` grant may no longer use `${{ under.self.scope.<namespace> }}`** (`400` at authoring
  on `POST`/`PUT` of a role or profile). That form resolves against your own current value for a
  namespace `/assume` can itself change, so what it admitted would depend on what was last assumed.
  Use `${{ under.self.userId }}`, `${{ member.scope.<namespace>[:level] }}`, or a plain literal — all
  of which mean the same thing before and after an assume. The same matcher remains valid in
  `data_scope`, where it is re-derived per write.

- **A role or access profile's `scopes` list is now size-limited.** Authoring a `Role` or
  `AccessProfile` (`POST`/`PUT /v1/app-contexts/{id}/roles`, `.../profiles`) with more than 300
  clauses, or whose `scopes` **and `assumable` combined** serialize to more than 5.5 KiB once
  compressed, now returns a clear `400` at authoring time. Previously nothing bounded this, so an
  oversized role could silently save and then fail unpredictably at request time once its holder's
  token exceeded API Gateway's header budget. Existing scoped tokens (`st_*`) are unaffected in shape
  — this only bounds new/updated authoring.

### Fixed

- **`POST /v1/users/invite` no longer rejects inviting an email that already has an identity in
  your OTHER tenant (test vs. live).** Previously this returned `409` unconditionally — even for
  the common case of inviting yourself, or a teammate who has already used your live tenant, into a
  freshly-created test tenant. That email now gets a genuine second, independent membership in the
  tenant you're inviting into, exactly like a brand-new email would.

- **`POST /v1/users/invite` also grants access to an ADDITIONAL app context in the SAME tenant, for
  an email that already resolves to an existing member there** — including the tenant owner's own
  email, the account's founding identity every tenant has from creation. Previously this
  unconditionally `409`'d, exactly like the cross-tenant case above, and for the owner's own email it
  was unconditional and permanent: nothing about it could ever be worked around, since the owner's
  identity exists from the moment the tenant does. If the email resolves to an ACTIVE member with no
  existing access to the app context you're inviting into, that access is granted immediately (`201`,
  no email is sent — there is nothing to accept, `emailSent` is `false` and the token/link fields are
  absent) — this case additionally requires the `users:r` scope alongside `users:c`, since the
  response names the existing member's `userId`. If it resolves to a member whose ORIGINAL invitation
  is still pending, the new context's access is attached to that same outstanding invitation and its
  token is rotated — requiring `users:r`+`users:u`, same as an ordinary same-context resend. A
  collision within the SAME app context (an active/suspended member of that exact context, or another
  outstanding invitation for that email under that exact context) still returns `409` as before, and a
  SUSPENDED identity is never granted new-context access this way — reactivate it explicitly first.

- **The immediate-grant case above now checks that the existing member can actually sign in to the
  new context before granting it silently.** An active member's credential only works for the identity
  provider it was originally created through — granting them a context that uses a DIFFERENT provider
  (a different `POST /v1/auth/issuers` registration) left no way to ever authenticate into it, a silent
  dead end previously reachable by, for example, a tenant owner (Cognito-only) being granted a context
  federated through your own registered IdP. When the existing member's credential is compatible —
  the new context has no registered issuer of its own AND the member isn't already proven bound
  exclusively through a different, BYO IdP, or they're already bound through that exact issuer —
  behavior is unchanged from above. When it isn't, a normal, independent invitation is created instead
  — its own new `userId`, and a real token/accept link when `sendEmail` is `false` — exactly as if the
  email had no existing identity in the tenant at all.

- **`org` and `client` are no longer auto-provisioned at account creation.** Every namespace, `org`/
  `client` included, now requires the same explicit `POST /v1/namespaces` registration before use —
  matching what this reference doc already described (`docs/features/vectros/identity-access/
  reference.md`, on `main` since before this release) but the platform didn't yet do. For a **new**
  account: `GET /v1/namespaces` no longer lists `org`/`client` until you register them, and
  `POST /v1/entities/org` (or `client`) 400s with "not entity-backed" until you do. Existing accounts'
  already-provisioned `org`/`client` registrations are unaffected — this changes provisioning-time
  behavior only, not existing rows. As a side effect, a `?scope=org:...`/`?scope=client:...` list
  cursor issued before this release is invalidated on first use after deploy (`400`, fail-closed) —
  the same precedent as the prior `scopes2`→`scopes3` cursor-format change.

### Added

- **Entity responses now carry `contextId`.** `GET`/`POST`/`PUT /v1/entities/{namespace}/...` never
  surfaced which app context an entity belongs to — tenant-wide vs. a specific context — even though
  0.40.0's split-registry work made a namespace registration's placement visible in its own response.
  Every entity response now carries `contextId` (`null` for a tenant-wide entity), the same convention
  `GET /v1/namespaces` already uses. Additive; no existing field changes.

- **A blueprint's bootstrap credential can now register a tenant-wide namespace.** `POST
  /v1/namespaces` previously 403'd unconditionally whenever a bootstrap credential omitted
  `?contextId=` — a tenant-wide registration required a direct root API key. The bootstrap credential
  now carries a second, independent, owner-only capability that permits exactly that one case (a
  `null`-target registration visible to every app context); naming any other context still confines
  the credential to its own, exactly as before. **This is live today for any account OWNER**, not
  merely a platform-side capability awaiting a client: every bootstrap token minted via
  `GET /developer/cli-bootstrap-token` carries the new capability unconditionally, the same way it has
  always unconditionally carried the cross-context app-context-create capability — so a direct API
  caller (no CLI required) can register a tenant-wide namespace as soon as this ships. The
  CLI/blueprint-schema convenience surface — a `tenantWide` blueprint field
  (`@vectros-ai/blueprints` 0.14.0) and an `--allow-tenant-wide-namespaces` flag (`@vectros-ai/cli`
  0.17.0) that lets `vectros bootstrap` request this deliberately and refuse otherwise — is **already
  shipped**, not pending; it's a **client-side courtesy**, not the authorization boundary, since the
  OWNER check on the bootstrap mint is and remains the actual control.

- **New endpoint: `POST /v1/auth/token/assume`** (supersedes the never-released `/switch` design
  from this same still-open section — see below). Re-mints the presented `st_*` scoped token with
  exactly one `identity.<namespace>` value changed — for a caller whose ROLE explicitly grants
  assuming more than one value in that namespace (for example, an invited hr-admin or a multi-org
  case-handler) and needs to change which one new writes are placed under. The request body names
  exactly one namespace in canonical `scope:<namespace>` form, e.g. `{"scope:org": "orgB"}`, and the
  value must be a plain literal — never a `${{ ... }}` placeholder. `st_*`-only — a root API key or
  scoped API key gets `403`; neither needs this. The requested value must be explicitly granted by the
  role's new `assumable` field (below) for that namespace — a POINT check against the one value
  requested, deliberately separate from what the role's `data_scope` permits reading or writing:
  holding broad `data_scope` reach in a namespace does NOT by itself grant assuming any value in it.
  The re-minted token's `exp` is identical to the presented token's — this call can never extend a
  session's life, only change which value is active within the window the original exchange already
  established. Every other claim (including `data_scope` itself) is preserved verbatim. A fresh,
  independently-revocable `jti` is stamped on every call, and the token also carries a `root_jti`
  revocation-lineage claim so revoking the FIRST token in a chain closes every value ever assumed from
  it at any depth. Uses the ordinary `{"message":...}` error shape, not the OAuth envelope
  `POST /v1/auth/token/exchange` uses.

  _(Design note, not part of the shipped surface: an earlier `/switch` draft of this endpoint computed
  entitlement as a union across every clause's `data_scope` — a value legitimately present in an
  unrelated, possibly read-only clause silently widened a different clause's write authority once
  assumed, and re-assuming the same value could launder a revoked token. That design never shipped;
  `/assume`'s point-checked `assumable` grant plus `root_jti` lineage close both holes from the start.)_

- **`Role`'s `assumable` field — the entitlement grant `POST /v1/auth/token/assume` reads.** A new,
  optional, role-level map (unlike `scopes`' per-clause `data_scope`) naming which values, per
  `scope:<namespace>`, a holder of that role may assume: `{"scope:org": ["org_engineering",
  "org_sales"]}`. The principal (`userId`) can never be named. Each value accepts a
  plain literal, `${{ under.self.userId }}`, or `${{ member.scope.<namespace>[:level] }}` — never
  `${{ under.self.scope.<namespace> }}` (see above), a bare `${{ self.<dim> }}`, or `${{ any }}}`, all
  rejected at authoring time. Omitting the field grants no assumption of anything, the safe default.
  Present on `Role` create/update/upsert, and directly authorable on an `AccessProfile` too — but only
  alongside that profile's own inline `scopes`; setting it alongside `roleId`/`roleIds` is rejected
  outright with `400` (it would never be read — a role-composed profile's `/assume` entitlement comes
  from each referenced role's own `assumable`, checked live against the caller's current roles, never
  from anything stored on the profile). A non-root credential authoring an `assumable` grant broader
  than its own live composed reach gets `403`. Blueprint-loader and CLI schema support for authoring
  this field are **already shipped** — a top-level `roleAssumable` map (`@vectros-ai/blueprints`
  0.15.0), wired into role creation and `vectros role get`/`blueprint plan` (`@vectros-ai/cli` 0.18.0,
  which requires this SDK build for `Role.assumable`) — in addition to the direct `PUT`/`POST
  /v1/roles` call.

### Fixed

- **The last-OWNER delete refusal no longer names an endpoint the caller can't reach.** `DELETE
  /v1/users/{id}`'s `409` when deleting the account's last OWNER pointed at `POST
  /developer/account-owner` — a route gated on the developer-portal session surface, not reachable by
  the partner-API root key that hits this endpoint (nor by a human without developer-portal access).
  The message no longer names it; it directs you to contact support until a partner-API-reachable
  transfer path exists. Message text only — the `409` status and refusal condition are unchanged.

### Changed

- **Clarified `externalSubject` on `PUT /v1/users/{id}`'s invitation-activation request.** No
  behavior change — the description now states plainly that this field is stored exactly as sent,
  with no normalization, and is informational record-keeping only — not an authentication binding
  and not a correlation key (use `externalId` for your own correlation/bookkeeping needs). Setting
  it never grants sign-in to the Vectros DevPortal, Admin App, or other Vectros-hosted web apps, and
  it separately does not enable — or deduplicate against — a later sign-in via
  `POST /v1/auth/token/exchange` either. If you want an invited person to be able to sign into any
  Vectros-hosted surface, send them the invitation directly (the invite link for dev-portal/web-app
  access, or the raw `inviteToken` for a party redeeming it themselves via token exchange) rather
  than calling this endpoint on their behalf — it exists for a caller running their own backend who
  wants Vectros to record which of their own users a given account corresponds to.

- **`GET /v1/users/{id}/versions`'s `404` now documents both causes.** No behavior change: the `404`
  fires both when the user doesn't exist and when a context-confined credential's own app context holds
  no access profile for it, and the two are indistinguishable.

- **`POST /v1/users`'s collision-echo and `?upsert=true` are now context-mediated.** A context-confined
  credential colliding on `externalId` with a user outside its own app context receives the uniform
  `400` ("already exists") instead of being shown that user's data (plain create) or allowed to
  overwrite it (`?upsert=true`). This is in addition to the existing rule that a plain collision echo
  also requires the `users:r` scope. A root API key, or a credential with cross-context reach, is
  unaffected.

- **The CLI bootstrap credential can be echoed a user across app contexts on a plain (non-upsert)
  collision.** A narrow carve-out for the platform-minted `provisioning:c` capability — only
  `vectros bootstrap` mints it, and no partner-authored role can acquire it — so a blueprint re-applying
  against a service principal in a non-`default` app context isn't blocked by its own bootstrap
  credential. Does not extend to `?upsert=true` or to `GET`/`PUT`/`DELETE`.

### Added

- **End-subject erasure requests are live.** `POST /v1/erasure-requests` + `GET /v1/erasure-requests/
  {id}` — previously reserved as a `501` stub with the request/response shapes already frozen — now
  perform a real, asynchronous right-to-erasure sweep for a single subject (`user`, or an ownership
  namespace such as `org`/`client`/a custom namespace you've registered). `POST` returns `202` with a
  `requestId`; poll `GET .../{id}` until `status` reaches `completed` for the verifiable completion
  certificate (contexts swept, per-context deletion counts, dangling-reference and shared-row reports)
  or `failed`. Requires the root API key, exactly as documented since the reservation — a scoped
  credential still gets `403`. The request/response shapes themselves are unchanged from the reserved
  contract; only the previously-`501` behavior is now real. See `POST /v1/erasure-requests`'s
  description for the full semantics (what "solely owns" means for co-owned data, what's reported vs.
  erased for shared/dangling references, and the `auditDisposition` options).

- **App contexts can now declare a per-principal metering axis.** `PUT /v1/app-contexts/{id}` accepts
  three new optional fields: `meteringAxis` (`user`, or `scope:<namespace>`) declares what "per-
  principal" means for that context; `principalBurstLimit` sets a per-principal, per-minute request
  cap; `principalUsageCap` sets a per-principal, per-billing-period operation cap. All three are
  additive, `null`/omitted by default (no behavior change unless set), and only take effect for an
  account with the corresponding feature enabled. `GET`/list app-context responses echo all three.

- **Batch get-by-id is live.** `POST /v1/records/batch-get` — previously reserved as a `501` stub with
  the request/response shapes already frozen — now fetches multiple records by id (`ids`,
  maximum 100) in one call. Returns only the records you can access: any id that does not exist,
  belongs to another account/AppContext, or is outside your token's scope is silently omitted, with no
  per-id existence signal (matching the not-found behavior of the single-record GET, not a difference
  worth documenting further — that indistinguishability is the point). Payloads are hydrated the same
  way a by-id GET hydrates them. Requires the `records:r` scope, same as before. The request/response
  shapes are unchanged from the reserved contract; only the previously-`501` behavior is now real.

## 0.40.0 — 2026-08-19

### Added

- **Scope clauses can grant named platform capabilities.** Every scope clause — in a role, an access
  profile, or an invitation — accepts an optional `granted_capabilities` array alongside `allowed_actions` and
  `data_scope`. A capability names a bounded effect that reaches across a partition boundary, which the
  `resource:ops` verb grammar cannot express.

  Four names are accepted in this release. **`member-lifecycle`** — create and remove identities in your
  tenant. Adding someone who already has an identity here resolves them by email or `externalId` as part
  of that operation; it does not let the credential look an identity up on its own.
  **`forensic-read`** — read the access log across every context in your tenant, by the credential that
  performed each read (*"what did key K read"*). **`context-directory-read`** — read your tenant's app
  context directory (which contexts exist, their roles and access profiles, and which contexts a given
  principal is in) across every context, not just the credential's own; see the `GET
  /v1/principals/{id}/profiles` entry below for its concrete effect. It confers no data-plane reach —
  records, documents, folders, schemas and entities are outside it entirely — and no write.
  **`delegate-mint`** — mint a scoped API key (`ssk_*`) bound to a *different* principal than your own via
  `POST /v1/admin/keys/scoped`; see the entry below for the behavior it replaces. The list is
  closed and versioned with the API: a later release may accept further names, and a name this release
  does not accept is rejected when you author the clause rather than stored inert.

  Three rules govern the field, all fail-closed, and all worth knowing before you author one:

  - An **absent or empty** list grants no capability. Absence is silence, not a wildcard.
  - An **unrecognized name denies that whole clause** rather than being ignored — so a typo fails loudly
    at the first request instead of quietly downgrading the grant to its surviving verbs.
  - A **`"*"` in `allowed_actions` confers zero capabilities.** `"*"` is not a wildcard in this list, and
    is rejected as a name; name each capability explicitly.

  Each capability's reach is fixed by its own definition and the clause carries names only. Where a
  capability acts on individual records the clause's `data_scope` narrows it as well — but neither name in
  this release is per-record, so `data_scope` does not narrow either one.

  `member-lifecycle` is bounded by the app contexts the credential can reach. Grant it deliberately for
  that reason: its reach is any app context the credential can already administer. That is an ordinary
  grant where app contexts separate your *apps* — but not where you have modelled your own *customers*
  as separate app contexts.

  `forensic-read` is bounded by the **tenant**, not by the credential's app context — answering
  *"what did this key read"* crosses contexts by design, and it confers nothing on its own: the endpoint
  still requires the `access-log:r` action, so the effective grant is `access-log:r` **plus** this name.
  Grant it to an audit or support role.

  `context-directory-read` is likewise bounded by the **tenant**, not by the credential's app context —
  it is tenant-wide by design and not narrowed by the credential's own app context. Grant it only to an
  admin or support role, for the same reason as `forensic-read`.

  `delegate-mint` is also bounded by the **tenant**, not by the credential's app context: the credential
  it lets you mint literally *is* another principal, wherever that identity is later used, so it is
  tenant-wide by design in the same way `forensic-read` and `context-directory-read` are. The target must
  still already be a member of your own app context, and the minted key's effective scope can never
  exceed your own — this capability lifts only *who* the minted key may be, not *what* it may do. Grant
  it to an operator or automation role that provisions credentials on other members' behalf.

  **On self-signup, precisely — and this cuts one way, not all four:** a role carrying
  `forensic-read`, `context-directory-read`, or `delegate-mint` is refused when a self-signup policy
  creates a profile, so no *new* self-signup can be created against such a role. `member-lifecycle` is
  the deliberate exception — it is bounded by app context rather than the tenant, so a self-signup role
  may carry it; granting it to your default self-signup role is the intended way to let a founder sign
  up and then invite their own team. For the three refused names, the check runs at profile-creation
  time rather than continuously — so adding one of them to a role your users can **already** sign up to
  grants it to those existing members at their next token. Treat re-scoping such a role as the
  authoring decision it is.

- **`GET /v1/admin/logs` now returns a `delegationChain` field on each entry.** Present only for a
  request made under a delegate-minted `ssk_*` key (`delegate-mint`, above) — a JSON-encoded record of
  who delegated the credential. Absent (`null`) for the overwhelming majority of requests, which carry
  no delegation. Purely additive; no existing field changes shape or meaning. Separately, `?resource=`
  no longer accepts `clients`/`orgs` as filter values (`400` instead) — the `/v1/orgs` and `/v1/clients`
  routes were retired in an earlier release and no log row has ever carried either value.

- **`profiles:c`/`u`/`d` now accept an optional qualifier confining WHICH principal the grant may act
  on** — a literal `usr_<id>`, or the bare `self` sentinel. `self` matches only when the target of the
  request is the credential's own bound principal; it never widens a grant, and it is a plain literal in
  the qualifier position, not a `${{ }}` template. A bare, unqualified entry (`profiles:cru`) is
  unchanged and stays broad — it remains the context-admin grant. `profiles:r` does not accept a
  qualifier: `GET`/list routes are unaffected by this release. Example:
  `{"allowed_actions": ["profiles:u:self"], "data_scope": {}}` lets a credential update only its own
  access profile in a context, never another principal's.

### Changed

- **A namespace registration now declares where its entities — and its membership — live.**
  `/v1/namespaces` is TENANT-WIDE by default (the behavior you have today: the namespace's entities are
  shared by every app context, the way `org` and `client` are) and becomes CONTEXT-OWNED when you supply
  `?contextId=` on `POST /v1/namespaces`. A context-owned registration belongs entirely to that one app
  context: two different contexts can each register a `team` namespace with no collision, no shared
  configuration, and no visibility into each other — registering, reading, updating or deleting one
  context's `team` namespace never touches another context's. **A registration's `contextId` is fixed
  once set** — an entity's partition is part of its key, so a namespace cannot be re-homed after
  registration. The response carries `contextId` (`null` for a tenant-wide registration); there is no
  `placement` field.

  **`?contextId=` applies to every `/v1/entities/{namespace}` operation, not only creation** — `GET`
  (both the by-id form and every list and lookup mode), `PUT`, `DELETE`, `GET .../versions`, and `POST
  .../lookup` all take it. It is **required** for a context-owned namespace and **rejected** for a
  tenant-wide one, and a context-confined credential may only name its own context. A context-owned
  namespace's entities are invisible from the other contexts: a read naming context A never returns,
  updates or deletes an entity belonging to context B. **A context-confined credential explicitly
  naming a context other than its own is refused with `403`** before the lookup ever runs — it can
  never reach a sibling context's entities to receive a `404` for them. Within its own context
  (named or omitted), an entity that doesn't exist there answers `404`, same as anywhere else; an
  unconfined (root) credential naming a context an entity doesn't live in also gets `404`, since only
  confined credentials are restricted at context selection. Reading a namespace's own registration
  (`GET /v1/namespaces/{name}`, `GET /v1/namespaces`) is likewise confined: a credential may read only
  its own context's registration, or the tenant-wide one — never a sibling context's.

  **Writing a namespace registration — create, update, or delete — always requires a root API key.**
  The CLI bootstrap's provisioning capability may additionally CREATE one, confined to its own bound app
  context (the same confinement every other provisioning-gated write already has); it may not update or
  delete any registration, and it may not create one in a context other than its own.

  A schema whose `allowedSurfaces` is `["entity"]` is now written by an ordinary scoped credential in the
  caller's own context, where it previously required the root API key — so a blueprint can provision one.
  A schema binding the `user` surface is unchanged and still root-only: partner users are tenant-global,
  so their schema has one tenant-wide home. `GET /v1/schemas?surface=entity` reads the caller's context
  and the tenant-wide home together, newest context first, through a single cursor.

  **One naming collision to know about: a credential confined to an app context literally named
  `default` does not get a private, context-owned entity schema.** `default` is also the name of the
  tenant-wide home that `user`-surfaced schemas use, so a credential confined to that context writes its
  entity-surfaced schema into the shared tenant-wide home instead — visible from every other context
  too, not private to `default`. It is still gated on the ordinary `schemas` scope, so this doesn't widen
  who can write; it only means `default` can't hold a context-private entity schema of its own. If you
  need a private entity-schema home, use an app context named anything other than `default`.

- **A namespace registration can declare where its MEMBERSHIP lives, resolved per request.** A
  registration may additionally carry `membershipRecordType` / `membershipTargetField` (which record
  type and field name the grant, together, all-or-nothing) and, optionally, `membershipLevelField` +
  `membershipLevels` (a closed set of tier labels, e.g. `admin`/`viewer`). A tenant-wide registration
  also needs `membershipContextId` — which app context holds the grant records — since a context-owned
  registration's grants live in its own context by construction and need no separate field.

  Declaring a membership backing grants nothing by itself. A `data_scope` clause opts in explicitly with
  `${{ member.scope.<namespace> }}` (every membership, any level) or `${{ member.scope.<namespace>:<level>
  }}` (one level only) — the bare and level-qualified forms are independent grammar, and authoring a
  level the namespace has not declared is rejected at write time, before it can ever resolve to an
  inert, silently-widening clause later. Resolution happens **once per request**, never at mint time, so
  a revoked membership takes effect on your very next request rather than waiting out the token's
  lifetime.

- **`org` and `client` are ordinary namespace registrations — reserved names, not built-ins.** They were
  previously entity-backed by virtue of their names, with no stored row behind them. Now they are real
  registrations, on the same terms as any namespace you register yourself: they appear in
  `GET /v1/namespaces`, and can be updated or (once empty) deleted like any other. No request or response
  shape changes. The retired `/v1/orgs` and `/v1/clients` routes remain retired.

  **Register both explicitly before creating an `org`/`client` entity** — `POST /v1/entities/org`/`client`
  returns `400` ("not entity-backed") until you do. Register each with a root API key:
  `POST /v1/namespaces {"namespace": "org", "entityBacked": true, "specificityRank": 1000}` and the same
  for `client` at `specificityRank: 2000`. This is a one-time step per account; once registered, both
  behave identically to any other namespace.

- **`GET /v1/principals/{id}/profiles` no longer answers cross-context unconditionally.** Previously any
  credential holding `profiles:r` received the requested principal's access profiles across every app
  context in your tenant, regardless of which context the credential itself was confined to. Now:
  looking up your **own** principal — or holding the new `context-directory-read` capability (see
  Added, above) — still returns the profiles across every context you administer. A context-confined
  credential looking up a **different** principal instead sees only that principal's profile in the
  credential's own context, at most one result. **If you rely on `profiles:r` alone to enumerate another
  principal's access across contexts, grant `context-directory-read` to the calling role as well.**

- **Reading or deregistering a trusted issuer is now confined to the caller's own app context.**
  `GET /v1/auth/issuers/{issuerId}`, `GET /v1/auth/issuers`, and `DELETE /v1/auth/issuers/{issuerId}`
  previously let any credential holding the root API key or the CLI bootstrap's provisioning capability
  read or deregister an issuer registered in **any** app context in the tenant. A credential confined to
  one context (the bootstrap token) now sees and may only delete issuers registered in its **own**
  context; naming or listing a sibling context's issuer now behaves exactly as if it doesn't exist
  (`404`). A root API key is unaffected. **Separately**, `POST /v1/auth/issuers`'s idempotent-echo no
  longer returns a sibling context's full issuer configuration when a confined credential's request
  happens to collide on `issuerId` with a registration it doesn't own — that collision now fails with
  `400` instead of echoing the other context's `jwksUri`/`audience`.

- **Internal:** most writes on the app-context, access-profile, role, issuer, namespace and user
  surfaces now pass through a shared partition check before they are persisted. No request or response
  shape changes — the check confirms, at the moment a row is written, that it belongs to the tenant and
  app context the caller is already confined to. It changes no outcome on its own; the
  behaviour changes in this release are called out separately below.

- **Registering a trusted issuer is now confined to the caller's own app context.**
  `POST /v1/auth/issuers` previously let a credential carrying the CLI bootstrap's provisioning
  capability register an issuer against *any* app context in the tenant. It now requires `contextId` to
  be the context that credential is bound to; naming another one returns **403**, exactly as every other
  context-bound operation already did. A root API key is unaffected and may still register an issuer for
  any of its contexts.

  This matters because a blueprint's `issuers` block is applied with the bootstrap credential, and a
  blueprint may come from outside your organization. Without this, a pack could name an app context it
  had nothing to do with and attach an identity provider to it.

  **If you apply a blueprint whose `issuers` target a context other than the bootstrap credential's,
  `@vectros-ai/cli` 0.16.0+ handles it transparently** — the bootstrap credential re-mints itself,
  pinned to the blueprint's own context, specifically to register those issuers; no blueprint authoring
  change is needed. If you register issuers directly against this endpoint outside the CLI, the
  restriction is real and by design: the credential `vectros bootstrap` mints is bound to the `default`
  app context, so a request naming a different context returns `403` — register those issuers with a
  root API key instead, or apply the blueprint with a credential bound to the target context.

- **A scoped credential can no longer tell *why* an invitation collided on an email address.**
  `POST /v1/users/invite` returns 409 when the address is already taken. Previously one of the three
  causes — the address belonging to an identity elsewhere in your account — returned a distinct body
  carrying an `email_already_associated` code, so a caller could learn whether an arbitrary address
  belonged to someone in your organization by inviting it and reading the error.

  All three causes now return the **same 409** to a scoped credential (`ssk_*` / `st_*`). A **root API
  key still receives the structured `email_already_associated` body**, since it can list its own users
  directly and so learns nothing new from it. If you branch on that error code, do so on a root key, or
  treat any 409 from this endpoint as "address unavailable".

- **Minting a scoped key no longer reveals whether a user id exists elsewhere in your tenant.**
  `POST /v1/admin/keys/scoped` requires the named user to have an access profile in the named app
  context. Previously a user id that existed nowhere returned `404 User not found: …`, while a user id
  that existed but had no profile in that context returned `404 AccessProfile not found for user …`.
  A credential confined to one app context could tell those apart, and so could test arbitrary user ids
  against your whole tenant.

  Both cases now return the **same** `404`, the one naming the access profile — which keeps the
  actionable part (*create one via `POST /v1/app-contexts/{contextId}/profiles` first*). **If you branch
  on the text of this 404, match on the access-profile message.** A user id that exists and has a
  profile in that context is unaffected.

- **Minting a scoped key bound to a *different* principal now requires the `delegate-mint` capability.**
  `POST /v1/admin/keys/scoped` lets a caller name any `userId` with an access profile in the named app
  context, and the minted `ssk_*` resolves to that profile's identity — every `${{ self.* }}` clause it
  carries evaluates as the named user, not the caller. Previously the `keys:c` action plus the existing
  scope-subset check (the minted key's scope may not exceed the caller's) was sufficient on its own; a
  scoped credential holding `keys:c` could mint a key that *is* any co-member of its own app context, with
  no separate authorization for choosing someone other than itself.

  **Minting a key bound to your own principal is unaffected — no capability is needed.** Minting a key
  bound to someone else now additionally requires `granted_capabilities: ["delegate-mint"]` on the calling
  credential; without it the request now returns `403` instead of succeeding. A root API key is unaffected
  and may continue to mint against any principal. **If your integration uses a scoped credential to
  provision `ssk_*` keys on behalf of other members, grant that credential's role `delegate-mint`.**

- **A context-confined credential's reach on the `users` resource is now mediated by access-profile
  membership in the credential's own app context.** Previously a scoped credential holding
  `users:r`/`u`/`d` could read, update, or delete *any* user in your tenant, regardless of app context —
  `users` had no per-context reach limit, unlike most other resources. Now, for a context-confined
  credential: `GET`/`PUT`/`DELETE /v1/users/{id}`, `GET /v1/users` (list), `GET /v1/users?externalId=`,
  and `GET`/`POST /v1/users/lookup` only reach a user who holds an access profile in the credential's own
  app context. A user who exists but isn't a member of that context answers exactly as if it didn't exist
  — `404` on the id-addressed routes, silently absent from a list/lookup result — the same
  uniform-not-found shape used elsewhere on this API. **If your integration uses a context-confined
  credential to manage users across your whole tenant, this is a live break: grant it cross-context
  reach, or scope the workflow to per-context member management.** A root API key, or a credential with
  cross-context reach, is unaffected.

- **`DELETE /v1/users/{id}` now refuses to delete your account's last OWNER.** Deleting the OWNER user
  that removes your account's only remaining owner access now returns `409` instead of succeeding — an
  account must always retain at least one owner; contact support if you need to change who holds it. If
  your account has more than one owner-eligible principal, or you are deleting a non-owner (`SUB_USER`)
  identity, this change has no effect on you.

  **Separately, called by a context-confined credential, it also refuses when the user has access in an
  app context other than the caller's own** (`409`) — deleting a user's identity removes it everywhere,
  so a context-confined credential can no longer delete a user out from under access it can't see.
  Remove the user from your own context first (`DELETE /v1/app-contexts/{contextId}/profiles/{principalId}`),
  or use a credential with cross-context reach to delete the identity outright.

- **`POST /v1/auth/token` returns the standard error body on its `404` and `400` responses.** This route
  assembled those bodies by hand, so they carried only `{"message": …}` with no `requestId`, and the
  response set no `Content-Type`. Every `404` (app context not found) and `400` (a malformed body, a
  missing or invalid `scope`, an unrecognized `userId`/entity id, or an `expiresInSeconds` over the
  1-hour maximum) now returns the same envelope as every other error on the API — `requestId` included,
  `Content-Type: application/json`. Status codes and message text are unchanged; if you parse these
  bodies generically, nothing breaks.

- **`POST /v1/auth/issuers` now enforces one active IdP per app context.** Registering a second issuer
  against a context that already has an active one now returns `400` instead of succeeding — a context
  has exactly one active IdP at a time. Registering the SAME issuer against several DIFFERENT contexts
  (each via its own `audience`) is unaffected and remains the supported way to serve more than one
  context from one IdP. If your integration only ever registers one issuer per context, this change has
  no effect on you.

- **`DELETE /v1/auth/issuers/{issuerId}` now refuses to deregister an issuer that has ever bound a
  user.** Previously unconditional — deleting an issuer registration succeeded even if a prior
  self-signup or accepted invite had already created a user via it, silently orphaning that user's
  future ability to obtain a new token. It now returns `409` in that case; deactivate the affected users
  first, or register a replacement issuer, before removing this one. An issuer that has never
  successfully completed a `POST /v1/auth/token/exchange` can always be deregistered.

- **`POST /v1/auth/token/exchange` accepts an optional `context_id` field.** If your issuer is
  registered against more than one app context (each via its own audience), `context_id` selects which
  one to target, disambiguating a `subject_token` whose `aud` claim could otherwise match more than one
  of your registered contexts. Omit it when your token's `aud` claim matches only one registered context
  — the common case, and the field's addition changes nothing about that path. Naming a context this
  issuer is not registered against is refused identically to an unrecognized issuer (`404`, no
  distinguishing information).

- **`GET /v1/usage` no longer returns account-wide totals to a context-confined credential.**
  Previously the `credits`, `search`, `documents`, `records`, `identity`, `inference`, `readAccess`, and
  `tenants` sections always summed your **whole account** — both the live and test environment, every
  app context — regardless of which app context the calling credential was confined to; only the
  `contexts` breakdown itself respected `contextId`. A credential confined to one app context now sees
  every section narrowed to that context alone, and the environment (`tenants.live` / `tenants.test`)
  its context is **not** bound to is now `null` rather than the other environment's real totals. A root
  API key, or a scoped credential with cross-context reach, is unaffected — you continue to see your
  full account-wide totals, and `contextId` continues to work as a display-only filter on the `contexts`
  breakdown. **If your integration reads top-level usage totals from a context-confined token, those
  numbers now reflect only that context** — read `contexts[]` if you need the full picture, or use a
  credential with cross-context reach. Supplying `contextId` on a context-confined credential now
  requires it to name that credential's own context; naming another one returns `403`.

  **Two narrower exceptions, worth knowing precisely rather than assuming "everything narrows":**
  `reads.calls.used` and `reads.dataOut.bytes` — the raw call-count and egress-byte counters — have no
  per-app-context breakdown to narrow to at all (they're metered per account, not per context), so a
  context-confined credential sees `0` for both rather than a narrowed figure; the corresponding
  **charge** fields (`reads.calls.overageCredits`, `reads.dataOut.overageCredits`) *do* narrow correctly,
  since overage is charged per context. Don't read `reads.calls.used == 0` as "this credential made no
  calls this period" — it means "this dimension isn't visible at context granularity." Separately,
  `credits.limit` continues to reflect your whole plan's ceiling (a property of your plan, not of any
  one context's usage); only `credits.used`/`credits.remaining` narrow, so `credits.remaining` may
  overstate the account's true remaining room for a context-confined credential.

- **`POST /v1/app-contexts/{contextId}/profiles` now requires a `usr_` principal to be a real user.**
  Previously the endpoint validated only the *format* of `principalId`, so a request naming a user that
  does not exist — or one that exists in a different tenant — created an access profile anyway. Those
  profiles conferred no access (a token mint resolves the user before it reads the profile), but they
  accumulated with no limit and there was no way to notice. A `usr_`-prefixed `principalId` that does not
  name a live user in your tenant is now rejected with `400`, and no profile is created. **Create the user
  first, then grant it an access profile.** `key_` principals are unaffected. `?upsert=true` is checked the same way,
  because it rewrites a grant. The plain idempotent-collision response is NOT: it writes nothing, so an
  access profile whose user was deleted still returns its existing row rather than an error, and can
  still be removed with `DELETE`.

  If your credential holds `users:r` the error names the reason; otherwise the response is deliberately
  indistinguishable from the malformed-`principalId` rejection, so a credential that cannot list your
  users cannot use this endpoint to discover which user ids exist.

### Fixed

- **The Vectros-hosted admin app and CLI bootstrap credentials can mint a `delegate-mint`-requiring key
  again.** When `delegate-mint` shipped above, the two first-party credential shapes that fill it out on
  your behalf — the admin app's session token and the CLI's per-context bootstrap token, both minted
  through `GET /developer/scoped-token` — were not given the capability, so binding a scoped key to
  anyone but yourself through either of them returned `403`. Both now carry `delegate-mint` by default,
  restoring the behavior that existed before this release: minting an `ssk_*` bound to a different
  member of your own tenant, from either surface, works as it did. **No action needed** — this affects
  only Vectros's own first-party credentials, never a role or access profile you authored yourself; the
  entry above (*"grant that credential's role `delegate-mint`"*) still applies unchanged to your own
  integrations.

- **Revoking a leaked CLI bootstrap token now takes effect immediately on the writes its bootstrap
  capability gates — issuer registration and namespace registration (see Changed, above), and app-context
  creation.** The capability check previously read only the token's declared scope, not whether it had
  since been revoked, so a bootstrap token you revoked in response to a known leak could still succeed at
  these writes — including registering a trusted issuer with an attacker-controlled `jwksUri` — for the
  remainder of its normal lifetime (up to one hour) after revocation. Revocation status is checked on
  every request now, closing that window. No action needed — this only affects what a token could do
  *after* you had already revoked it.

- **`POST /v1/entities/{namespace}/lookup` and `GET /v1/entities/{namespace}/{id}/versions` now honor
  their own `?contextId=` reliably.** On a Lambda container reused across requests, either route could
  read the `contextId` left by a *prior*, unrelated request instead of its own. The effect differed by
  caller shape and by whether the container had ever served this route before:

  - An **unconfined (root or provisioning) caller** could have a lookup silently resolve against the
    wrong app context instead of the one actually named.
  - A **context-confined caller was not immune** — a stale, non-null leftover value could make a
    request that supplied *no* `contextId` at all fail with
    a spurious `403` (or, on a tenant-placed namespace, a spurious `400` naming a `contextId` the request
    never sent), because the stale value was still compared against the caller's own context even though
    the caller's own context is resolved independently.
  - On the **very first** request either route ever served on a given container, the field had never been
    set at all and the request failed with a `500` rather than resolving against the wrong context.

  If you have observed an unexplained `403`/`500` on these routes, or a lookup answer for the wrong app
  context, this is fixed; no request shape changed.

## 0.39.0 — 2026-08-07

### Added

- **Register a trusted external IdP issuer.** `POST /v1/auth/issuers` registers an
  `(issuer, audience)` pair your tenant trusts. `GET /v1/auth/issuers/{issuerId}` and
  `GET /v1/auth/issuers` read back a single registration or the full tenant list (a
  `{data, nextCursor}` envelope).

  Registration is idempotent by `issuerId` within your tenant — registering the same `issuerId`
  again returns the existing registration unchanged (`200`) rather than erroring. The
  `(issuer, audience)` pair itself must be globally unique across tenants; a second registration
  naming a pair already claimed elsewhere is rejected (`400`).

  This is a provisioning-time operation — it requires a root API key or the credential minted by
  `vectros bootstrap`, and there is no grantable scope for it.

- **Exchange a third-party identity provider token for a Vectros token.** `POST
  /v1/auth/token/exchange` implements [RFC 8693 OAuth 2.0 Token
  Exchange](https://www.rfc-editor.org/rfc/rfc8693): trade a JWT issued by an identity provider
  you've registered (above) for a Vectros `st_*` scoped bearer token — no Vectros credential
  required to call it. The exchanged token's permissions are resolved entirely server-side from
  the matched user's access profile; the endpoint accepts no caller-supplied scope, so a
  compromised or malicious caller can never request broader access than that user already has.

  This closes the loop opened by issuer registration: once you've registered your identity
  provider, your own users can authenticate directly against it and exchange their token for a
  Vectros credential, with no backend service of yours minting tokens on their behalf.

  Uses the standard OAuth error envelope (`{"error": ..., "error_description": ...}`, per [RFC
  6749 §5.2](https://www.rfc-editor.org/rfc/rfc6749#section-5.2)) rather than this API's usual
  `{"message": ...}` shape, since its caller is generic OAuth tooling rather than the Vectros SDK.

- **Access profiles now inline the principal's email.** `AccessProfileResponse` (returned by every
  `/v1/app-contexts/{contextId}/profiles*` read/write, and `GET /v1/principals/{principalId}/profiles`)
  carries a new `email` field — the `usr_` principal's email when it resolves to a user in your
  tenant AND your token also holds `users:r`, absent otherwise (a `key_` principal, a `usr_`
  principal with no matching user, or a token that holds `profiles:r` alone). `GET
  /v1/app-contexts/{contextId}/profiles` resolves the whole page in one batched lookup, closing the
  N+1 a "who's on my team" UI previously had to pay per principal.

- **Check whether a user exists by email, scoped to one app context.** `GET /v1/users/exists-by-email`
  answers "does a user with this email hold an ACTIVE access profile in this app context" — a narrow
  existence check (`{exists, userId, status}`), not a lookup of the full user record. `exists` is
  `false` both for an email that was never a member of the context and for one whose access there has
  been suspended. The answer is scoped to the `contextId` you supply and does not reveal whether the
  email exists anywhere else in your tenant or account. Useful for resolving an email to a `userId`
  (for example, before referencing an existing member in another operation) without paging through the
  full context member list. Requires the `users:r` scope.

- **Self-service signup — let end users create their own account, no invite required.** Registering an
  issuer (above) can now also declare `selfSignupPolicies`: a list of `{signup_type, role_id}` pairs.
  When a first-time `POST /v1/auth/token/exchange` caller matches no existing user and presents no
  invite, but names a configured `signup_type` (or the registration has exactly one policy and the field
  is omitted), a brand-new active user is created and bound to that policy's role — no admin, no invite
  email, no separate provisioning step.

  `signup_type` is a plain value the caller supplies; it does not need to come from your identity
  provider. This is deliberate and safe: every policy entry is, by construction, something you already
  decided any caller may have — that's what "no invite required" means — so naming a different
  configured entry than your frontend intended isn't a privilege escalation, only a different (equally
  pre-approved) role. To enforce that, no `selfSignupPolicies` entry may ever target a role that carries
  elevated (provisioning, wildcard, or key/profile/user/context-management) scope — checked when you
  register the policy and always re-checked, unskippable, at the moment a caller actually signs up.

  **Read this before enabling it: "any caller" means literally anyone who can obtain a token from the
  registered issuer for the registered audience** — not "any employee," regardless of how narrow your
  frontend UI makes signup_type look. If your issuer is a typical IdP tenant with open or social-login
  signup, that's the entire internet, not your internal directory. Self-signup is only as narrow as who
  can authenticate against your identity provider — restrict that there, not here, if you need it
  narrower. A good fit: an internal tool backed by an IdP tenant only your organization's members can
  authenticate against, where any successfully-authenticated caller becoming a member is exactly what
  you want.

- **`DELETE /v1/auth/issuers/{issuerId}`** deregisters a trusted issuer. Requires a root API key or the
  bootstrap's provisioning capability, same as registering one. Existing user accounts a prior
  self-signup or invite created via that issuer are not deleted or modified, but they lose the ability
  to obtain a *new* token this way — every exchange call against the deregistered issuer 404s. Register
  a replacement issuer (or restore this one) before affected users' current tokens expire if you want
  their access to continue uninterrupted.

### Changed

- **Scoped tokens (`st_*`) now last at most 1 hour.** `POST /v1/auth/token` previously accepted an
  `expiresInSeconds` of up to 86400 (24 hours); the maximum is now **3600**, and a larger value is
  rejected with a `400`.

  **This breaks no caller who did not choose a TTL.** 3600 was already the default on this endpoint,
  and is the fixed lifetime of a token from `POST /v1/auth/token/exchange` — so the change removes only
  the ability to opt *above* the default. If you were explicitly passing a value over 3600, lower it to
  3600 (or omit the field). A scoped token is a bearer credential that carries its permissions inline,
  and shortening its life is the most direct way to bound what a leaked one can do.

- **`GET /v1/ping` now reports a scoped token's own id in `principalKeyId`.** For an `st_*` credential
  this field is the token's JWT id (`jti`) — unique per mint, so two tokens issued for the same user
  report different values. It previously echoed the bound user id (or, for a token minted with no user
  identity, the id of the API key that minted it), which never matched this field's documented meaning.

  `principalKeyId` is unchanged for `sk_*` and `ssk_*` keys, where it remains the key id. If you are
  using this value to identify the *user* behind a scoped token, read the user id from your own token
  claims instead — this field identifies the **credential**. Tokens minted before this release carry no
  `jti` and continue to report the previous value until they expire; if you minted one with a long
  `expiresInSeconds` shortly before upgrading, that can be up to 24 hours, since the new cap applies
  only to tokens minted after it takes effect.

- **Clearer error documentation on three endpoints, with no behaviour change.** Creating or updating an
  access profile now documents that `roleId` must name a role that already exists in the app context
  (a `400`), and the update endpoint documents its `400` responses at all. Minting a scoped API key
  (`POST /v1/admin/keys/scoped`) now documents that `keys:c` alone is not sufficient for a scoped
  caller: the referenced profile's permissions may not exceed your own, and you may only mint against a
  profile whose identity values you hold yourself. All three were already enforced; only the
  descriptions were incomplete.

### Removed

- **The `create_own_scoped_key` literal `allowed_actions` verb has been removed.** It was never wired to
  any enforcement path — no handler ever checked for it — so authoring a scope with this literal granted
  no actual capability beyond what was otherwise already true of the credential; the 0.33.1 entry
  describing it as a valid, working option was inaccurate for the entire time it stood. Correcting the
  record here rather than editing that historical entry, which stands as released. An `allowed_actions`
  entry naming `create_own_scoped_key` now fails author-time validation with a 400, same as any other
  unrecognized colon-less string (e.g. bare `read`); the compact `resource:ops[:qualifier]` form and `*`
  remain the only grantable shapes. If you were authoring this literal, remove it — it was not doing
  anything for you regardless.

## 0.38.0 — 2026-07-29

**Upgrading — three things need action; everything else is additive.**

1. **Page on the cursor, not the page size.** Keep requesting while `nextCursor` is non-null and stop
   only when it is null. A short or empty page does **not** mean you are done. If you built a
   `startFrom` yourself from a record id, that now returns `400` — echo back `nextCursor` verbatim.
2. **`lookupFields[].fieldName` is now optional**, so typed SDK clients that read it must handle its
   absence. It is the only change in this release that can stop a client compiling.
3. **A cursor issued before this release is refused** with `400`; start the query again. This affects
   only a page turn that straddles the deploy.

### Added

- **Look records up by several fields at once.** A schema's `lookupFields` entry can now name two or
  three fields together instead of one, and the lookup then matches all of them — *"every record where
  `status` is `open` **and** `area` is `billing`"*, exact, completely, in a stable order, with the same
  cursor pagination every other lookup has. Declare it with `fieldNames` in place of `fieldName`:

  ```json
  { "lookupFields": [{ "fieldNames": ["status", "area"] }] }
  ```

  Query it by joining the field names with commas as `field`, plus one value per field in the new
  `values` parameter — repeated on `GET`, or an array in the body on `POST`:

  ```
  GET /v1/records/lookup?type=ticket&field=status,area&values=open&values=billing
  ```

  `values` is a **new, optional** parameter that sits alongside `value`; supply one or the other. A
  single-element `values` is exactly `value`, so nothing you already send changes meaning.

  **The order of `fieldNames` matters, and it is not something you can change later.** Re-declaring the
  same fields in a different order creates a *separate* lookup, indexed independently and at its own
  index cost, matching only records written after you declare it — it does not reorder the original. So
  choose the order deliberately: you can match on the first field alone, the first two together, and so
  on — any leading run of the list — but never on a later field by itself. Declare a separate lookup for
  that.

  **Supplying fewer values than the lookup declares is allowed, and returns the records grouped by the
  fields you left unspecified.** Matching only `status` on a `[status, area]` lookup returns every `open`
  record, ordered so that records sharing an `area` come out together. `sortBy` orders records *within*
  each group, not across them — so if you need a single overall ordering, or a `sortFrom`/`sortTo`
  window, supply a value for every field.

  Available on record lookups. A lookup over several fields cannot set `unique` (tuple-uniqueness is not
  enforceable) or `rangeEnabled` (it is an exact-match index — declare the range lookup separately), and
  its schema must list `record` as its only entry in `allowedSurfaces`. Document, user and entity lookups
  match a single field and are unchanged.
- `users:crud` to the standard scope-action catalog, so the `users` verbs are selectable in the
  developer portal's scope editor.
- **`Vectros-Version` request header — pin the response shape your client expects.** Send
  `Vectros-Version: 2026-08-01` to lock the shape of what you get back: fields, envelope, pagination,
  enum values, error bodies. It never affects behaviour — authorization, tenant isolation, quota
  enforcement and security fixes are identical under every version.

  **Sending nothing changes nothing.** Requests without the header are served exactly as they are today.
  Responses echo `Vectros-Version` so you can confirm which shape you received; a version we do not
  publish returns `400` with `errorCode: UNSUPPORTED_WIRE_VERSION` and the list of supported versions.

  `2026-08-01` is the only published version and describes the API as it behaves today. Versions are
  supported for 12 months after a successor is published, and deprecated versions carry `Deprecation`
  (RFC 9745) and `Sunset` (RFC 8594) headers first.

  Two limits to know: the streaming endpoints (`POST /v1/rag`, `POST /v1/chat`,
  `POST /v1/documents/{id}/ask`) ignore the header for now — note `POST /v1/documents` honours it while
  `POST /v1/documents/{id}/ask` does not — and the SDKs do not send it yet, so SDK calls use your
  account's default. Both land before a second version is published.

  **`Vectros-Version` is not the SDK version.** The SDK version moves on any change to the API surface,
  including additive ones like a new endpoint or field. A new `Vectros-Version` is published only when
  the shape of an existing response changes in a way that would break a client. Upgrading your SDK
  therefore does not change the response shape you are served — this header does.
- **`sortFrom` / `sortTo` on record lookup.** An exact-`value` lookup on `GET` or
  `POST /v1/records/lookup` can now be narrowed to a window of the lookup field's sort key — for
  example, one session's records from a given timestamp onward — instead of paging the whole match.
  Both bounds are inclusive, either may be given on its own, and they are expressed in the sort
  field's own units (epoch milliseconds when the lookup sorts by `createdAt` or `lastUpdated`).
  Records with no value for the sorted field are never included in a bounded window — they sort
  ahead of every record that has one — so the sorted field does not need to be `required`.
  Results page with the usual `{data, nextCursor}` envelope — **keep
  requesting until `nextCursor` is `null`**, since a page can come back empty or shorter than
  `limit` while further results remain.
- **Placement matchers in `data_scope`.** A clause can now claim a whole ownership dimension without
  enumerating its values. `"${{ any }}"` matches any value present in that dimension — deliberately
  *not* an item with no value there, so combine it with `null` to cover both.
  `"${{ under.self.userId }}"` and `"${{ under.self.scope.<namespace> }}"` match values whose immediate
  parent is your credential's own — one level, not a full ancestor walk — so a credential confined to an
  organization can work with the clients under it without naming each one at mint time. The existing
  `"${{ self.* }}"` forms are unchanged.
- **The `"*"` dimension wildcard in `data_scope`.** `{"*": ["${{ any }}", null]}` states a rule for
  every dimension the clause does not name explicitly; a named dimension always takes precedence over
  it. Use it to grant across all ownership dimensions without enumerating them.
- Any other `${{ … }}` spelling in a `data_scope` value is now **rejected when you author it**, rather
  than being stored as a literal that quietly matched nothing.

### Security

- **A user's `email` can no longer be changed while an invitation to them is outstanding.**
  `PUT /v1/users/{id}` and `POST /v1/users?upsert=true` now return `400` for that one
  case; every other mutable field is unaffected, and the address is editable again once the
  invitation is accepted or the user is removed. Revoke and re-invite to change where an outstanding
  invitation goes. Previously the change was accepted, leaving the invitation addressed to one
  mailbox and the user record pointing at another.

- **An access profile can no longer name a role that does not exist.** A `roleId` naming no role in the
  app context now returns `400` identifying it — on `POST /v1/users/invite`, on
  `POST /v1/app-contexts/{contextId}/profiles` (including `?upsert=true`) and
  `PUT /v1/app-contexts/{contextId}/profiles/{principalId}`, and when creating a scoped key bound to
  such a profile. Previously the reference was accepted and stored: the profile looked valid, the
  principal it named could never obtain a scoped token, and nothing indicated why.

- **Changing or clearing an access profile's `identityOverrides` now requires that you hold the value you
  are replacing.** A scoped credential could already only *set* an identity value it holds itself. It could
  still point another principal's established identity at its own value, or wipe it by sending an empty
  map — neither of which is setting a value you do not hold, so neither was refused. Both now return `403`
  on `PUT /v1/app-contexts/{contextId}/profiles/{principalId}` and on
  `POST /v1/app-contexts/{contextId}/profiles?upsert=true`.

  **`DELETE /v1/app-contexts/{contextId}/profiles/{principalId}` is covered by the same rule**, because
  deleting a profile removes its identity just as completely: a scoped credential can no longer delete a
  profile whose `identityOverrides` hold a value it does not. Deleting a profile that carries no
  `identityOverrides` — the ordinary case — is unchanged.

  Also unchanged: giving your own identity to a profile that has none, editing or deleting a profile whose
  identity is already yours, omitting `identityOverrides` to leave it alone, and root API keys, which are
  exempt throughout. If you manage profiles with a scoped credential that carries no identity of its own,
  note that it can no longer clear `identityOverrides`, nor delete an identity-bearing profile — use a
  root key for those.
- **Minting a scoped API key now requires that you hold the identity it would carry.** `POST
  /v1/admin/keys/scoped` already refused to mint a key whose *scopes* exceed your own; it did not check the
  bound access profile's `identityOverrides`, even though a scoped key acts as that identity when it reads
  and writes. A scoped caller binding a key to a profile whose identity it does not hold now gets `403`.
  Root API keys are unaffected, and a profile with no `identityOverrides` mints exactly as before.
- Resending an invitation requires `users:r` and `users:u` in addition to `users:c`. Without them, a
  re-invite collision returns `409`.
- An API key administers scoped API keys (`ssk_*`) only in its own environment; a key in your other
  environment returns `404`.
- **A clause that says nothing about an ownership dimension no longer authorizes writing data into it.**
  Previously a clause with no `data_scope` — or one omitting a dimension — could place a record in that
  dimension. Reading is unaffected: a dimension a clause does not mention still does not narrow reads.
  If a credential needs to place data in a dimension, name that dimension in `data_scope` (or use `"*"`).
- **Pagination cursors are now encrypted, so a cursor no longer reveals anything about the row it
  resumes from.** A cursor has always been documented as opaque — *echo it back, do not parse it* — but
  it was only encoded, so a determined caller could decode one. What that exposed matters when your
  credential's data scope narrows what you may read: a listing filters results **after** reading a page,
  and the cursor necessarily points at the last row *read* rather than the last row *returned*. A caller
  could therefore decode their own cursor and learn the id — and, for a lookup ordered by a text field,
  the field value — of a row their scope had just denied them, one row per request and with no entry in
  the read-access log. Cursors are now sealed: the id and field values a cursor carries can no longer be
  read out of one.

  **Nothing changes for a conforming client.** If you echo the cursor back unmodified, as the contract has
  always required, you will notice only that the token looks different. If you have code that inspects,
  parses, stores long-term, or **compares** cursors, it will stop working — comparison in particular, since
  the same resume position now encrypts to a different token every time.

### Changed

- **`lookupFields[].fieldName` is now optional.** This is the one change in this release that can stop a
  typed SDK client compiling, so it is worth stating precisely: a lookup now declares either `fieldName`
  or `fieldNames`, so `fieldName` became optional on `LookupDef` in the TypeScript, Python and Java SDKs.
  Code that *reads* `fieldName` off a schema response must handle it being absent — a lookup over several
  fields reports `fieldNames` instead. This affects reading a schema as much as writing one: if you GET a
  schema, change something, and PUT it back, a lookup over several fields must carry its `fieldNames`
  through, or the update is rejected.

  Nothing else about lookups changed shape. `values` is a new optional parameter everywhere it appears
  (`GET`/`POST /v1/records/lookup` and `BatchLookupInput`); no existing parameter changed type, and no
  request you send today means anything different.

- **Scope values now have a character grammar.** The `<value>` half of a `<namespace>:<value>` pair must
  be 1–128 characters, start with a letter or digit, and otherwise contain only letters, digits, `_` and
  `-`. This applies everywhere a scope value is written or filtered on: the `scopes` array on records,
  documents, folders, schemas and entities; the `?scope=` list filter; an access profile's
  `identityOverrides`; and the `identity` block of `POST /v1/auth/token`. Entity IDs and identifiers such
  as `eng-team` or `org_x` are unaffected — but a value containing a colon, a space, or other punctuation
  is now rejected with `400` where it was previously accepted and stored. A stored value containing a
  colon could cause scoped list and search requests in your account to fail, so such values are now
  refused at the point they are written rather than at the point they are read.
- The scoped-API-key plan limit is counted per environment rather than per account.
- **A credential's `identity` no longer limits which value it may write into a scope dimension — the
  clause does.** `identity` supplies the default value stamped when you do not state one, and is what
  `${{ self.* }}` resolves against. So a credential whose identity is `org:A` and whose clause admits
  `["A","B"]` may now create *and* update records into `B`; previously it could do neither on update,
  and the two verbs disagreed. To confine a credential to its own value, write that in the clause as
  `"${{ self.scope.<namespace> }}"`.
- **The owning user (`userId`) remains server-attributed**, on create and update alike: a request whose
  `userId` conflicts with the credential's identity is rejected rather than silently overridden, even
  when the clause would otherwise admit the value.
- **An update rejected because its resulting ownership falls outside your `data_scope` now returns `400`
  consistently.** Some of these previously returned `403` and some `400` — which one you got depended on
  whether your clause happened to name the dimension you were writing to, not on anything you did
  differently. The verdict is unchanged; only the status is now predictable, and it matches what the same
  denial has always returned on create. `403` on an update is now reserved for reassigning the
  server-attributed `userId`, and a record your credential may not write remains a uniform `404`.
- **Creating an identity entity no longer matches your clause against the entity's own namespace
  dimension.** An entity in namespace `team` is always in scope `team:<its own id>`, and that id is
  generated by the create you are making — so no clause written beforehand could name it, and until now
  a credential confined to `{"scope:team": ["${{ under.self.userId }}"]}` could read the entities it
  owned but never create one. That single dimension is now exempt from the match on
  `POST /v1/entities/{namespace}`, and only there: every other dimension in the same clause still
  decides, and reads, updates and deletes match it normally. One consequence worth knowing when you
  write roles — a clause whose `data_scope` names **only** that dimension does not restrict what you may
  create, and the entity it creates may fall outside it once the id exists.
- **`startFrom` is now an opaque token on every lookup and list endpoint, and must be echoed back
  unchanged.** Pass back the `nextCursor` you were given, verbatim. If you built a `startFrom` yourself
  from a record id — which worked on some endpoints — that now returns `400`.

  The one exception is the `/versions` history reads (on records, documents, folders, schemas, entities,
  roles and access profiles), which still resume from a row id. Handing one of those a cursor from a
  list call returns a `400` naming `startFrom`.

- **`RecordLookupResponse` is renamed to `RecordLookupPage`.** SDK type name only — the JSON is
  byte-identical. **Typed SDK clients that name the class must rename it**; clients reading the JSON
  need no change.
- **A cursor is valid only for the exact query that returned it.** Keep every other parameter identical
  while paging. Changing the lookup mode mid-pagination (dropping `sortTo`, switching from `from`/`to`
  to `value`) returns a `400` explaining what to do.
- **A transient storage fault on a listing now returns an error instead of an empty page.** These
  previously returned `200` with `{data: [], nextCursor: null}` — indistinguishable from "you have no
  matching records". Retry on `5xx` rather than treating an empty page as authoritative.
- **In-flight pagination does not survive this release.** A cursor issued before it is refused with
  `400`; start the query again. Cursors are short-lived by design, so this affects only a page turn
  that straddles the deploy. **`Vectros-Version` does not shield you from this** — it pins the *shape*
  of a response, not the validity of a token already issued, and a cursor is a value rather than a
  shape.

### Fixed

- **A listing or lookup could report itself complete while results remained.** These endpoints decided
  "there are no more pages" by checking whether a page came back full. A page comes back short for a
  reason unrelated to being the last one: the storage layer returns at most 1 MB per read, so a page of
  large records ends early with more behind it. You would receive a handful of records and
  `nextCursor: null`, and correctly conclude that was all of them. A range lookup reads as exhaustive
  ("everything between these two values"), so the wrong answer looked right.

  Affected: `GET /v1/records` and `GET /v1/documents` on every filter (`?type=`, `?folderId=`,
  `?recent=true` and unfiltered); `GET`/`POST /v1/records/lookup` and `/v1/documents/lookup` narrowed by
  `from`/`to` or `prefix`, and the same modes on `/v1/users` and `/v1/entities/{namespace}`; and
  `GET /v1/records`, `/v1/documents`, `/v1/folders`, `/v1/schemas` and `/v1/entities/{namespace}`
  narrowed by `userId` or `scope`. On the range and prefix modes your `limit` never reached the query at
  all.

  **How exposed you were depended on a setting you chose.** A schema with `storageProfile: LOW_LATENCY`
  keeps record payloads inline, so on those types as few as **three large records** could fill a read and
  truncate the listing. `STANDARD` and `LARGE_PAYLOAD` schemas were far less affected. This applied to
  every credential, including root API keys.

  All of these now page on a cursor, which is the only sound completeness signal. **The affected case is
  a page that came back SHORTER than your `limit`**, not one that came back full.

- **A schema could be deleted while records of its type still existed.** Deleting a schema checks first
  that no records reference it, but a transient storage fault during that check was indistinguishable
  from "no records found", so the delete proceeded and left those records pointing at a schema that no
  longer existed. The check now fails loudly on a fault, and the delete is refused.
- **Lookup ordering on text fields is now the field's natural order**, and an item that leaves the sort
  field empty sorts consistently ahead of the items that fill it rather than being placed among them by
  creation time. Ordering by a number, date, boolean or timestamp field is unchanged. Applies to every
  lookup that accepts `order` — records, documents, folders, users and entities.

  **Both apply to items written from this release onward.** Items written earlier keep their previous
  ordering until they are next updated, so a lookup spanning both can show the two groupings side by
  side; any update to an item brings it onto the new ordering.

- Ownership-constrained scoped credentials no longer receive a spurious `404` on app contexts, roles and
  access profiles, for any verb their scope grants. Also affects `POST /v1/admin/keys/scoped` and
  `POST /v1/users/invite[/resend]`.
## 0.37.0 — 2026-07-27

### Added

- **`contextId` on `POST /v1/auth/token`.** A root API key can now mint a scoped token targeting any
  app context that already exists in your account, not just the one your own key operates in — omit
  it to keep minting into your key's own context as before. Must reference an existing context; an
  unrecognized value returns a uniform `404 not found`.
- **`basedOn` on record schemas.** A schema can now declare `basedOn: <schemaId>` to mark itself as a
  customization of another schema sharing its `typeName` — a team or user tweaking a context-wide type
  (an added field, a stricter validation) while remaining the same conceptual type for references,
  listings, and blueprints. The first schema created under a name has no `basedOn` and becomes that
  name's shared base; every other schema of that name must declare it. `basedOn` is immutable once set
  and must point directly at the base (one hop). See `POST /v1/schemas`.
- **`specificityRank` on scope namespaces.** `POST /v1/namespaces` now requires an explicit, account-unique
  `specificityRank` for every custom namespace — an integer position in your account's specificity order,
  used to break a tie when a caller holds two scope dimensions at once during `basedOn` schema resolution
  (see below). `PUT /v1/namespaces/{namespace}` accepts it too; omit to leave it unchanged.
- **`userId`/`scope` selectors on document lookup.** `GET`/`POST /v1/documents/lookup` accept `userId`
  and `scope` (root API key only) to resolve the looked-up type's schema as a specific owner would,
  mirroring the selector already on `GET /v1/schemas?recordType=`.
- **`externalId` on search hits.** `POST /v1/search` results and `POST /v1/rag` citations now carry the
  matched item's `externalId` alongside `documentId`, so you can correlate a hit back to your own record
  identity without a follow-up lookup. Null when the item was ingested without one, or when it was
  indexed before this field existed and hasn't been updated or reindexed since.
- **`hasMore` on search responses.** `POST /v1/search` now returns an explicit `hasMore` boolean — true
  when more matching results are available past this page, false once you've reached the last page or
  the `offset` maximum (200) — so you no longer need to infer it yourself from `totalResults`, which
  reports an approximate matching-pool size and is not a reliable paging signal on its own.

### Changed

- **`POST /v1/search` `limit` now honors its full advertised range (1–100).** Previously a `limit` above
  50 was silently capped to 50 with no error and no signal that results were incomplete, even though the
  documented maximum was always 100. Requests with `limit` in 51–100 now return up to that many results,
  matching the OpenAPI contract. Pair with the new `hasMore` field to detect when more results remain.
- **Search-index status docs clarify semantic-search timing.** The `indexStatus` description on record
  and document responses now states plainly that `INDEXED` means keyword (TEXT) matching is immediate,
  while the semantic (vector) index is eventually consistent for queries — typically queryable a second
  or two later, longer under heavy indexing load. A HYBRID or RAG query that shares keywords with a
  freshly indexed item still surfaces it immediately via the keyword leg; a query that can only match it
  semantically (a `mode: SEMANTIC` search, or a HYBRID/RAG query sharing no words with the item) may not
  return it for those first few seconds. No behavior changed; this documents an existing, previously
  unstated timing characteristic.
- **`textScore` in TEXT mode is no longer always `0`.** `POST /v1/search`'s TEXT-mode results now carry a
  real, descending-by-relevance `textScore` for each hit (previously hardcoded to `0` for every TEXT-mode
  result, which also made the keyword-relevance guidance surfaced by AI-agent clients spuriously trigger
  on every non-empty TEXT-mode search). It reflects the result's rank, not a normalized BM25 magnitude —
  treat it as meaningful for ordering within one response, not as a value comparable across requests or
  against HYBRID/SEMANTIC scores. `/v1/rag`'s `RagSearchResult.textScore` carries the same clarification.
- **`recordType` resolution now shadows by ownership instead of failing loud on cross-owner ambiguity.**
  `GET /v1/schemas?recordType=`, `POST /v1/records`/`POST /v1/documents` by `typeName` alone, and the
  record/document lookup-by-field endpoints resolve a same-named type to the caller's own `basedOn`
  variant when one exists, otherwise the shared base — replacing the `400 AMBIGUOUS_RECORD_TYPE` these
  endpoints returned when more than one owner had defined the type name (0.36.0). That errorCode is now
  reserved for a should-be-impossible data-integrity state, not a normal multi-owner situation.
- **Creating a second schema with an already-used `typeName` now requires `basedOn`.** Previously any
  owner could freely define a schema under a name another owner already used, with no relationship
  between the two. A create that omits `basedOn` when the name already exists is now rejected with a
  `400`; the first schema under a name must additionally be created by a root/unscoped credential with
  no `userId`/`scopes` (it becomes that name's shared base). This is a breaking change for a caller that
  relied on same-name schemas coexisting with no declared relationship.

## 0.36.0 — 2026-07-23

### Added

- **`requestId` and `errorCode` on activity-log entries.** Each `GET /v1/admin/logs` entry carries
  the call's `requestId` — the same id returned in an error response body, and the one to quote to
  support. A failure rejected with a typed code also carries that `errorCode`: `RATE_LIMITED`,
  `SUBSCRIPTION_LIMIT_EXCEEDED`, `INSUFFICIENT_BALANCE`, `RESOURCE_IN_USE`, `VERSION_CONFLICT`, or
  `SESSION_REFRESH_REQUIRED`. Request and response bodies are never logged, so a failure carrying
  only a message reports no `errorCode`.
- **Inference error responses carry an `errorCode`.** A `402` from `/v1/chat`, `/v1/rag`, or
  `/v1/documents/{id}/ask` now includes the typed code in its body, so you can branch on the cause —
  `INSUFFICIENT_BALANCE` (top up your pre-paid balance) versus `SUBSCRIPTION_LIMIT_EXCEEDED` (upgrade
  your plan or raise your cap) — without matching the message text.
- **`indexFailure` on record and document responses.** Present when `indexStatus` is `FAILED`, and
  absent otherwise. It carries a stable `code` you can branch on and a human-readable `message`
  suitable for showing to an end user:
  - `SOURCE_UNAVAILABLE` — the underlying item could not be loaded and may have been deleted.
  - `TEXT_INDEX_FAILED` — keyword indexing failed; the content may still be findable by semantic search.
  - `EMBEDDING_FAILED` — semantic indexing failed; the content may still be findable by keyword search.
  - `INDEXING_FAILED` — no index leg is serving this content, so it is not findable by search at all.
  - `VECTOR_LIMIT_EXCEEDED` — your vector storage limit was reached, so semantic indexing was skipped;
    keyword search is still serving this content.
  - `INTERNAL` — an error on our side; retry, and contact support if it persists.

  Branch on `code`, not on `message` — the wording may change between releases. Several codes mean
  the content is still partly findable, which is the practical difference between "retry this" and
  "this one needs attention".
- **`indexFailure` on the `record.failed` and `document.failed` webhook payloads**, in the envelope's
  `data` block, with the same `code` and `message`.
- **`AMBIGUOUS_RECORD_TYPE` errorCode.** `GET /v1/schemas?recordType=`, `POST /v1/records` by
  `typeName` alone, and the record/document lookup-by-field endpoints now return a `400` with this
  errorCode when more than one schema in the context shares that type name under different owners,
  instead of guessing one. Resolve by `id` (schemas) or `schemaId` (records) instead.

### Changed

- **Every number must fall within the signed 64-bit range**, `-9223372036854775808` to
  `9223372036854775807` — whole or fractional, in any field, whether or not your schema declares it,
  and in a search request body as well as a record or document payload. A value outside the range is
  refused with a `400` naming the field, as are values carrying more than 38 significant digits,
  magnitudes below roughly `1e-130`, and values that are not finite. Send large whole numbers as
  strings: a field declared as `string` stores them exactly and supports exact-match lookup.

  **A record written before this release may hold an out-of-range value.** A whole number beyond the
  64-bit range was lost at write — the create returned `201` and echoed the value back, but the record
  reads back as not found and is absent from lists. A value written in exponent or decimal notation
  was stored, but a field your schema declares as `number` returns it as a string rather than a
  number. Updating either is refused with a `400` until the field is brought into range or changed to
  a string.
- **Ownership placement is authorized against a single scope clause.** A create or update that sets
  `scopes` succeeds only if **one** clause of the credential permits the operation, matches the
  resulting ownership, and — on update — is confined to every namespace the change touches, **including
  one it removes**. Removing a label moves the item out of that compartment and widens who can read it,
  so it is authorized exactly like adding one: a credential with no grant over a namespace can neither
  add nor clear a label in it.

  `scopes` on a `PUT` is a complete declaration, so a client that round-trips an item re-sends labels
  it did not change. If one of those is in a namespace the granting clause does not cover, the `PUT`
  is rejected with a `403` naming the namespace. Send only the labels the credential is scoped for,
  or use `PATCH`.
- **`userId` is authorized as a placement, on create as well as update.** A credential may attribute
  an item to a user other than its own only if the clause authorizing the write constrains the user
  dimension. If a credential now returns a `403`, list the user ids it may write in that clause's
  `userId` values — do not remove its `dataScope`, which would grant it strictly more.
- **A credential with a `dataScope` but no `identity` must state ownership on create.** Omitting
  `scopes` is refused rather than producing an account-wide item. To create account-wide items
  deliberately, include `null` in the `dataScope` value list.
- **Access-profile identity overrides are authorized like scopes.** Setting `identityOverrides` on a
  profile is subject to the same authority check as `scopes` and `roleId`, even when the request body
  contains *only* `identityOverrides`. A scoped credential may set an override only to an identity
  value it holds itself (a `403` otherwise); a root key's override values must reference entities that
  exist in your account (a `400` naming the value otherwise).
- **An idempotent create returns the existing item only if your credential can read it.** A `POST` that
  collides with an existing item by `externalId` (or, for folders, by slug) returns that item — `200`
  with `created: false` — only when a single clause of your credential permits reading it *and* matches
  its ownership. A credential that can create but not read the colliding item — or that holds a read
  grant only on a *different* resource or namespace — now receives the uniform `already in use` `400`
  instead of the item's id, ownership, and contents, so a create grant can no longer reveal items in
  compartments you cannot read. This applies uniformly to records, documents, folders, identity
  entities, schemas, users, roles, and access profiles, and to the file upload-init re-issue — for a
  role or access profile the read grant is `profiles:r` (a create-only credential colliding on an
  existing `roleId`/`principalId` receives `already exists` rather than the existing scopes or grant).
  Full-authority API keys are unchanged.
- **Referencing a folder requires read access to it.** Creating a record or document with a `folderId`,
  or a sub-folder with a `parentFolderId`, now requires a single clause of your credential to grant
  `folders:r` and match that folder's ownership. Previously any clause whose `dataScope` matched the
  folder was accepted — even a grant on a different resource, or one with an empty `dataScope` — so a
  credential that could not read a folder could still file into it and probe whether it exists. A folder
  your credential cannot read now returns the uniform `Folder not found` `400`, the same response as a
  folder that does not exist. Filing into your context's default folder (omitting `folderId`) is
  unchanged, and full-authority API keys are unaffected.
- **A qualifier is rejected on an op that doesn't correlate it.** `allowed_actions` entries in the
  `resource:ops:qualifier` form correlate their qualifier on two independent axes: `records`/`entities`
  correlate it for **every** op (c/r/u/d/s — qualified by record type / namespace); `documents`/`users`
  correlate it **only for the `s` (sensitive-reveal) op** (qualified by record type). An entry whose
  qualifier is inert on **every** op it names — e.g. `folders:c:bar`, `schemas:r:baz`, or `documents:r:foo`
  (read has no qualifier axis on documents) — was previously accepted at authoring but silently ignored
  at read/write time, granting the unqualified action on **every** item, not the named subset.
  `POST /v1/auth/token`, and access-profile/role authoring, now reject such an entry with a `400` naming
  the resource and op(s). **Remediation depends on which op(s) are inert — read carefully, this is not
  always "drop the qualifier":**
  - `folders:c:bar`, `schemas:r:baz` (resource has no qualifier axis at all): drop the qualifier — no
    capability is lost, it was never narrowing anything.
  - `documents:r:foo`, `users:c:foo` (op has no qualifier axis on this resource, but `s` does): drop the
    qualifier from this entry, but if you intended to narrow **sensitive-field reveal** specifically,
    author a **separate** `documents:s:foo` / `users:s:foo` entry instead — do not simply widen a
    sensitive-reveal grant to match the (already-unrestricted) read grant.
  - `documents:rs:foo`, `users:crs:foo` (a **mixed** entry — `s` correlates the qualifier here, the other
    op(s) don't): split it into two entries, e.g. `documents:r` + `documents:s:foo` — the combined form
    can't express "unrestricted read, reveal-restricted-to-foo" with one qualifier segment, and is
    rejected rather than silently doing the wrong thing for one of the two ops.
  This validation runs at authoring only: a token or stored access-profile/role scope minted before this
  release that already carries a rejected-shape entry keeps working exactly as before. If you have one,
  re-author it per the remediation above — check whether any of its ops were `s` before deciding a
  qualifier is safe to drop.

### Fixed

- **The activity-log `resource` filter accepts the identity surfaces.** `GET /v1/admin/logs` filters
  on `entities` and `namespaces`, which have been accepted since 0.35.0 but were not listed among the
  documented values, so there was no way to discover that the identity surface is filterable at all.
  `clients` and `orgs` remain accepted, and match log rows written before those surfaces were folded
  into `entities`.
- **`requestId` identifies a single call.** One `requestId` could be repeated across many different
  calls, in both error responses and activity-log entries, so quoting it to support did not identify
  the request. Each call now reports its own.
- **Activity-log entries report the app context the call actually ran in.** An entry could carry the
  context of an earlier call, so filtering by `contextId` could return calls that never ran in that
  context.
- **Ownership parents are validated on update.** Updating an item to reference a non-existent entity
  in an entity-backed namespace fails with a `400` naming the missing parent, matching the
  create-time validation.
- **The bootstrap CLI's own credential can complete an idempotent create-or-get on retry.** Per the
  read-requirement described above, re-running `vectros bootstrap` (or `--rotate`) against an
  already-provisioned tenant collided on the existing user/entity and received `already in use`
  instead of the existing record, so a bootstrap re-run that used to converge could fail. The
  bootstrap token now also carries the paired read scope on the surfaces it creates.

## 0.35.0 — 2026-07-19

**One ownership vocabulary.** Organizations and clients are no longer special-cased types with their
own endpoints, fields, and scopes — they are ordinary namespaces alongside the ones you define. The
same endpoints that serve your `team` or `project` entities now serve `org` and `client`, and
everything that used to speak `orgId`/`clientId` speaks `scope:<namespace>`.

**This is a breaking release**, and deliberately a clean one: there is no compatibility shim and no
deprecation period, because the alternative was carrying two spellings for two privileged namespaces
forever. Every change is mechanical, and every removed form fails loudly — as a compile error or a
`400` that names its replacement. Nothing about *how* ownership behaves changes: the values, the
matching rules, and the `null` tenant-level opt-in are all exactly as before.

### Added

- **`/v1/entities/{namespace}` — one CRUD surface for every identity entity.** Create, read, update,
  delete, list, look up, and read version history for an entity in any entity-backed namespace —
  including the built-in `org` and `client`. Supports `externalId` idempotent create, `?upsert=true`,
  sensitive-field masking, `?scope=<namespace>:<value>` parent listing, and the full schema field
  lookup surface (`?type=&field=&value=`, plus `POST /v1/entities/{namespace}/lookup` for sensitive
  values) — the same capabilities the per-type endpoints had, now uniform across namespaces.
- **`/v1/namespaces` — declare your own entity-backed namespaces.** Register a namespace with
  `entityBacked: true` and it gains a full identity-entity surface, ownership references that are
  existence-checked on create, and parent listing. Reads are open to any credential; writes require
  a root API key. `org` and `client` are built in and read-only.
- **Ownership parents are validated.** Creating an item scoped to an entity-backed namespace now
  fails with a clear 400 if the referenced entity does not exist in your account, instead of storing
  a dangling reference.

### Changed

Ownership is declared and filtered one way, everywhere:

- **Declare it with `scopes`** — an array of `namespace:value` entries — on records, documents,
  folders, schemas, and uploads. On update the array is the complete declaration: what you omit is
  removed, and `[]` clears ownership. `userId` is unchanged and remains its own field: the user is a
  separate dimension, not a namespace.
- **Read it back from `scopes`** on those same responses and on the webhook envelope.
- **Filter by it with `?scope=<namespace>:<value>`** on the list endpoints, and with
  `"scope": "<namespace>:<value>"` in search and in the RAG `search` block. One entry per query — pair
  it with `?userId=` when you need both dimensions.
- **Bind a credential to it with `scope:<namespace>`** in a token's `dataScope`/`identity` and in an
  access profile's `identityOverrides`; reference it with `${{ self.scope.<namespace> }}`. Grant
  access to it with `entities:<verb>:<namespace>` — mirroring the existing `records:<verb>:<type>`
  grammar. `GET /v1/ping` reports the resulting binding as `dataScope.scopes`.
- **`subjectType` is a plain string**, not a fixed enum, on `ErasureRequest`, `ExportRequest`, and
  read-access-log rows: it takes `user` or any ownership namespace, including one you define. A
  subject kind is your data, not our list.
- **`scopes` is a reserved payload field name**, alongside `userId`/`externalId` — it
  is a first-class top-level field, so it cannot double as a payload key or a schema lookup field.
- **Schemas bind to `entity`, not to `org` or `client`.** A schema's `allowedSurfaces` takes `record`,
  `document`, `user`, or `entity`. Identity entities in *every* namespace — the built-in `org` and
  `client` included, alongside any you register — bind under the single `entity` surface, and the
  schema's `typeName` tells them apart. `allowedSurfaces: ["org"]` is rejected with a 400 naming the
  replacement, and `GET /v1/schemas?surface=` filters on the same set. There cannot be a per-namespace
  bind surface: the list is fixed, so a namespace you register tomorrow would have nowhere to bind.
- **`GET /v1/usage` reports one `identity.entities` line** in place of `identity.orgs` and
  `identity.clients` — the same fold, on the billing side. This also closes a gap: identity-entity
  writes were being metered but were missing from the usage breakdown entirely.
- **A schema reference can point at any entity namespace.** A `reference` field's `targetSurface` takes
  `record`, `document`, `user`, or the name of an identity namespace — `org`, `client`, or one you
  registered, such as `team`. It is no longer a fixed enum, since the namespaces are yours to define.
  Referencing an `org` or a `client` is unchanged; referencing a namespace of your own is new. The
  namespace must already be registered and entity-backed, or the schema is rejected — a reference that
  could never resolve now fails when you define it, rather than on every write against it.
- **More reserved namespace names.** `record`, `document`, and `entity` join `user`, `self`, `tenant`,
  `context`, and `scope` as names a namespace may not take, and `versions` and `lookup` are reserved
  too. The first three would collide with a reference `targetSurface`; the last two with the
  `/v1/entities/{namespace}/...` sub-paths.

### Removed

The dedicated `org`/`client` vocabulary, in favour of the generic forms above. If you are upgrading
from 0.34.0, every one of these is a compile error or a `400` that names its replacement — there is no
silent behaviour change to hunt for:

- `/v1/orgs` and `/v1/clients` (use `/v1/entities/org` and `/v1/entities/client`), and the
  `orgs:<verb>` / `clients:<verb>` action scopes.
- The `org` and `client` schema surfaces (use `entity`), and the `identity.orgs` / `identity.clients`
  sections of `GET /v1/usage` (use `identity.entities`).
- The `orgId` / `clientId` request-body fields, response fields, list filters, lookup `field=` values,
  and search/RAG filters.
- The `orgId` / `clientId` keys in `dataScope`, `identity`, and `identityOverrides`; the
  `${{ self.orgId }}` / `${{ self.clientId }}` placeholders; and `dataScope.orgId` on `/v1/ping`.

One narrowing is deliberately not carried over: `?scope=` takes a single entry, so filtering on two
scope dimensions at once is no longer expressible (`?userId=` plus one `?scope=` still is). Repeated
`?scope=` is additive and may land later.

### Fixed

- **A declared `number` is returned as a number, not a string.** Reading a record,
  document, user, or identity entity returned `"priority": "30"` where it
  should have returned `"priority": 30` — on every read: fetch by id, list, and
  lookup. Nothing needs to be re-saved; your existing data returns the declared
  type immediately.

  If you worked around this by parsing the string form, you can drop the
  workaround. **In Java it must go:** code that cast the value to `String` will
  now fail against a number.

- **Updating an item no longer fails on a number field you did not send.**
  Renaming a record, archiving it, moving it to another folder, or changing its
  owner could be rejected with `400 Field 'priority' must be a Number` even though
  the request never mentioned `priority`. Sending a string for a declared `number`
  is still rejected, as before.

- **Version history returns declared numbers as numbers.** Entries from
  `GET /v1/records/{id}/versions` (and the document, user, and entity
  equivalents) could carry the string form of a number field inside
  `previousContent` and `changedFields` for history written before this
  release. All entries — old and new — now serve the declared types. Values are
  typed against the item's current schema; an entry written under a schema
  whose field types have since changed reports its values in the current types.

- **Re-pointing a document to a compatible schema works.** A `PATCH` that
  changes a document's `schemaId` to a schema declaring the same field types
  (the common v1-to-v2 migration) was rejected with a `400` on an untouched
  number field. It now succeeds. Re-pointing to a schema that declares
  DIFFERENT types for stored fields is validated honestly: include the
  re-typed values in the same request, or the response names the fields to fix.

- **Identical re-applied writes are no-ops again.** Re-sending an unchanged
  item with `?upsert=true`, or re-applying the same update, could bump the
  version, write an audit entry, and re-index — every time — when the payload
  contained a declared `number`. An unchanged write is now recognized as
  unchanged: no version bump, no audit entry, no re-index, no write fee.

- **Search indexing status no longer sticks in `PENDING_INDEX`.** An update to
  a record with a numeric payload field could leave `indexStatus` reporting
  `PENDING_INDEX` after indexing had in fact completed.

- **A storage outage can no longer truncate a large item.** For items whose
  payload is stored externally (large payloads), an internal storage failure
  during an update, upsert, or re-upload could silently persist only the small
  indexed subset of the payload — permanently losing the rest. Such writes now
  fail whole with a `500` and leave the stored item untouched; retry when the
  incident clears. Reads are unaffected: they keep returning the indexed subset
  with `payloadExternalized: true` until the payload is reachable again.

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
