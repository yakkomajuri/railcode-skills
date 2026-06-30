# Platform Magic

Use this reference when an agent needs to explain or rely on Railcode's zero-config auth,
data, SDK, access, SQL, service-connector, or LLM behavior on the **multi-tenant** platform.

## Request Routing

Railcode is multi-tenant: every app belongs to an **organization**, and apps are static apps
served from per-org subdomains. With the default two-label host strategy:

```text
https://<app_slug>.<org_slug>.<BASE_DOMAIN>/      e.g. https://notes.acme.railcode.dev/
```

(A platform configured for `single_label` serving drops the org label:
`https://<app_slug>.<BASE_DOMAIN>/`. The CLI records which strategy your instance uses and
prints the right live URL.)

The same host also exposes the platform data plane at **`/_api/*`**. Because the browser
calls same-origin URLs, app code needs no CORS config, no API URLs, and no credentials. The
backend scopes every `/_api/*` call server-side to `(org, app)` from the request's host +
session — the app never names itself or its org in client code.

Reserved subdomains (the dashboard/login parent, `api`, `admin`, etc.) cannot be app slugs.

## Auth And App Identity

Railcode derives context from server-controlled request state, not from browser JavaScript:

- **Serving (app pages + `/_api/*`)** is gated by a **parent-scoped serving cookie**
  (`Domain=.<parent>`), the only cross-subdomain credential. Caddy `forward_auth` verifies
  org membership and per-app access before any app byte or `/_api/*` response is served. App
  identity comes from the `Host` header.
- **Dashboard/API** (the control plane) uses **bearer tokens** — `Authorization: Bearer
  <token>` — with no cookies/CSRF.
- **CLI** uses a long-lived, revocable **personal API token**. CLI/token-driven app routes
  are explicit and org-scoped: `/api/organizations/{org}/apps/{app}/...` (deploy, SQL, LLM,
  connections in `railcode dev`).

The "magic": frontend code just calls same-origin `/_api/*`, and the backend already knows
both who is calling and which app/org is calling.

## SDK Surface

Every app loads the SDK with a same-origin script tag:

```html
<script src="/_api/sdk.js"></script>
```

On load it attaches a fixed set of globals to `window`. There is no `loadRailcodeSdk()`
bootstrap and no wrapper module — call the globals directly (in TypeScript, `declare` them).
Every call is same-origin against `/_api/*`, credentialed by the serving cookie:

```js
const who   = await me();          // { user:{uuid,name,email}, app:{uuid,slug,name}, org:{uuid,slug,name} }
const people = await appUsers();   // [{ uuid, name, email, role }] — the app's org members
const ds    = await designSystem();// the org's design-system guidance (markdown string)

await db.collection("notes").put("n1", { text: "hi", n: 3 });
await db.collection("notes").get("n1");

await files.upload("logo.png", blob, "image/png");
const url = files.url("logo.png");

const rows = await postgres("analytics").runSQL("select * from orders where id = $1", [id]);
const conns = await dataConnectors();             // [{ engine, name }]

const resp = await connector("stripe").fetch("/v1/charges", { method: "POST", body });
const svc  = await serviceConnectors();           // [{ name, description, auth_type, allowed_methods }]

const out  = await llm.generate("Summarize this record.", { metadata: { feature: "summary" } });
```

The globals are exactly: `me`, `appUsers`, `designSystem`, `db`, `files`, `postgres`,
`dataConnectors`, `connector`, `serviceConnectors`, `llm`. Notes:

- `me()` returns nested objects. Use **`me().user.uuid`** as the stable per-user key for
  ownership/permissions/KV prefixes; `me().user.name`/`.email` are for display.
- Only the **postgres** engine exists today. There is no `mysql` namespace and no
  `databaseConnectors()` alias (those were single-tenant CLI surfaces).

