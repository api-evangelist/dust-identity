---
name: dice-api-integration
description: Integrate software against the DUST platform API (apid) — authentication with API keys and bearer tokens, org/team context headers, and the core flows: creating Threads, binding and identifying physical Identifiers, uploading files (tus), sharing with other Teams, and sending Shipments. Use this skill whenever you are writing code that calls the DUST API directly or through the @dustid/apid-client TypeScript client, wiring a customer backend to the DUST/DICE platform, or debugging DUST API requests (401s, context-header errors, upload finalization, transfer lifecycle).
version: 2026.7.4
---

# DUST API integration (apid)

Generated from DUST API OpenAPI spec 2026.7.4 (147 paths), docs version 2026.7.4.

This skill is self-contained: it covers auth, request conventions, and the five core integration flows with both TypeScript-client and raw-curl variants. For anything beyond it, consult:

- Interactive API reference (Scalar UI): https://apid.dustid.io/api/docs
- Raw OpenAPI 3 spec (always current for that server): https://apid.dustid.io/api/openapi.json
- Documentation site: https://docs.dustid.io

Production base URL: `https://apid.dustid.io`. All examples below use `$APID_URL` / `baseUrl` so they work against any environment.

## Terminology

- **Thread** — the record for one physical asset/item; holds typed data fields, files, identifiers, and an event history.
- **Team** — the ownership and access boundary inside an organization. The wire protocol sometimes says `Grp`/`group` (legacy name) — always read it as "team".
- **Identifier** (wire name: *tag*) — a physical marker bound to a Thread: a DUST tag, QR, barcode, Data Matrix, or NFC. Endpoints live under `/api/v1/tags/*`.
- **Folder / Category** (wire name: *bundle*) — containers for Threads; `bundleId` in APIs.
- **Shipment** (wire namespace: `/api/v1/transfers`, legacy naming) — transfers ownership of Threads to a connected Team.
- The product name is always written **DICE** (all caps).

## 1. Authentication

Every `/api/v1/*` request needs a bearer JWT issued by AuthD (the DUST account service). Flow:

1. An organization admin creates a **Service Account** (a machine identity owned by exactly one organization) and issues it an **API key**, in the AuthD account portal at https://authd.dustid.io (organization page → Service accounts; requires the org's service-accounts entitlement — contact support@dustidentity.com to enable it). Personal/user-owned API keys are not supported.
2. Exchange the key for a short-lived **bearer token** at `GET /api/auth/token`, passing the key in the `x-api-key` header.
3. Send `Authorization: Bearer <token>` on every API call.

```bash
export APID_URL="https://apid.dustid.io"
export DUST_API_KEY="your-service-account-key"

export DUST_TOKEN="$(
  curl -fsS "$APID_URL/api/auth/token" \
    -H "x-api-key: $DUST_API_KEY" | jq -r '.token'
)"
```

```ts
const tokenResponse = await fetch(`${apidUrl}/api/auth/token`, {
  headers: { "x-api-key": process.env.DUST_API_KEY! },
});
if (!tokenResponse.ok) {
  throw new Error(`Token exchange failed: ${tokenResponse.status}`);
}
const { token, expiresIn } = await tokenResponse.json();
// { "token": "eyJhbGciOi...", "expiresIn": 900, "expiresAt": "2026-07-14T22:40:00.000Z" }
```

Tokens are short-lived JWTs — read the lifetime from the response (`expiresIn` seconds / `expiresAt` ISO 8601; currently ~15 minutes) instead of hardcoding it. There is no refresh endpoint. Generated clients should cache the token and re-exchange the key ~60 s before `expiresIn` lapses, AND treat one `401 UNAUTHORIZED` as refresh-and-retry-once (covers clock skew and mid-lifetime revocation). Jobs that outlive one token must mint per request from that cache, not once at startup. The token's signing keys are public at `GET /api/auth/jwks` if your backend needs to verify DUST-issued JWTs itself.

Service Accounts also support **OAuth2 client_credentials** for enterprise middleware: create an OAuth client on the Service Account, then `POST https://authd.dustid.io/api/auth/dust/service-accounts/token` with `grant_type=client_credentials` (client_secret_post or HTTP Basic) — the resulting token is identical in rights and shape to the key-exchange token.

**Attribution (optional, sometimes required):** to record which human initiated an action in the integrating system, send the `Dust-Ctx-Declared-Actor` header — JSON `{"id": "...", "system"?, "displayName"?, "role"?}`, ≤1 KB, URI-encode if non-ASCII. It is recorded verbatim on events and shown as *declared*; it never affects permissions. If the Service Account's attribution policy is `required`, mutating requests without it are rejected with `403 ATTRIBUTION_REQUIRED`.

