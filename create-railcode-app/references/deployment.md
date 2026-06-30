# Deployment

Use this reference when deploying a Railcode app, setting app access, or standing up a local
Railcode stack to test against. This is the **multi-tenant** platform; deploys are
org-scoped.

## App Deploy

From the app directory:

```bash
railcode deploy
```

`railcode deploy` reads `railcode.json` (`{ app, build?, dist? }`), runs the build command
when one is configured, and uploads the resolved output directory (which must contain a root
`index.html`) over HTTP to the configured Railcode instance. It does not restart any backend
service.

- **Which server** — resolved from `--api-url`, then `RAILCODE_API_URL`, then the saved CLI
  config from `railcode login` (prompt default `http://api.127.0.0.1.nip.io`). There is no
  `deploy.apiUrl` manifest key.
- **Auth** — the saved personal API token, or `RAILCODE_API_TOKEN` for non-interactive
  deploys. On a `401` the token is cleared and you're asked to `railcode login` again.
- **Where it lands** — the app is created-or-resolved by slug in your saved org
  (`POST /api/organizations/{org}/apps/{appUuid}/deploy`, a multipart upload). A deploy
  writes an immutable tree and going live is a single pointer flip server-side.
- **Output resolution** — `railcode.json` `dist` wins (`"."` = no-build static); else
  `build` + `dist/`; else a `package.json` build script runs `pnpm run build` and `dist/` is
  uploaded; else an interactive root-`index.html` deploy.

After upload the CLI prints the live URL: `http://<app>.<org>.<serving-domain>/`
(e.g. `https://my-app.acme.railcode.dev/`).

There are **no `--private`/`--public` flags** and no `deploy.access` manifest key.

## App Access

A newly created app defaults to **`organization`** access (every member of the org may open
it). Access is **not** managed from the CLI on the multi-tenant platform — there is no
`railcode access` command. The owner or an org admin changes it in the **admin UI**, or via
the access API:

```text
GET  /api/organizations/{org}/apps/{app}/access
PUT  /api/organizations/{org}/apps/{app}/access      { "mode": "organization" | "private" | "restricted", ... }
```

Modes:

- `organization` — every org member (default).
- `private` — owners only.
- `restricted` — owners plus explicitly-granted members.

Org admins/owners bypass per-app access (they manage every app). A user who lacks access
sees a 404, not a 403.

## Local Stack To Test Against

To exercise a real backend locally (auth, design system, SQL, LLM, connectors) bring the
whole stack up with Docker behind Caddy on the `127.0.0.1.nip.io` parent:

```bash
make selfhosted     # or: make cloud     (single-tenant-style vs multi-tenant edge config)
make logs           # follow logs
make down           # stop;  make reset / make down-volumes to wipe local data
```

Then `railcode login` against the printed dev API URL (default `http://api.127.0.0.1.nip.io`)
and deploy into it. Modes, the two-terminal `uv` + `pnpm` dev flow, and every environment
variable are documented in the repo's **`docs/deployment.md`**; the developer quickstart is in
`README.md`.

## Running / Updating The Platform Itself

Operating the Railcode **server** is separate from deploying an app onto it, and only matters
when the user runs the platform. The multi-tenant platform runs on **AWS Fargate behind a
Cloudflare edge** (infra-as-code in `infra/terraform/`, driven with the `tofu`/OpenTofu CLI),
with `docker-compose.prod*.yml` + `Caddyfile.prod*` for self-managed hosts. Do not attempt a
platform update for an app-only change. For the full deployment model — modes, production,
TLS/serving, and env vars — read the repo's `docs/deployment.md` and `docs/architecture.md`;
don't reconstruct it here.

## Post-Deploy Verification

After an app deploy, open the printed URL `https://<app>.<org>.<BASE_DOMAIN>/` and check:

- Unauthenticated visitors are sent through the platform login (the serving gate), not a
  custom app login.
- The app loads `/_api/sdk.js` with no CORS or mixed-content errors.
- `me()` returns the expected user, app, and org.
- KV and file reads/writes succeed.
- SQL / LLM / service-connector features show configured, empty, or disabled states cleanly
  (never a raw error or a hang).
- The app's access mode in the admin UI matches the intended audience.