The SDK also ships a hidden live inspector drawer that logs every call (`db`, `files`, `llm`,
`postgres`, `connector`, `me()`, `appUsers()`, `designSystem()`) with a pending → ok/error
transition and timing. It has no on-screen affordance — toggle it with ``Ctrl+` `` (control +
backtick). It is present in production too, just dormant until opened. Do not swallow SDK
errors; surface useful error states in the app.

## Access Policies

Access governs **who, within the app's org, may open the app** — distinct from org membership
(non-members never reach an org's apps at all). A user who can't access an app gets a **404**
(existence hidden), never a bare 403.

An app's `access_mode` is one of:

- **`organization`** — every org member (the **default** for a newly created/deployed app).
- **`private`** — owners only.
- **`restricted`** — owners plus explicitly-granted members.

Org admins/owners **bypass** per-app access — they see and manage every app in the org.
Access is read/set in the **admin UI** or via `GET`/`PUT
/api/organizations/{org}/apps/{app}/access`. There is **no `railcode access` CLI command** on
the multi-tenant platform.

`appUsers()` returns the app's org members (`{ uuid, name, email, role }`); use it for
assignee pickers, mentions, and display, not as an authorization check (the server already
enforces access).

## KV Store

`db.collection(name)` is a per-app JSON key/value store, shared across that app's allowed
users. Mutations: `put(key, value)`, `get(key)`, `delete(key)`, `list()`.

Use shared keys for collaboration surfaces, and **user-prefixed keys** for per-user private
state (prefix by the stable user uuid):

```js
await db.collection("messages").put(messageId, message);            // shared

const who = await me();
await db.collection("drafts").put(`${who.user.uuid}:${draftId}`, draft);  // per-user
```

For anything that can grow past a small list, use the query builder instead of `list()`:

```js
const page = await db.collection("messages")
  .where("roomId", "eq", roomId)
  .orderBy("updatedAt", "desc")
  .page(1, 50);                       // page(pageNumber, size?) — size is a number

