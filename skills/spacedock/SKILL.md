---
name: spacedock
description: Deploy a directory to a live HTTPS URL on SpaceDock and read back what happened — startup errors, logs, screenshots, SQL over its database, fork-with-state. Use when deploying or redeploying an app, diagnosing why a deployed app is broken or 502ing, reading its runtime logs, screenshotting it, querying or migrating its database (its own SQLite, or an external Postgres), forking it to try a migration, promoting a fork, setting a secret, tagging an app so the app list can be filtered, or destroying an app. Also covers what runtimes are supported — Bun/TypeScript and a compiled Rust or Go binary. Also use when the user says "deploy this", "ship it", "put this on a URL", or mentions SpaceDock or the spacedock MCP tools.
---

# SpaceDock

The `spacedock` MCP server ships with this plugin and needs `PLATFORM_API_KEY` in the
environment. If every call returns `invalid api key`, that is the missing piece: the user mints
one in the console, https://spacedock-console-production.up.railway.app, under **Account**.

## The loop

```
deploy → read status → (crashed/timeout? read errors + log_tail) → fix → deploy again
```

`deploy` is synchronous and returns the *result*. **There is nothing to poll and no dashboard
to check.** `status` is `healthy`, `crashed` or `timeout`, and only `healthy` means the URL
works: before traffic moves, the platform sends a real `GET /` from outside the container and
fails the deploy on a transport error or a 5xx (a 4xx passes).

**A failed deploy takes the app down**: the old process stops before the new one starts, and the
URL fails until a deploy comes back `healthy`. To change an app people rely on, deploy the change
to a `fork` and `promote` it.

**`bundle.excluded[]`** names top-level entries that shipped nothing. If the thing you meant to
deploy is in it, that is your bug. Inside a git repo, gitignored files are left out at any depth,
build output included, and a file inside a shipped directory is not listed when it goes missing.

## Before the first deploy

1. **Bind `0.0.0.0`, never `127.0.0.1`.** Traffic arrives on the container's bridge address.
   Bun's `port` and Node's `server.listen(port)` bind every interface by default; the trap is
   writing the host explicitly.
2. **Listen on `process.env.PORT`.** It is always `8080`, but read the variable.
3. **An entry point the runner can find**, in this order: `scripts.start` in `package.json`;
   else the first of `index.ts`, `index.tsx`, `index.js`, `server.ts`, `server.js`, `main.ts`,
   `app.ts`, `src/index.ts`, `src/index.js`, `src/server.ts`; else a root `index.html`, served
   as a static site. No `package.json` skips `bun install` entirely, which is the fastest path.

**Static sites: do not write a file server.** A root `index.html` with no entry point is served
with MIME types and an SPA fallback.

**App data belongs in `/data`** (`process.env.DATA_DIR`). It survives redeploys and sleep, so
there is no need to test that. `bun:sqlite` is built in; name the file `.db`, `.sqlite` or
`.sqlite3`, or snapshots, `fork` and `sql` will not see it. Apps sleep after 15 idle minutes and
wake on the next request in ~1s. That is normal for an app that answers requests, but sleep also
stops `setInterval`, cron and polling loops until a request arrives: tell the user.

**Never deploy a `.env` file**: the bundle drops it, and every `*.pem`, `*.key` and `id_rsa*`
too. `set_secret` needs the app to exist, so on a new app: deploy, `set_secret` (one call with
`env`), deploy again.

The name defaults to the directory's basename, slugified, and the same name updates that app in
place. **Pass `app` when the directory is generic** (`dist`, `build`, `browser`, `src`, `tmp`),
or every such project lands on one app.

## When something is wrong

**`healthy` but wrong at its URL: call `logs` first**, before a screenshot, a guess or another
deploy. The line that names the problem is already recorded there. A status is a claim, not
evidence.

`screenshot` always captures `/`, with console errors and failed requests; check any other path
over HTTP.

## Tell the user, and confirm first

- **The URL is the app's only access control**: an unguessable suffix, no password, no login.
  Do not deploy anything sensitive, and never call a deployed app private.
- **`destroy` deletes the app, its data and its URL, irreversibly.** Confirm with the user.

## Limits

256MB and 0.5 vCPU per app · **WebSockets are not proxied** · Bun (TypeScript/JavaScript) or a
binary you compiled; no Python, Ruby or JVM.

## Read these when the task needs them

They sit next to this file.

- [`reference/binaries.md`](reference/binaries.md): **deploying Rust, Go or any compiled
  binary, or a gRPC server.** It must be a linux/x86-64 build named by `package.json`, and the
  binary must be in the bundle.
- [`reference/data.md`](reference/data.md): **before `fork`, `promote`, `sql`, a migration,
  or anything touching an external Postgres.** A fork copies secrets, so it points at the
  *same* external database: forking does not protect it.
- [`reference/secrets-and-tags.md`](reference/secrets-and-tags.md): **before `set_secret` or
  `set_tags`.** A secret named `PORT` breaks the deploy, and `set_tags` replaces the whole set.
