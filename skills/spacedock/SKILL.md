---
name: spacedock
description: Deploy a directory to a live HTTPS URL on SpaceDock and read back what happened — startup errors, logs, screenshots, SQL over its database, fork-with-state. Use when deploying or redeploying an app, diagnosing why a deployed app is broken or 502ing, reading its runtime logs, screenshotting it, querying or migrating its SQLite database, forking it to try a migration, promoting a fork, setting a secret, or destroying an app. Also use when the user says "deploy this", "ship it", "put this on a URL", or mentions SpaceDock or the spacedock MCP tools.
---

# SpaceDock

Deploy a directory, get a live HTTPS URL in ~1s. Built so you can close the loop without a
human pasting anything: every call returns the whole outcome, errors first.

The `spacedock` MCP server ships with this plugin. It needs `PLATFORM_API_KEY` in the
environment — if every call returns `invalid api key`, that is the missing piece and the user
mints one at spacesagents.com under **Account**.

## The loop

```
deploy → read status → (crashed? read errors + log_tail) → fix → deploy again
```

`deploy` is synchronous and returns the *result*, not an acceptance. **There is nothing to
poll and no dashboard to check.** A crashed app comes back as `status: "crashed"` with the
errors in the response, and traffic is never moved onto it — so a failed deploy leaves the
previous version serving.

```json
{
  "app": "fibonacci",
  "url": "https://fibonacci-287e13d3eb.example.dev",
  "status": "healthy",
  "startup_ms": 343,
  "entry_cmd": "bun run index.ts",
  "errors": [],
  "root_check": { "ok": true, "status": 200 },
  "bundle": { "via": "git", "files": 12, "top_level": ["index.ts", "public"], "excluded": [] },
  "log_tail": ["fibonacci listening on :8080"]
}
```

`status` is one of `healthy`, `crashed`, `timeout`. Only `healthy` means the URL works.

**`healthy` means something actually requested the app.** Before traffic moves, the platform
issues a real `GET /` from outside the container and fails the deploy on a transport error or
a 5xx, with the status line and a body excerpt in `errors`. A 4xx passes and is reported —
an API-only app answering 404 on `/` is not broken. So `status: "healthy"` plus a URL that
errors is no longer a state you have to go looking for.

**`bundle` is what got uploaded.** `top_level` and `files` are what arrived; `excluded[]` is
what did not, with a reason. A non-empty `excluded[]` is a warning worth reading before you
debug anything else — if the thing you meant to deploy is in that list, that is your bug.

**Five apps per account on the free plan.** `app limit reached: 5 of 5` means the account is
full, not that anything is broken: `deploy` to a *new* name and `fork` both refuse there,
while redeploying an app that already exists never does. Retrying is pointless — `destroy`
something first, and the slot is usable immediately. `list_apps` is how you see what is
taking the five.

## Before the first deploy

Three things decide whether it comes up. Get them right and you rarely need a second attempt.

**1. Bind `0.0.0.0`, never `127.0.0.1`.** Traffic arrives from the proxy on the container's
bridge address, so an app listening only on loopback is unreachable however healthy it looks
from inside. This now fails the deploy rather than lying about it — the `GET /` runs from
outside the container, and a loopback-only app refuses it — but the fix is still yours. Bun's
`port` and Node's `server.listen(port)` bind all interfaces by default; the trap is writing
the host explicitly.

**2. Listen on `process.env.PORT`.** It is always `8080`, but read the variable.

**3. Have an entry point the runner can find**, in this order:

- `scripts.start` in `package.json` → runs `bun run start`
- otherwise the first of `index.ts`, `index.tsx`, `index.js`, `server.ts`, `server.js`,
  `main.ts`, `app.ts`, `src/index.ts`, `src/index.js`, `src/server.ts`
- otherwise an `index.html` at the root → served as a static site (below)

No match is an error naming all three. A `package.json` triggers `bun install` against a
shared content-addressed cache; **no `package.json` skips install entirely**, which is the
fastest path for a single-file app.

## Static sites

**Do not write a file server.** A bundle whose root has an `index.html` and no entry point is
served statically: directory index, MIME types, and an SPA fallback for extensionless paths
(a missing `.js` gets a 404 rather than your HTML with a 200). Deploy the build output
directly — `deploy("./dist")` — and nothing else is needed.

There is **no build step on the platform**: `bun install` runs, `scripts.build` does not. Run
the build locally and deploy its output. Apps have 256MB and half a vCPU, which is under what
a Vite build wants, so this is a deliberate boundary rather than a missing feature.

Runtime is Bun (TypeScript/JavaScript) — no Python, Ruby, or JVM. A prebuilt static binary
works via `{"scripts": {"start": "./server"}}`, which is a documented accident rather than a
contract — it holds only for a static `linux/amd64` binary and nothing keeps it working.

## When something is wrong

Two different situations, and the second one is the one people get wrong:

**The deploy came back `crashed` or `timeout`.** The response already has it: read `errors`,
then `log_tail`. You do not need `logs` for this.

**The deploy came back `healthy` but the app behaves wrongly at its URL — call `logs`
first.** Not a screenshot, not a guess at the cause, not another deploy: `logs`. This is the
case where the answer is already recorded and nothing has shown it to you yet. A single line
like `ENOENT: no such file or directory, open '/app/src/dist/index.html'` names the whole
problem, and it is sitting in `logs` from the first bad request.

The general rule underneath it: **a status is a claim, not evidence.** The moment what you
observe contradicts what a deploy reported, stop reasoning from the status and read the logs.

The runner restarts a dying process 3 times with backoff before giving up. After that the URL
serves a 502 whose body carries the same error lines, so a user hitting the URL sees what you
would.