const mine    = await db.collection("drafts").prefix(`${who.user.uuid}:`).page();
const changed = await db.collection("messages").updatedSince(lastSeenIso).count();
const first   = await db.collection("messages").where("pinned", "eq", true).first();
```

Query semantics (the dev engine is a byte-for-byte port of the backend, so dev matches prod):

- `where(field, op, value)` operators are the **string names** `eq`, `ne`, `gt`, `gte`,
  `lt`, `lte`, `in` — not symbols. `in` takes an array.
- `field` is a virtual field — `key` (string) or `updatedAt`/`updated_at` (datetime; no `in`)
  — or a **dotted JSON path** into the stored value (`done`, `assignee.email`). The
  comparison is typed by the operand (number/boolean/string), not lexicographic. A
  missing/null field excludes the row.
- `prefix()` does a byte-range key scan; `updatedSince()` is inclusive, `updatedBefore()` is
  exclusive; `orderBy(field, "asc"|"desc")` defaults to `key`/ascending.
- Paging is 1-based; **default size 100, max 500**. `list()` returns the first page of all
  rows (shaped `{ key, value, updated_at }`); prefer `page()`/`first()`/`count()` for large
  collections.

## Files

Files are per-app objects:

```js
await files.upload("logo.png", blob, "image/png");   // upload(name, data, contentType?)
const url = files.url("logo.png");                    // GET URL (use in <img src>, fetch)
const entries = await files.list();                   // [{ name, content_type, size, updated_at }]
await files.delete("logo.png");
```

`data` may be a `Blob`, `ArrayBuffer`, or typed array; the content type is inferred from a
`Blob` when omitted. File names cannot start with `/`, contain `\`, or use `.`/`..` traversal
segments. Don't model folders in the name — keep names flat and store logical hierarchy in
KV. Served bytes are returned `Content-Disposition: attachment` + `nosniff`.

## SQL (Postgres)

An admin registers global Postgres connections server-side. Browser apps query them through a
per-engine namespace without ever seeing DSNs or passwords:

```js
const rows = await postgres("analytics").runSQL(
  "select id, total from orders where customer_id = $1",
  [customerId],
);
// rows is an array of row objects, plus rows.columns / rows.rowcount / rows.truncated
```

Rules:

- `postgres('name').runSQL(sql, params)` targets a named connection; `postgres.runSQL(...)`
  (no name) targets the connection named `default`.
- Only **postgres** is supported today; the backend rejects other engines.
- Treat SQL as **read-only**. Always use `$1, $2, …` placeholders + a params array; never
  concatenate user input.
- Call `dataConnectors()` to discover configured connections as `{ engine, name }`. Expect it
  to be empty in unauthenticated local dev (and show a clean empty state).
- The server caps result rows (the envelope's `truncated` flag tells you when it did).

## Service Connectors (third-party HTTP)

A **service connector** is an admin-configured proxy to a downstream SaaS API (Stripe,
Mixpanel, …). The app names a connector and supplies only the method, path, and body; the
backend pins the host and injects the credential the app never sees:

```js
const conn = connector("stripe");
const resp = await conn.fetch("/v1/charges?limit=3");                 // GET by default
const post = await conn.fetch("/v1/charges", { method: "POST", body: "amount=500&currency=usd" });
if (resp.ok) {
  const data = await resp.json();   // also: await resp.text(), resp.status, resp.headers
}
```

`serviceConnectors()` lists what this app may call as
`{ name, description, auth_type, allowed_methods }`. The proxy rejects a method not in
`allowed_methods` (405) and strips upstream auth/`Set-Cookie` headers; the response is
truncated at a size limit (`resp.truncated`).

## LLM

LLM setup is admin-controlled — apps do not choose providers, models, or API keys:

```js
const result = await llm.generate("Classify this customer.", {
  output: {
    type: "json",
    schema: {
      type: "object",
      additionalProperties: false,
      properties: {
        status: { type: "string", enum: ["healthy", "risk", "unknown"] },
        reason: { type: ["string", "null"] }
      },
      required: ["status", "reason"]
    }
  },
  metadata: { feature: "customer-health", object_type: "customer", object_id: customerId }
});
// result: { text, output, usage, cost, provider, model, finishReason, requestId }
```

- Input is a prompt string **or** a `messages: [{ role, content }]` array. Options:
  `model`, `system`, `output`, `temperature`, `maxOutputTokens`, `metadata`.
- `llm.generate()` supports `{ output: { type: "json", schema } }`. JSON schemas run in
  **strict mode**: every object must set `additionalProperties: false` and list **all** keys
  in `required` — make optional fields nullable (`{ type: ["string", "null"] }`) rather than
  omitting them.
- `llm.stream(input, opts)` is an async iterator of `{ type: "text" }` / `{ type: "done" }` /
  `{ type: "error" }` events; it is **text-only** and rejects JSON output client-side.
- Always send `metadata` for audit/attribution. Expect daily token caps, provider timeouts,
  and input limits enforced server-side; render those failures as normal app states and do
  not retry indefinitely.

## Local Dev Bridge

`railcode dev` preserves the same SDK calling style locally (see
[cli-workflow.md](cli-workflow.md) for full behavior):

- Identity (`me`), `appUsers`, KV, and files are **emulated on local disk** under
  `~/.railcode/dev/<instance>/<app>/`. The KV query engine is a port of the backend, so
  `where`/`prefix`/`orderBy`/`page`/`first`/`count` behave exactly as in production.
- `designSystem()`, `postgres().runSQL()`, `dataConnectors()`, `serviceConnectors()`,
  `connector().fetch()`, and `llm` **forward to the real instance** when the CLI has a saved
  token — real provider, quota, databases, and connectors (real spend + data).
- Not logged in: `dataConnectors()`/`serviceConnectors()` return empty and
  `postgres().runSQL()`/`llm` return `503` (never `401`). The startup banner says which mode
  you're in, so you don't have to fire a request to find out.

This lets agents build most app behavior without a live server, then layer on
production-backed SQL/LLM/connectors once credentials and access are available.
