---
name: create-railcode-app
description: Build, modify, debug, and deploy Railcode static apps/tools end-to-end. Use when creating a Railcode app from an idea, using the Railcode CLI, wiring the zero-config SDK globals, explaining Railcode auth/data "magic", testing with railcode dev, configuring access policies, or deploying apps to a Railcode server.
---

# Create Railcode App

## Core Workflow

Start from the Railcode workspace root when possible. A normal app-builder loop is:

```bash
railcode init my-tool
cd apps/my-tool
railcode dev
railcode deploy
```

If `railcode` is unavailable in a source checkout, build and link the CLI first:

```bash
cd cli
npm install
npm run build
npm link
```

Use lowercase app names with digits and dashes only. `railcode init <tool>` creates source in `apps/<tool>` and deployable static output in `tools/<tool>`. Treat `apps/<tool>` as the source of truth; `tools/<tool>` is build output unless the project is an intentionally hand-written static tool.

## Decide What To Load

Load only the reference needed for the task:

- [CLI workflow](references/cli-workflow.md): exact `railcode` commands and local dev/deploy behavior.
- [Platform magic](references/platform-magic.md): how same-origin auth, `/_api/sdk.js`, tool identity, access policies, KV/files, SQL, and LLM work.
- [App patterns](references/app-patterns.md): implementation patterns for React/Vite apps, SDK wrappers, data modeling, SQL, LLM, and frontend expectations.
- [Deployment](references/deployment.md): server setup, DNS, app deploys, platform deploys, access policy deploys, and verification.

## Implementation Rules

Build the app as a static browser app. Do not add app-specific backend services, API keys, auth code, or hardcoded Railcode URLs unless the user explicitly asks for platform work. Browser code should call same-origin `/_api/*` through the Railcode SDK.

Use the starter's wrappers in `src/lib/railcode.ts` after `loadRailcodeSdk()` has loaded `/_api/sdk.js`. The global SDK surface is `me()`, `toolUsers()`, `db.collection()`, `files`, `connections()`, `sql()`, and `llm`.

Model data intentionally:

- KV is scoped per tool and shared by that tool's allowed users. Prefix keys with the logged-in user if the app needs per-user records.
- Files are scoped per tool. File API names cannot contain `/`; encode hierarchy in metadata or key names instead.
- SQL connections are admin-configured server-side and read-only. Always use placeholders plus params.
- LLM provider/model/API key are admin-configured server-side. Send `metadata` for audit and attribution.

## Local Development

Run `railcode dev` from `apps/<tool>` or any directory with a `railcode.json`. It serves the app at `http://127.0.0.1:7331`, runs the asset dev server when applicable, serves the SDK at `/_api/sdk.js`, and stores local KV/files under `~/.railcode/dev/<tool>`.

Local dev mocks identity, tool users, KV, and files. It forwards backend-backed APIs such as SQL, connections, and LLM to the configured Railcode API when the CLI has a saved API token.

## Validation

Before handing off a new or changed app, run the app's normal build:

```bash
cd apps/<tool>
npm run build
```

When changing the CLI, SDK, backend, deploy scripts, or platform behavior, run the relevant project checks in addition to the app build. Typical checks are `npm run build` in changed Node packages and, for backend changes, `cd backend && uv run pytest && uv run ruff check`.

## Deployment Rule Of Thumb

Use `railcode deploy` from the app directory for day-to-day static app publishes. Use `./deploy/deploy tools` when deploying all built tools from the platform repo. Use `./deploy/deploy backend`, `caddy`, or `all` only when changing platform infrastructure, backend routes, the admin UI, Caddy config, or the bundled SDK.
