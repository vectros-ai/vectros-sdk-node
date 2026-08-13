# Changelog — Vectros API & SDKs

The Vectros API and its official SDKs — **TypeScript** (`@vectros-ai/sdk`),
**Python** (`vectros`), and **Java** (`ai.vectros:vectros-sdk`) — share one version
line and this changelog. The SDKs are generated from the OpenAPI surface, so a
release moves the API and all three SDKs together. This file is the canonical
source (maintained alongside `sdk-version.txt`); the publish pipeline copies it
into each SDK package + mirror **and the `vectros-api-spec` repo**.

This project adheres to [Semantic Versioning](https://semver.org).

## 0.39.0 — 2026-08-07

_This section is still open — further 0.39.0 changes land here until release._

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
  `PUT /v1/users/{userId}` and `POST /v1/users?upsert=true` now return `400` for that one
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
