# CLI Workflow

Use this reference when an agent needs exact Railcode CLI behavior, command options, local dev behavior, or API utilities.

## Install Or Link The CLI

In a Railcode source checkout:

```bash
cd cli
npm install
npm run build
npm link
```

The CLI stores config under `${RAILCODE_HOME:-~/.railcode}/config.json`. The default API URL is `http://auth.127.0.0.1.nip.io:8080`. Override per command with `--api-url` or for the process with `RAILCODE_API_URL`.

For non-interactive login, pass `--username`, `--password`, and `--api-url`, or set `RAILCODE_USERNAME`, `RAILCODE_PASSWORD`, and `RAILCODE_API_URL`.

## Create An App

Run from the workspace root:

```bash
railcode init my-tool
```

Behavior:

- Validate tool names against `^[a-z0-9][a-z0-9-]{0,62}$`.
- Create `apps/<tool>` from `cli/railcode-templates/railcode-react`.
- Create `tools/<tool>` for build output.
- Replace `__RAILCODE_APP__` and `__RAILCODE_TITLE__` placeholders.
- Refuse to overwrite existing source or build output unless `--force` is passed.

The starter uses React, Vite, Zustand, Tailwind, lucide-react, TypeScript, and exact package pins. Keep direct dependency versions exact unless there is a reason to upgrade.

The starter's `vite.config.ts` builds to `../../tools/<tool>` and serves the asset dev server on `127.0.0.1:5173`.

## Local Dev

Run from `apps/<tool>`:

```bash
railcode dev
```

Useful options:

```bash
railcode dev --verbose
railcode dev --reset
railcode dev --port 7332
railcode dev --asset-port 5174
railcode dev --command "npm run dev -- --host 127.0.0.1 --port 5173"
```

App detection order:

- `--app` option or positional app argument.
- `railcode.json` `app` or `tool`.
- App name inferred from an `apps/<tool>` or `tools/<tool>` path.
- The only app under `apps/` if exactly one exists.
- `notes` for legacy/demo workspaces when present.
- Current directory basename if it matches the tool-name regex.

Root and asset server detection:

- If `railcode.json` has `dev.root`, use it relative to the manifest.
- Otherwise infer from `apps/<tool>` or `tools/<tool>`.
- If the root has `package.json` with a `dev` script, run it.
- Prefer `pnpm`, then `yarn`, otherwise `npm`.
- Run `npm ci` when a package-lock exists and dependencies are missing; otherwise run the package manager's normal install.

Local API behavior:

- `/_api/sdk.js`: serve the bundled SDK from the CLI package or source checkout.
- `/_api/me`: return `{ user: "local-dev", tool, app }`.
- `/_api/tool-users`: return local mode with an empty complete roster.
- `/_api/kv/*`: store JSON in `~/.railcode/dev/<tool>/kv.json`.
- `/_api/files/*`: store files in `~/.railcode/dev/<tool>/files/` and metadata in `files.json`.
- `/_api/connections`: return `[]` unless a saved API token exists.
- Other `/_api/*`: forward to `<api-url>/v1/apps/<tool>/*` with bearer auth when logged in.

If the remote backend rejects forwarded local-dev calls with `401` or `403`, `connections()` becomes `[]`; other backend-backed calls return a `502` explaining that local identity/KV/files still work.

## Login And Status

```bash
railcode login --api-url https://auth.tools.example.com --username you@example.com
railcode status
railcode status --tool my-tool
railcode whoami --tool my-tool
railcode logout
```

`railcode login` logs in through `/login`, creates an API token through `/api-token`, and stores both session cookies and the bearer token locally.

Remote app API commands require a saved API token:

```bash
railcode store list notes --tool my-tool
railcode store get notes key --tool my-tool
railcode store put notes key '{"text":"hello"}' --tool my-tool
railcode store put notes key --file data.json --tool my-tool
railcode store delete notes key --tool my-tool

railcode files list --tool my-tool
railcode files upload logo logo.png --tool my-tool --content-type image/png
railcode files download logo logo.png --tool my-tool
railcode files delete logo --tool my-tool
```

File API names cannot contain `/`.

## Deploy A Tool With The CLI

Build first:

```bash
cd apps/my-tool
npm run build
cd ../..
```

Deploy:

```bash
railcode deploy my-tool
railcode deploy my-tool --target ubuntu@tools.example.com
railcode deploy my-tool --remote-root /var/www/tools
railcode deploy my-tool --dry-run
```

Target resolution:

- `--target` wins.
- Otherwise load `deploy/config.env` and use `SSH_USER@HOST`.
- `--remote-root` defaults to `TOOLS_ROOT` from `deploy/config.env`, then `/var/www/tools`.
- `SSH_OPTS` from `deploy/config.env` is passed to ssh/rsync.

The deploy command rsyncs:

- `tools/<tool>/` to `<target>:<remote-root>/<tool>/`
- `tools/` to `<target>:<remote-root>/` with `--all`

Use `railcode deploy <tool> --access private|workspace|restricted` to configure tool access after a successful sync. Restricted deploys accept comma-separated `--access-users` and `--access-domains`. Access configuration requires an admin login session from `railcode login`.
