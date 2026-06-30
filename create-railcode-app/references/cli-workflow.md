# CLI Workflow

Use this reference when an agent needs exact Railcode CLI behavior, local dev behavior, or
app deploy behavior on the **multi-tenant** Railcode platform.

The CLI ships as the npm package **`railcode`**. Its full command set is:

```
railcode login [--api-url <url>]              Sign in (browser) and mint a personal API token
railcode init <app> [--template static|react] Scaffold a new app directory
railcode dev [--port <n>] [--asset-port <n>] [--reset]   Run the app locally against an emulated /_api
railcode deploy                               Build (if configured) and deploy the app here
railcode design-system                        Print your org's design-system guidance (markdown)
railcode --version
railcode --help
```

There is **no** `railcode upgrade`, `railcode get`, `railcode network`, `railcode db`, or
`railcode access` — those were commands of the older single-tenant CLI. Upgrade the CLI
through your package manager (`npm install -g railcode@latest`).

## Install Or Link The CLI

Install the published CLI globally:

```bash
npm install -g railcode@latest        # or: pnpm add -g railcode@latest
```

In a multi-tenant Railcode source checkout (the repo uses **pnpm**):

```bash
cd cli
pnpm install
pnpm build              # produces dist/index.js (the `railcode` bin)
pnpm link --global      # or run it directly: node dist/index.js <command>
```

The CLI stores config at `${RAILCODE_HOME:-~/.railcode}/config.json` (dir `0700`, file
`0600`):

```json
{ "apiUrl": "...", "email": "...", "apiToken": "rc_...", "tokenPrefix": "rc_...",
  "orgUuid": "...", "orgSlug": "...", "appHostStrategy": "two_label" }
```

API-URL resolution for every command: `--api-url` flag > `RAILCODE_API_URL` env > saved
config > prompt (the prompt default is the dev URL `http://api.127.0.0.1.nip.io`). Set
`RAILCODE_API_TOKEN` to override the saved token in CI. On a `401`, the saved token is
cleared and you're told to `railcode login` again.

## Log In

```bash
railcode login [--api-url <url>]
```

Login is **browser-based** (not an email/password prompt). The CLI:

1. Starts a localhost HTTP callback and prints a browser authorization link.
2. You open it; the browser does normal dashboard auth and approves the CLI.
3. The CLI exchanges the one-time code for a long-lived, revocable **personal API token**,
   resolves your organization, and saves everything to `~/.railcode/config.json`.

Browser login needs a TTY. In non-interactive environments set `RAILCODE_API_TOKEN`
instead. If you have no organization yet, finish onboarding in the dashboard, then run
`railcode login` again so the org is saved (deploy needs it).

## Create An App

```bash
railcode init <app> [--template static|react]
```

Behavior:

- Validates the app slug against `^[a-z0-9][a-z0-9-]{0,62}$` (a DNS label: lowercase
  letters, digits, dashes).
- Scaffolds a **single self-contained directory `./<app>/`** — there is no
  `apps/`/`app-bundles/` workspace split, and no template repo is copied.
- Refuses to scaffold into a non-empty directory.

Templates:

- **`static`** (default) — a no-build app: `index.html` that loads `/_api/sdk.js` and demos
  `await me()` + `db.collection().put/get`, plus `railcode.json` with `{ "app": "<slug>",
  "dist": "." }`. No dependencies, no build step.
- **`react`** — a React 19 + Vite 7 + Zustand 5 + TypeScript starter that builds to `dist/`,
  with `railcode.json` `{ "app": "<slug>", "build": "pnpm run build", "dist": "dist" }`.
  Run `pnpm install` before `railcode dev`/`railcode deploy`.

Treat the starter as functional scaffolding, not a style guide.

## Local Dev

```bash
railcode dev [--port <n>] [--asset-port <n>] [--reset]
```

Run it from the app directory (any directory with a `railcode.json` that has an `"app"`
slug). Behavior:

- Serves the app on a single loopback origin, starting at `http://127.0.0.1:7331` and
  climbing (`7332`, …) when the port is busy. Print-and-open the URL it reports.