`logs(app, since?, level?, limit?)` takes `level: "error"` to get only stderr, and an RFC3339
`since` to avoid re-reading what you have already seen. Lines come back oldest first, and
within one stack trace they are in the order the app printed them.

## Screenshots

`screenshot(app, path?, viewport?)` returns the PNG plus browser console errors and failed
network requests — the two things a screenshot alone would not tell you. `path` saves a copy
to disk; `viewport` is `WIDTHxHEIGHT`, default `1280x800`.

**It always captures `/`.** There is no way to screenshot a subpath, so you cannot see a
result page or a route behind a form. To verify anything else, request it over HTTP and check
the response. Screenshots are serialized platform-wide (one at a time) and take ~2–3s.

## State, sleep and fork

App data belongs in `/data` — a bind mount that survives container stop, so state outlives
sleep and redeploys. `/app/src` holds the code.

Apps sleep after 15 minutes idle and wake on the next request, holding it while they start
(~1s, because wake replays the current deploy and skips fetch and install). **Sleeping is
normal and invisible; do not "fix" it.** An app that `list_apps` reports as `sleeping` is
idle, not broken.

`fork(app)` clones an app *including its SQLite state* onto its own URL, using the online
backup API — so it works on a running app with a live writer. This is how you test a
migration:

```
fork(app) → deploy the migration to the fork → verify → promote(fork_app)
```

`promote` points the parent's URL at the fork with a one-row swap and parks the old container
stopped, so it is reversible. The forked app keeps its own URL too.

## SQL

`sql(app, sql?, file?)` runs statements against the app's SQLite database in `/data`. Omit
`sql` and you get the schema — table names and their `CREATE` statements.

**It reads the file on the host side of the bind mount, so the app does not have to be
running.** A sleeping app is not woken and a crashed app still answers — which is exactly when
the data is most worth reading. After a failed deploy, `sql` still works.

Writes and migrations are allowed, several statements in one call are fine, and **there is no
undo** — the only rollback is an operator restoring a snapshot. Fork first (above) when the
migration is the thing you are unsure about.

Reading the response:

- A result set is `rows` plus a `count`; zero rows is an empty array, not an error.
- **A migration comes back as `output` text, not `rows`.** DDL and inserts print nothing at
  all, and two selects print two arrays with no document around them, so anything but one
  clean result set arrives as a string. A missing `rows` is not a failure.
- Capped at 256KB with `truncated: true`, and there is no pagination or cursor — put the
  `limit` in the statement instead of expecting one.
- BLOBs arrive as escape sequences: fine for text and numbers, useless for stored bytes.
- Only tables are listed in the schema. Views, indexes and triggers need a
  `select … from sqlite_master` of your own.

Which database, when an app keeps more than one: the **schema** call picks the first and
returns the full list as `files`, so `sql(app)` then `sql(app, sql, file)` is the way in. A
*statement* with several databases and no `file` is refused rather than guessed at —
`this app has several databases — pass one as 'file': …`.

Two failures to avoid rather than debug:

- **The extension is load-bearing** — `.db`, `.sqlite` or `.sqlite3`. Under any other name
  `sql` will not find the database, and neither will snapshots, `fork` or replication.
- **It will not create one.** An app with no database is an error, not an empty result:
  `this app has no database yet — create one at $DATA_DIR/app.db`. Creating it is the app's
  job, at startup.

## Secrets

`set_secret(app, key, value)` is **write-only and takes effect on the next deploy** — nothing
reads a secret back out, by design. Deploy again after setting one, or the app will not see
it. Names are listable; values never leave the box.

Pass `env` instead of `key`/`value` to set a whole `.env` in one call:
`set_secret(app, env={"STRIPE_KEY": "...", "DATABASE_URL": "..."})`. That is the form to
use when you have just read a config file — one call, not one per line.

**Never deploy a `.env` file.** `deploy` drops them from the bundle, along with `*.pem`,
`*.key` and `id_rsa*`, so an app that reads its config from a checked-in `.env` will come
up with none of it. Read the file, `set_secret` it, deploy. Secrets set this way are
encrypted at rest and arrive as ordinary environment variables, which is what the app
already expects.

A secret whose name collides with one the runner sets — `PORT`, `HOST`, `DATA_DIR`,
`NODE_ENV` — overrides it. `PORT` is the one that bites: the app then listens somewhere the
health probe is not, and the deploy fails as unreachable rather than as misconfigured.

## Access control — tell the user this

An app's URL is currently its *only* access control. The subdomain suffix is unguessable
(40 bits) but there is no password and no login: anyone with the link has the app. Do not
deploy anything sensitive and do not describe a deployed app as private.

There is also no sharing model — apps belong to one account, and sharing means sharing the
URL.

## Limits and gaps

| | |
|---|---|
| Bundle | 50MB via MCP (platform accepts 64MB). Inside a repo `git ls-files` decides, so anything gitignored is left out — **including your build output, if `.gitignore` lists it**. Outside a repo a default list drops `node_modules`, `.git`, `.env*`, `*.pem`, `*.key`, `id_rsa*`. Either way the response's `bundle.excluded[]` names what went missing |
| Memory / CPU | 256MB, 0.5 vCPU per app |
| WebSockets | **not proxied** — an app needing them will not work |
| Screenshot | root path only |
| `destroy` | deletes the app, its container, its data and its URL. **Not reversible** — confirm with the user first |

## Naming

`deploy(dir, app?)` defaults the name to the directory's basename, lowercased with anything
outside `[a-z0-9-]` collapsed to `-`. Deploying the same name again updates that app in
place; a new name creates a new app with a new URL. Pass `app` explicitly when the directory
is called something like `tmp` or `src`.
