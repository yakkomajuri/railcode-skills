# Deployment

Use this reference when deploying a Railcode app or setting up/updating a Railcode server.

## First-Time Server Install

Run the installer on the server:

```bash
curl -fsSL https://raw.githubusercontent.com/railcode/railcode/main/install.sh | bash
```

From an existing checkout:

```bash
bash install.sh
```

The installer verifies Docker, Docker Compose v2, and curl; prompts for the base domain; explains required DNS records; asks for storage, admin, and optional LLM settings; writes `.env` and `.railcode/Caddyfile`; then runs `docker compose up -d --build`.

DNS records:

```text
auth.<domain>  A  <server-ip>
admin.<domain> A  <server-ip>
*.<domain>     A  <server-ip>
```

Use `127.0.0.1.nip.io` as the base domain for local HTTP installs. Other domains are treated as production HTTPS installs with Caddy on-demand TLS, so ports 80 and 443 must be reachable from the public internet.

Production setup decisions:

- Platform DB: SQLite by default, Postgres with `STORAGE_BACKEND=postgres` and `POSTGRES_DATABASE_URL`.
- File bytes: local disk by default, S3-compatible storage with `OBJECT_STORAGE_BACKEND=s3`.
- Auth/admin: set a strong `SECRET_KEY`, change the bootstrap admin password, and avoid demo credentials.
- LLM: configure provider, API key, model, and `LLM_DAILY_TOKEN_LIMIT` before apps can use `llm`.

For env details, read the repo's `docs/service-config.md`.

## App-Only Deploy

For a single app:

```bash
cd apps/my-tool
npm run build
cd ../..
railcode deploy my-tool --access workspace
```

`railcode deploy` syncs `tools/my-tool/` to the server's tool root. It does not rebuild the app and does not restart backend services.

Access examples:

```bash
railcode login --api-url https://auth.tools.example.com
railcode deploy my-tool --access private
railcode deploy my-tool --access workspace
railcode deploy my-tool --access restricted --access-users teammate@example.com
railcode deploy my-tool --access restricted --access-domains example.com
```

Use `--target`, `--remote-root`, and `--dry-run` when needed:

```bash
railcode deploy my-tool --target ubuntu@tools.example.com --remote-root /var/www/tools
railcode deploy my-tool --dry-run
```

If `--target` is omitted, `railcode deploy` reads `deploy/config.env`:

```text
HOST=tools.example.com
SSH_USER=ubuntu
SSH_OPTS="-i ~/.ssh/lightsail.pem"
```

## Platform Repo Deploy Script

Use `./deploy/deploy` for platform-level deployment from the Railcode source repo:

```bash
cp deploy/config.env.example deploy/config.env
./deploy/deploy tools
./deploy/deploy backend
./deploy/deploy caddy
./deploy/deploy all
./deploy/deploy logs
./deploy/deploy ssh
```

Commands:

- `tools`: rsync all `tools/` folders to `/var/www/tools`.
- `backend` or `api`: build `frontend`, build `sdk`, rsync `backend/` to `/opt/tools-api`, run `uv sync`, restart `tools-api`.
- `caddy`: sync `Caddyfile` and reload Caddy.
- `all`: backend, caddy, then tools.
- `logs`: tail `sudo journalctl -u tools-api -f`.

Do not use a backend deploy for an app-only change unless the app change depends on SDK/backend behavior that has also changed.

## Local Docker Flow

For a full local browser flow:

```bash
docker compose up --build
```

Open:

```text
http://notes.127.0.0.1.nip.io:8080
http://guestbook.127.0.0.1.nip.io:8080
http://auth.127.0.0.1.nip.io:8080
```

Local Compose uses `admin@example.com/admin`, SQLite at `/data/platform.db`, local files at `/data/files`, and a `railcode-data` Docker volume. Wipe local data with:

```bash
docker compose down -v
```

## Post-Deploy Verification

After an app deploy:

```bash
railcode status --tool my-tool --api-url https://auth.tools.example.com
railcode whoami --tool my-tool --api-url https://auth.tools.example.com
```

Open:

```text
https://my-tool.<BASE_DOMAIN>/
```

Check:

- Auth redirects to `auth.<BASE_DOMAIN>` when not logged in.
- App loads `/_api/sdk.js` without CORS or mixed-content errors.
- `me()` returns the expected user and tool.
- KV/files reads and writes succeed.
- SQL/LLM features show configured, empty, or disabled states cleanly.
- Access policy matches the requested mode.

For app data inspection:

```bash
railcode store list <collection> --tool my-tool --api-url https://auth.tools.example.com
railcode files list --tool my-tool --api-url https://auth.tools.example.com
```