- **Static mode**: serves files straight from the app root (the deploy resolution mirrored).
- **Asset mode**: when `package.json` has a `dev` script (or `railcode.json` sets
  `dev.command`), the CLI runs the app's own dev server (Vite) and reverse-proxies it,
  tunnelling the HMR WebSocket. `--asset-port` / `railcode.json` `dev.port` set the starting
  Vite port (default `5173`). It does **not** install dependencies for you — if
  `node_modules` is missing it tells you to run `pnpm install` first.
- `--reset` wipes this app's local KV/files before starting.

`railcode dev` emulates the `/_api/*` data plane on local disk and proxies the rest to your
real instance:

- `GET /_api/sdk.js` — serves the bundled SDK from the CLI package or source checkout.
- `GET /_api/me` — synthetic identity `{ user, app, org }` (user `dev@localhost`).
- `GET /_api/app-users` — a single synthetic member.
- `GET /_api/config/design-system` — your org's **real** configured markdown when logged in;
  empty otherwise (never errors).
- `/_api/kv/*` — JSON KV stored under `~/.railcode/dev/<instance>/<app>/kv.json`, queried by
  the same engine production uses.
- `/_api/files*` — bytes under `~/.railcode/dev/<instance>/<app>/files/`, metadata in
  `files.json`.
- `/_api/connections`, `/_api/sql`, `/_api/llm/generate`, `/_api/llm/stream` — **proxied to
  the real instance** with your saved token (app-scoped under
  `/api/organizations/{org}/apps/{app}/…`). These hit the org's real provider, quota, and
  databases — **real spend and real data**. The first `llm`/`sql` call creates the app
  server-side if it doesn't exist yet; a load-time `GET /connections` only resolves an
  existing app, never creates one.

When you're **not logged in**, `connections`/`service-connectors` degrade to empty and
`llm`/`sql` return `503` (never `401`, which the SDK would treat as a session lapse and
reload-loop on). The startup banner states which mode you're in.

The local state directory is namespaced by `(instance, org)` so two orgs' same-slug apps
never share KV/files. Concurrent `railcode dev` sessions for the same app/org share that
directory.

## Read The Design System

```bash
railcode design-system
```

Prints your org's configured design-system markdown straight to stdout (so it pipes/feeds
cleanly into an agent). Needs a logged-in CLI; resolves the server like every other command.
Returns empty when no admin has configured a design system for the org.

## Deploy An App With The CLI

```bash
railcode deploy
```

Deploy behavior:

- Requires a `railcode.json` with an `"app"` slug in the current directory.
- Resolves the output directory, then runs a build command when needed (see resolution
  below), and uploads every file in the output dir (which must contain a root `index.html`)
  to the org-scoped multipart deploy API
  (`POST /api/organizations/{org}/apps/{appUuid}/deploy`).
- Skips `.git`, `node_modules`, `.DS_Store`, and (at the app root) `railcode.json`,
  `package.json`, `pnpm-lock.yaml`.
- The app is **created-or-resolved by slug in your saved org**; first deploy creates it with
  the default **`organization`** access mode.
- Uses the saved API token (or `RAILCODE_API_TOKEN`); clears the token and asks you to log in
  again on `401`.
- Prints the live URL `http://<app>.<org>.<serving-domain>/` after upload.

Deploy output resolution order:

1. `railcode.json` `"dist"` wins (use `"."` for a no-build static app). `"build"` still runs
   first if also set.
2. Otherwise `railcode.json` `"build"` runs and `dist/` is uploaded.
3. Otherwise a `package.json` with a `build` script runs `pnpm run build` and uploads
   `dist/`.
4. Otherwise a root `index.html` can be deployed interactively (a `y/N` prompt); for CI set
   `"dist": "."`.

There are **no `--private`/`--public` flags** and no `deploy.access`/`deploy.apiUrl` manifest
keys. The `railcode.json` schema is `{ app, build?, dist?, dev?: { root?, command?, port? } }`.

## App Access (no CLI command)

A new app defaults to **`organization`** access — every member of your org may open it.
Access is **not** managed from the CLI on the multi-tenant platform. The owner or an org
admin changes it in the **admin UI**, or via the access API:

```text
GET  /api/organizations/{org}/apps/{app}/access
PUT  /api/organizations/{org}/apps/{app}/access      { "mode": "...", ... }
```

Modes: `organization` (every org member, the default), `private` (owners only), `restricted`
(owners plus explicitly-granted members). Org admins/owners bypass per-app access entirely.
See [platform-magic.md](platform-magic.md) for the access model.