**Security: keys and tokens are server-side only.** A Service Account credential is long-lived and carries all of the account's access; a bearer token minted from it carries the same. Never embed either in browser JavaScript, mobile binaries, or source control. If a web/mobile app needs DUST data, put a small backend in front that holds the credential and proxies the calls.

Verify auth works with the one common endpoint that needs no context headers:

```bash
curl -fsS "$APID_URL/api/v1/me" -H "Authorization: Bearer $DUST_TOKEN"
```

The `/api/v1/me` response includes an `organizations` array (each with `id`, `name`, `slug`, `roles`) — use it to discover the org UUID for the context headers below.

## 2. Context headers

Almost every endpoint acts inside an organization and a Team, selected by request headers:

| Header | Required | Value |
| --- | --- | --- |
| `Dust-Ctx-Org-Id` | Yes, on org-scoped endpoints | Organization UUID. |
| `Dust-Ctx-Team-Id` | No | Team UUID. Defaults to the organization's root team when omitted. Legacy spelling `Dust-Ctx-Grp-Id` is still accepted (Team-Id wins if both are sent). |
| `Dust-Ctx-Locale` | No | Locale for server-generated user-facing text (error `message` strings). Supported: `en` (default), `zh-CN`. |

Notes that matter in practice:

- Header values must be UUIDs; malformed values are rejected with `400 INVALID_REQUEST` before the endpoint runs.
- `X-`-prefixed aliases (`X-Dust-Ctx-Org-Id`, `X-Dust-Ctx-Team-Id`, and the legacy pair) are accepted.
- Context is an authorization boundary: results are what *that Team* can see. Sending a context you don't belong to does not escalate access — requests are checked against your real memberships. Missing required context fails with `ORG_ID_REQUIRED` / `TEAM_ID_REQUIRED`.
- Error **codes** are stable and never localized; branch on `code`, display `message`.

## 3. The TypeScript client (`@dustid/apid-client`)

The typed client is the same one the DICE web app uses; its types are generated from the OpenAPI spec. It is currently **distributed by DUST, not published to the public npm registry** — integration customers can request it (support@dustidentity.com). If you can't take the dependency, generate equivalent types in your own environment from the published OpenAPI spec and use plain `fetch`.

```ts
import { ApidClient } from "@dustid/apid-client";

const client = new ApidClient({
  baseUrl: "https://apid.dustid.io",
  bearerToken: token,      // Authorization: Bearer <token>
  organizationId: orgId,   // sent as Dust-Ctx-Org-Id
  teamId: teamId,          // sent as Dust-Ctx-Team-Id (optional; defaults to org root team)
  // fetcher?: custom fetch; defaultHeaders?: e.g. { "Dust-Ctx-Locale": "zh-CN" }; logger?
});

// Clients are immutable; re-scope with copies:
const asOtherTeam = client.withContext({ teamId: otherTeamId });
const asFreshToken = client.withToken(newBearerToken);
```

Resources: `client.me`, `client.threads`, `client.bundles`, `client.files`, `client.tags` (Identifier operations), `client.teams`, `client.sharing`, `client.transfers` (Shipments), `client.templates`, `client.events`, `client.relations`, `client.threadLinks`, `client.assemblies`, `client.slices`, `client.imports`, `client.fabric`, `client.users`, `client.certificates`, `client.certificateForms`.

**Error convention — important:** client methods return the parsed JSON response on success and **throw `ApiError`** on any non-2xx status. They do *not* return a `{ result, error }` tuple (that pattern belongs to DICE's internal RPC layer, not this client).

```ts
import { ApiError } from "@dustid/apid-client";

try {
  const record = await client.threads.get(threadId);
} catch (error) {
  if (error instanceof ApiError) {
    // error.code    stable code, e.g. "NOT_FOUND", "UNAUTHORIZED", "FORBIDDEN"
    // error.status  HTTP status number
    // error.message human-readable, localized per Dust-Ctx-Locale
    // error.detail  optional extra context (validation issues, etc.)
    if (error.code === "UNAUTHORIZED") {
      // token expired — re-exchange the API key and retry once
    }
  } else {
    throw error; // network failure or non-JSON response
  }
}
```

Edge behaviors: `204`/`205` responses resolve to `undefined`; a non-JSON response body throws a plain `Error` (not `ApiError`).

## 4. Error response shape (raw HTTP)

Every failed request returns one consistent JSON body:

```json
{
  "code": "UNAUTHORIZED",
  "message": "You are not authorized to perform this action",
  "status": 401,
  "detail": { }
}
```

| Field | Meaning |
| --- | --- |
| `code` | Stable machine-readable code — branch on this. Common early ones: `INVALID_REQUEST` (400), `UNAUTHORIZED` (401), `FORBIDDEN` (403), `NOT_FOUND` / `NO_DATA_FOUND` (404), `ORG_ID_REQUIRED` / `TEAM_ID_REQUIRED` (400), `THREAD_DATA_CONFLICT` (409, optimistic-concurrency conflict). |
| `message` | Human-readable, localized per `Dust-Ctx-Locale`. |
| `status` | Mirrors the HTTP status. |
| `detail` | Optional object with extra context (e.g. validation specifics). |

Every response carries an `x-request-id` header — log it and include it in support requests.

Pagination on list endpoints is cursor-based: send `pageSize` + `cursor` query params; responses contain the items array plus opaque `next`/`prev` cursors (missing `next` = last page).

## 5. Core flows

All curl examples assume:

```bash
AUTH=(-H "Authorization: Bearer $DUST_TOKEN" \
      -H "Dust-Ctx-Org-Id: $DUST_ORG_ID" \
      -H "Dust-Ctx-Team-Id: $DUST_TEAM_ID")
```

### 5.1 Create a Thread

`POST /api/v1/threads`. The body is a discriminated union on `type`; use `type: "single"` for one Thread. `thread.name` is the only required thread field; `bundleId` (destination Folder) and `data` (initial typed fields) are optional. The response is `201` with `{ "created": [ ...thread records... ], "uploadResponses": [...] }`.

```bash
curl -fsS "$APID_URL/api/v1/threads" "${AUTH[@]}" \
  -H "Content-Type: application/json" \
  -d '{
    "type": "single",
    "thread": { "name": "Tire SZ3J-11-ZJ17", "description": "Production asset" },
    "data": [
      { "name": "Serial Number", "type": "text",   "value": { "text": "SZ3J-11-ZJ17" } },
      { "name": "Max PSI",       "type": "number", "value": { "number": 51 } }
    ]
  }'
```

```ts
// createOne unwraps the batch response to the single created record
const created = await client.threads.createOne({
  type: "single",
  thread: { name: "Tire SZ3J-11-ZJ17", description: "Production asset" },
  // bundleId: folderId,   // optional destination Folder
  data: [
    { name: "Serial Number", type: "text", value: { text: "SZ3J-11-ZJ17" } },
    { name: "Max PSI", type: "number", value: { number: 51 } },
  ],
});
console.log(created.threadId); // UUID

// Read it back: GET /api/v1/threads/{thread_id} → { thread, events }
const record = await client.threads.get(created.threadId);
```

Every write to a Thread is recorded as an event; reads return `{ thread, events }`.

### 5.2 Bind an Identifier to a Thread

`POST /api/v1/tags/bind`, `multipart/form-data` (scan images are uploaded as files/blobs). The payload is one of three variants, all requiring `threadId` and `tagType`:

- **Text identifiers** (`tagType`: `QR`, `BAR_CODE`, `DATA_MATRIX`, `NFC`): `data` is the decoded string value.
- **DUST scan image** (`tagType: "DUST"`): `data` is the captured image — a binary file part or a `data:image/(png|jpeg);base64,...` string.
- **DUST fingerprint** (`tagType: "DUST"` + `fingerprintId`): binds a previously extracted fingerprint (from `POST /api/v1/tags/extract`) instead of raw image data.

Optional: `tagDescription` (label), `options` (JSON string in multipart; for DUST image binds may include `enrollmentSessionId`, a client-generated UUID that groups multi-scan enrollment sessions).

```bash
# DUST scan image
curl -fsS "$APID_URL/api/v1/tags/bind" "${AUTH[@]}" \
  -F "threadId=$THREAD_ID" \
  -F "tagType=DUST" \
  -F "tagDescription=Inbound receiving scan" \
  -F "data=@scan.jpeg"

# Text identifier (QR)
curl -fsS "$APID_URL/api/v1/tags/bind" "${AUTH[@]}" \
  -F "threadId=$THREAD_ID" \
  -F "tagType=QR" \
  -F "data=SZ3J-11-ZJ17"
```

```ts
await client.tags.bind({
  threadId,
  tagType: "DUST",
  tagDescription: "Inbound receiving scan",
  data: scanBlob, // Blob/File, or a base64 data: URI string
});
```

The response contains the created `tag` record (its id, type, thread linkage, timestamps).

**Identify** — find which Thread a scan belongs to — is `POST /api/v1/tags/identify` (same multipart shape, no `threadId`; optional `searchTeamIds` JSON array narrows the search). The response is a union: `{ type: "identified", identified: { tag, thread, ... } }` for a confident match or `{ type: "matches", matches: [...] }` for candidates.

**Verify** — check a fresh scan against tags already bound to a Thread — is `POST /api/v1/tags/verify` (multipart: `threadId`, `tagType`, `data`, optional `tags` selection).

```ts
const result = await client.tags.identify({
  tagType: "DUST",
  data: scanBlob,
  searchTeamIds: [teamId], // optional
});
if (result.type === "identified") {
  console.log(result.identified?.thread?.threadId);
}
```

### 5.3 Upload a file and attach it to a Thread

Two paths. For small files, one request does everything:

```bash
# Simple upload: POST /api/v1/files (multipart)
curl -fsS "$APID_URL/api/v1/files" "${AUTH[@]}" \
  -F "file=@report.pdf" \
  -F "threadId=$THREAD_ID"
```

```ts
const [resource] = await client.files.upload({
  file,             // File
  threadId,         // optional: attach to this Thread
  // fieldId?, isPrivate?, expectedUpdatedAt?
});
```

For large files, use the resumable **tus 1.0.0 protocol** endpoint, then finalize. (These tus endpoints exist on the server at `/api/v1/files/upload` but are registered method-agnostically, so they may be missing from the generated endpoint index below — trust this section.) Steps:

1. **Create the upload** — `POST $APID_URL/api/v1/files/upload` with tus headers: `Tus-Resumable: 1.0.0`, `Upload-Length: <total bytes>`, and `Upload-Metadata` containing a base64-encoded `filename` (optionally `fieldId`, `threadId`). The `201` response's `Location` header is the upload URL; its last path segment is the upload id (a UUID) — this becomes `resId`.
2. **Upload chunks** — `PATCH <Location>` with `Tus-Resumable: 1.0.0`, `Upload-Offset: <bytes so far>`, `Content-Type: application/offset+octet-stream`, and the chunk bytes. Repeat until the offset reaches `Upload-Length`. (`HEAD <Location>` returns the current offset for resumption.)
3. **Finalize** — `POST /api/v1/files/finalize` with JSON `{ "requests": [{ "resId": "<upload id>", "filename": "...", "size": <bytes> }], "threadId": "<optional thread to attach to>" }`. This creates the resource record(s); the response is an array of resource records with signed URLs.

```bash
# 1. Create
LOCATION=$(curl -fsS -D - -o /dev/null "$APID_URL/api/v1/files/upload" "${AUTH[@]}" \
  -X POST \
  -H "Tus-Resumable: 1.0.0" \
  -H "Upload-Length: $(wc -c < report.pdf)" \
  -H "Upload-Metadata: filename $(printf %s report.pdf | base64)" \
  | awk 'tolower($1)=="location:" {print $2}' | tr -d '\r')
RES_ID="${LOCATION##*/}"

# 2. Upload (single chunk shown; repeat with advancing Upload-Offset for chunks)
curl -fsS -X PATCH "$LOCATION" "${AUTH[@]}" \
  -H "Tus-Resumable: 1.0.0" \
  -H "Upload-Offset: 0" \
  -H "Content-Type: application/offset+octet-stream" \
  --data-binary @report.pdf

# 3. Finalize and attach to a Thread
curl -fsS "$APID_URL/api/v1/files/finalize" "${AUTH[@]}" \
  -H "Content-Type: application/json" \
  -d '{
    "threadId": "'"$THREAD_ID"'",
    "requests": [{ "resId": "'"$RES_ID"'", "filename": "report.pdf", "size": '"$(wc -c < report.pdf)"' }]
  }'
```

The TypeScript client has no built-in tus transport — use any tus client (e.g. `tus-js-client`) pointed at `$APID_URL/api/v1/files/upload` with the same auth/context headers, then call the finalize endpoint (via raw `fetch`; `client.files.upload` covers only the simple-upload path). Related: `GET /api/v1/threads/{thread_id}/files` lists a Thread's files; `GET /api/v1/files/{resource_id}/download` downloads; `POST /api/v1/files/urls` returns short-lived signed URLs (store resource ids, not signed URLs).

### 5.4 Share a Thread with another Team

Sharing grants another Team `viewer` or `editor` access to a Thread or a bundle (Folder/Category). Cross-organization sharing requires an active **Connection** between the Teams — `GET /api/v1/teams/connected` lists the partner Teams you can share with.

```bash
# Create shares: POST /api/v1/sharing
curl -fsS "$APID_URL/api/v1/sharing" "${AUTH[@]}" \
  -H "Content-Type: application/json" \
  -d '{
    "items": [
      { "item": "thread", "id": "'"$THREAD_ID"'", "teamId": "'"$PARTNER_TEAM_ID"'", "relation": "viewer" }
    ]
  }'
```

```ts
await client.sharing.add({
  items: [{ item: "thread", id: threadId, teamId: partnerTeamId, relation: "viewer" }],
});
```

Related endpoints: `GET /api/v1/sharing?direction=in|out` lists grants (`out` = what you shared, `in` = shared with you); `PATCH /api/v1/sharing/{tuple_id}` changes a grant's relation; `DELETE /api/v1/sharing` removes grants; `GET /api/v1/sharing/access-summary?objectId=…&objectType=thread|bundle` resolves *effective* access (direct grants + Folder inheritance); `GET /api/v1/sharing/partner-inventory?teamId=…` shows everything shared with one partner.

### 5.5 Send a Shipment (transfer ownership)

A Shipment moves Thread ownership to a connected Team: build a draft manifest, send it, the receiver responds. Namespace is `/api/v1/transfers` (legacy naming — the product calls these Shipments).

**Sender side:**

```bash
# 1. Create a draft
TRANSFER_ID=$(curl -fsS "$APID_URL/api/v1/transfers" "${AUTH[@]}" \
  -H "Content-Type: application/json" \
  -d '{ "name": "July shipment to Acme" }' | jq -r '.id')

# 2. Add a manifest item (simplest form: one whole Thread as a unit)
curl -fsS "$APID_URL/api/v1/transfers/$TRANSFER_ID/items" "${AUTH[@]}" \
  -H "Content-Type: application/json" \
  -d '{ "kind": "unit", "threadId": "'"$THREAD_ID"'" }'

# 3. Send to the receiving Team
curl -fsS "$APID_URL/api/v1/transfers/$TRANSFER_ID/send" "${AUTH[@]}" \
  -H "Content-Type: application/json" \
  -d '{ "toId": "'"$PARTNER_TEAM_ID"'" }'
```

```ts
const transfer = await client.transfers.create({ name: "July shipment to Acme" });
await client.transfers.addItem(transfer.id, { kind: "unit", threadId });
await client.transfers.send(transfer.id, { toId: partnerTeamId });
```

Item bodies are a union: `{ kind: "unit", threadId }` for a single Thread (optionally narrowing `fields`/`resources`/`tags` to include), `{ kind: "assembly", threadId, parts? }` for an assembly Thread with its parts, or `{ kind, name, source: { type: "bundle", bundleId, recurse? } | { type: "threads", threadIds } }` for bulk selection. Manage items with `PATCH`/`DELETE /api/v1/transfers/{transfer_id}/items/{item_id}`.

**Receiver side** (calls run under the receiving Team's context headers):

```ts
// Inbox: GET /api/v1/transfers?box=inbox   (also box=outbox|sent, status/direction filters)
const inbox = await client.transfers.list({ box: "inbox" });

// Inspect before accepting: GET /api/v1/transfers/{transfer_id} and GET .../preview
const detail = await client.transfers.get(transferId);

// Respond: accept | reject | request_changes (reason required for change requests)
await client.transfers.respond(transferId, { value: "accept" });
// or: { value: "request_changes", reason: "Missing certs for lot 7" }
```

Lifecycle summary: `draft` → (send) → `sent` → receiver responds `accept` / `reject` / `request_changes`. A change request puts the Shipment back in the sender's hands (status `change_request`; still editable, re-send when fixed). Acceptance triggers `processing` and then completion — ownership of the manifest Threads moves to the receiver. The sender can `POST .../cancel` while in `draft`/`sent`/`change_request`. If processing fails retryably, `POST .../retry` or `POST .../abandon` (reason required); `POST .../start-from-prior-manifest` clones a stopped Shipment's manifest into a new draft. Conversation: `POST .../messages`.

## 6. Endpoint index

<!-- generated:api-endpoint-index:start -->
### Auth

- `GET /api/auth/token` — Get auth token
- `GET /api/auth/jwks` — Get JWKS
- `GET /api/v1/me/feature-flags` — Current feature flags
- `GET /api/v1/me` — Current user

### Bundles

- `GET /api/v1/bundles` — List bundles with cursor pagination
- `POST /api/v1/bundles` — Create a thread bundle
- `GET /api/v1/bundles/children` — List visible child bundles
- `GET /api/v1/bundles/count` — Count bundles
- `GET /api/v1/bundles/{bundle_id}` — Get bundle
- `PATCH /api/v1/bundles/{bundle_id}` — Update a bundle
- `DELETE /api/v1/bundles/{bundle_id}` — Delete a bundle
- `PATCH /api/v1/bundles/parent` — Set bundle parent
- `POST /api/v1/bundles/{bundle_id}/add` — Add a thread to the bundle
- `PATCH /api/v1/bundles/{bundle_id}/move` — Move threads to a bundle
- `DELETE /api/v1/bundles/{bundle_id}/threads/{thread_id}` — Remove a thread from a category bundle
- `PATCH /api/v1/bundles/{bundle_id}/children` — Set bundle children

### Certificate Forms

- `POST /api/v1/certificate-forms` — Create a Certificate Form.
- `GET /api/v1/certificate-forms` — List Certificate Forms.
- `GET /api/v1/certificate-forms/{formId}` — Get a Certificate Form.
- `PATCH /api/v1/certificate-forms/{formId}` — Update a Certificate Form.

### Certificates

- `POST /api/v1/certificates/preflight` — Preflight a Certificate Form against a Thread.
- `POST /api/v1/certificates/preflight/batch` — Batch preflight (read-only) for the Dice batch workflow.
- `POST /api/v1/certificates/preview-form` — Render a Certificate Form preview with placeholder data.
- `POST /api/v1/certificates/preview-form-thread` — Render a Certificate Form preview resolved against a Thread.
- `POST /api/v1/certificates/resolve-form-values` — Resolve a Certificate Form's bindings against a Thread (values only, no PDF).
- `POST /api/v1/certificates/generate` — Generate (issue) a single Certificate.
- `POST /api/v1/certificates/void` — Void a Certificate.
- `GET /api/v1/certificates` — List a Thread's Certificates.
- `GET /api/v1/certificates/{certificateId}` — Get a Certificate.

### Connections

- `POST /api/v1/connections` — Create a connection
- `GET /api/v1/connections` — List connections
- `GET /api/v1/connections/{code}` — Get a connection
- `DELETE /api/v1/connections/{code}` — Delete a connection
- `PATCH /api/v1/connections/confirm` — Confirm a connection
- `PATCH /api/v1/connections/pause` — Pause a connection
- `PATCH /api/v1/connections/cancel` — Cancel a connection
- `PATCH /api/v1/connections/resume` — Resume a paused connection
- `POST /api/v1/connections/amend/propose` — Propose a connection direction amendment
- `PATCH /api/v1/connections/amend/accept` — Accept a pending connection amendment
- `PATCH /api/v1/connections/amend/confirm` — Confirm an accepted connection amendment
- `PATCH /api/v1/connections/amend/cancel` — Cancel a pending connection amendment
- `PATCH /api/v1/connections/accept` — Accept a connection
- `PATCH /api/v1/connections/reject` — Reject a connection

### Events

- `GET /api/v1/events` — List events

### Fabric

- `GET /api/v1/fabric/threads/{thread_id}/graph` — Visible Fabric Graph for a thread
- `GET /api/v1/fabric/threads/{thread_id}/versions` — Fabric Thread Versions for a thread
- `GET /api/v1/fabric/links/{link_id}/context` — Current disclosed data for a Fabric Link
- `GET /api/v1/fabric/threads/{thread_id}/disclosure/push/context` — Source-side context for a Fabric disclosure push
- `GET /api/v1/fabric/links/{link_id}/resources/{res_id}/content` — Disclosed resource content for a Fabric Link
- `GET /api/v1/fabric/links/{link_id}/events` — Fabric Event Views for a Fabric Link
- `GET /api/v1/fabric/links/{link_id}/timeline` — Disclosure revision timeline for a Fabric Link
- `GET /api/v1/fabric/links/{link_id}/assembly` — Frozen Assembly Parts for a Fabric Link version
- `POST /api/v1/fabric/threads/{thread_id}/disclosure/preview` — Preview a disclosure revision or redaction
- `POST /api/v1/fabric/threads/{thread_id}/disclosure/revise` — Revise Fabric Disclosure
- `POST /api/v1/fabric/threads/{thread_id}/disclosure/redact` — Redact disclosed Fabric assets
- `POST /api/v1/fabric/threads/{thread_id}/cursor/preview` — Preview Fabric event cursor advancement
- `POST /api/v1/fabric/threads/{thread_id}/cursor/advance` — Advance Fabric event cursor
- `POST /api/v1/fabric/threads/{thread_id}/disclosure/push/preview` — Preview a Disclosure Push
- `POST /api/v1/fabric/threads/{thread_id}/disclosure/push` — Disclosure Push
- `GET /api/v1/fabric/notifications` — List Fabric notifications for the downstream owner
- `POST /api/v1/fabric/notifications/dismiss` — Dismiss pending Fabric notifications for the given threads
- `POST /api/v1/fabric/notifications/{notification_id}/preview` — Preview accepting a Fabric notification
- `POST /api/v1/fabric/notifications/{notification_id}/accept` — Accept a Fabric notification
- `POST /api/v1/fabric/notifications/{notification_id}/reject` — Reject a Fabric notification
- `GET /api/v1/fabric/links/{consumer_link_id}/disclosure/sources/{source_link_id}/sync` — Review a source link's disclosed state for adoption
- `POST /api/v1/fabric/links/{consumer_link_id}/disclosure/sources/{source_link_id}/sync/adopt` — Adopt a disclosed datum into the destination thread
- `POST /api/v1/fabric/links/{consumer_link_id}/disclosure/sources/{source_link_id}/sync/decline` — Decline (skip) a disclosed datum
- `POST /api/v1/fabric/links/{consumer_link_id}/disclosure/sources/{source_link_id}/sync/hide-local-copy` — Hide the local copy of a withdrawn disclosed datum
- `POST /api/v1/fabric/links/{consumer_link_id}/disclosure/sources/{source_link_id}/sync/adopt-batch` — Adopt several disclosed datums atomically

### Files

- `GET /api/v1/files` — Search Resources
- `POST /api/v1/files` — Upload a file
- `GET /api/v1/files/search` — Search Resources
- `POST /api/v1/files/urls` — Get signed URLs
- `GET /api/v1/files/signatures` — Search resource signatures
- `GET /api/v1/files/metrics` — File verification metrics
- `GET /api/v1/files/verification/{resource_id}/{thread_id}` — Get latest verification
- `PATCH /api/v1/files/verification/{resource_id}/{thread_id}` — Update file verification
- `POST /api/v1/files/verification/{resource_id}/{thread_id}/start` — Start verification
- `GET /api/v1/files/{resource_id}/download` — Download a file
- `POST /api/v1/files/finalize` — Finalize uploaded resources

### Metrics

- `GET /api/v1/summary` — Get summary count metrics

### Notifications

- `GET /api/v1/notifications` — List notifications

### Org Admin

- `POST /api/v1/org/teams` — Create one or more teams
- `GET /api/v1/org/teams` — List all teams in the organization
- `PATCH /api/v1/org/teams/{team_id}` — Update a team
- `DELETE /api/v1/org/teams/{team_id}` — Delete a team
- `POST /api/v1/org/teams/members` — Add/Update memberships
- `DELETE /api/v1/org/teams/members` — Remove memberships
- `GET /api/v1/org/teams/members` — List memberships

### Organizations

- `GET /api/v1/orgs/invites` — List organization invites
- `POST /api/v1/orgs/invites/accept` — Accept an organization invitation

### Sharing

- `GET /api/v1/sharing` — List sharing relationships
- `POST /api/v1/sharing` — Share a bundle or thread
- `DELETE /api/v1/sharing` — Remove sharing relationships
- `GET /api/v1/sharing/access-summary` — Effective access summary for a thread or bundle
- `GET /api/v1/sharing/partner-inventory` — Per-partner sharing inventory
- `PATCH /api/v1/sharing/{tuple_id}` — Update a sharing relationship

### Slices

- `POST /api/v1/slices` — Slice a thread (same-team derivation)
- `POST /api/v1/slices/batch` — Slice a thread into many derived threads in one operation
- `GET /api/v1/slices/{slice_id}` — Get a Slice with its Fabric Links

### System

- `GET /livez` — Liveness check
- `GET /readyz` — Readiness check
- `GET /healthz` — Health check

### Tags

- `POST /api/v1/tags/extract` — Extract a tag identifier.
- `POST /api/v1/tags/bind` — Bind a tag identifier.
- `POST /api/v1/tags/verify` — Verify one or more tags
- `POST /api/v1/tags/identify` — Identify a thread
- `POST /api/v1/tags/text` — Set the text on a tag.
- `POST /api/v1/tags/update` — Update identifier lifecycle state.
- `POST /api/v1/tags/unbind` — Unbind a tag from a thread.

### Teams

- `GET /api/v1/teams` — List teams
- `GET /api/v1/teams/connected` — List connected teams

### Templates

- `POST /api/v1/templates` — Create a thread template
- `GET /api/v1/templates` — List thread templates
- `PATCH /api/v1/templates/{templateId}` — Update a thread template
- `GET /api/v1/templates/{templateId}` — Get a thread template

### Thread Links

- `POST /api/v1/links` — Create thread links
- `GET /api/v1/links` — List thread links
- `GET /api/v1/links/available-threads` — List available threads for linking
- `DELETE /api/v1/links/{link_id}` — Unlink two threads

### Thread Relations

- `GET /api/v1/relations/{relation_id}` — Get relation definition
- `DELETE /api/v1/relations/{relation_id}` — Delete relation definition
- `PATCH /api/v1/relations/{relation_id}` — Update relation definition
- `GET /api/v1/relations` — List relation definitions
- `POST /api/v1/relations` — Create relation definition
- `POST /api/v1/relations/count` — Get relation link counts

### Threads

- `GET /api/v1/assemblies` — List the assembly Threads the caller can view
- `GET /api/v1/assemblies/{assembly_id}/parts` — List the direct Parts of an assembly Thread
- `POST /api/v1/assemblies/{assembly_id}/parts` — Attach a Part to an assembly Thread
- `DELETE /api/v1/assemblies/{assembly_id}/parts` — Detach a Part from an assembly Thread
- `PATCH /api/v1/assemblies/{assembly_id}/parts/protection` — Set Part protection relation
- `PUT /api/v1/assemblies/{assembly_id}/thread-fields` — Set or clear a thread-valued assembly field
- `PATCH /api/v1/assemblies/{assembly_id}/kind` — Convert a Thread between unit and assembly kind
- `GET /api/v1/assemblies/{assembly_id}/rolled-up-parts` — List transitive Parts for an assembly Thread
- `POST /api/v1/threads` — Create one or more threads
- `GET /api/v1/threads` — List threads with cursor pagination.
- `GET /api/v1/threads/count` — Count threads with optional filters.
- `POST /api/v1/threads/permissions` — Get user permissions on threads.
- `POST /api/v1/threads/{thread_id}` — Update thread field data
- `GET /api/v1/threads/{thread_id}` — Get thread data.
- `POST /api/v1/threads/{thread_id}/data` — Update thread field data
- `GET /api/v1/threads/{thread_id}/data/archived` — List archived thread field data
- `POST /api/v1/threads/{thread_id}/data/restore` — Restore archived thread field data
- `POST /api/v1/threads/{thread_id}/presence` — Heartbeat presence and list the current viewers of a thread
- `GET /api/v1/threads/{thread_id}/access` — List access contexts for threads.
- `PATCH /api/v1/threads/archive` — Archive a set of threads.
- `PATCH /api/v1/threads/restore` — Restore archived threads.
- `PATCH /api/v1/threads/{thread_id}/thumbnail` — Set thread's thumbnail
- `POST /api/v1/threads/{thread_id}/thumbnail` — Upload thread thumbnail.
- `GET /api/v1/threads/{thread_id}/shared` — Thread shared with
- `GET /api/v1/threads/{thread_id}/files` — List resources for a Thread
- `GET /api/v1/threads/{thread_id}/files/{res_id}` — Get a file resource for a Thread
- `POST /api/v1/threads/{thread_id}/files/{res_id}` — Update a thread resource
- `GET /api/v1/threads/{thread_id}/links` — List thread links
- `POST /api/v1/imports/plan` — Dry-run an Assembly Import Package into an Import Plan
- `POST /api/v1/imports/commit` — Commit an Assembly Import Package all-or-nothing

### Transfers

- `GET /api/v1/transfers` — List transfers for the active team (status views or mail-box)
- `POST /api/v1/transfers` — Create a draft Transfer
- `GET /api/v1/transfers/{transfer_id}` — Get a Transfer with its Fabric Manifest items
- `GET /api/v1/transfers/{transfer_id}/preview` — Authoritative Transfer Preview for a sent Transfer
- `GET /api/v1/transfers/{transfer_id}/preview/resources/{res_id}/content` — Offered Resource content for a Transfer Preview
- `GET /api/v1/transfers/{transfer_id}/preview/graph` — Pre-acceptance provenance graph for a sent Transfer
- `GET /api/v1/transfers/{transfer_id}/preview/threads/{source_thread_id}/events` — Disclosed transaction history for an offered Thread
- `POST /api/v1/transfers/{transfer_id}/send` — Send a draft Transfer to a receiving team
- `POST /api/v1/transfers/{transfer_id}/respond` — Respond to a received Transfer (accept / reject / request changes)
- `POST /api/v1/transfers/{transfer_id}/messages` — Post a message on a Transfer's conversation
- `POST /api/v1/transfers/{transfer_id}/cancel` — Cancel a draft, sent, or change-requested Transfer
- `POST /api/v1/transfers/{transfer_id}/retry` — Retry a failed Transfer
- `POST /api/v1/transfers/{transfer_id}/abandon` — Abandon a failed Transfer
- `POST /api/v1/transfers/{transfer_id}/start-from-prior-manifest` — Start a new shipment from a stopped shipment's manifest
- `POST /api/v1/transfers/{transfer_id}/items` — Add a Manifest Item to a draft Transfer
- `PATCH /api/v1/transfers/{transfer_id}/items/{item_id}` — Update a Manifest Item on a draft Transfer
- `DELETE /api/v1/transfers/{transfer_id}/items/{item_id}` — Remove a Manifest Item from a draft Transfer
- `PUT /api/v1/transfers/{transfer_id}/primary-thread` — Set the primary Thread of a draft Transfer

### Users

- `GET /api/v1/users/profile` — Get current profile
- `PATCH /api/v1/users/profile` — Update profile
- `GET /api/v1/users/{user_id}` — Get a user's profile
<!-- generated:api-endpoint-index:end -->

## 7. Verification checklist for generated integrations

- Every endpoint path/method you emit must exist in the index above (or in the spec at `/api/openapi.json`); do not invent endpoints.
- All `/api/v1/*` calls carry `Authorization: Bearer`; all except `/api/v1/me` also carry `Dust-Ctx-Org-Id` (and usually `Dust-Ctx-Team-Id`).
- API keys and bearer tokens appear only in server-side code and environment/secret storage.
- Error handling branches on the stable `code` field, never on `message` text.
- On `401`, re-exchange the API key at `GET /api/auth/token` and retry once; refresh proactively off the exchange response's `expiresIn` (~60 s margin) so long jobs never run into mid-request expiry.
