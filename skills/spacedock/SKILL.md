---
name: spacedock
description: Deploy a directory to a live HTTPS URL on SpaceDock and read back what happened — startup errors, logs, screenshots, SQL over its database, fork-with-state. Use when deploying or redeploying an app, diagnosing why a deployed app is broken or 502ing, reading its runtime logs, screenshotting it, querying or migrating its database (its own SQLite, or an external Postgres), forking it to try a migration, promoting a fork, setting a secret, or destroying an app. Also covers what runtimes are supported — Bun/TypeScript and a compiled Rust or Go binary. Also use when the user says "deploy this", "ship it", "put this on a URL", or mentions SpaceDock or the spacedock MCP tools.
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

## Compiled binaries — Rust and Go

**A compiled binary runs, and it is not a special case**: it comes through `scripts.start`,
the same first rule as everything else. There is no interpreter involved, so there is nothing
for the platform to be missing. Verified against the box for both a Rust and a Go server, and
a `tonic` gRPC service is what the platform's h2c upstream leg was built for.

Three things have to line up, and a miss on any of them is a `crashed` deploy with the reason
in `errors` rather than a silent fallback:

- **Built for `linux/x86-64`.** Nothing compiles on the platform. The image is glibc
  (`oven/bun:1`), so either match that or target musl — from a mac,
  `cargo zigbuild --release --target x86_64-unknown-linux-musl`.
- **A `package.json` naming it**, `{"scripts": {"start": "./server"}}`. `detect_entry` reads
  that file and nothing else, so it is the only door a binary comes through. It also triggers
  `bun install`, which is a no-op with no dependencies but not free.
- **The binary actually in the bundle** — the one people miss. Bundling is gitignore-aware
  inside a repo and `cargo new` writes a `.gitignore` containing `/target`, so pointing
  `deploy` at a project root ships the source and no binary. Deploy a directory holding the
  binary and its `package.json`, and read `bundle.top_level` back to confirm.

**Strip it.** `strip = true` under `[profile.release]` for Cargo, `-ldflags="-s -w"` for Go —
measured on this platform, the same server is 227KB stripped from Rust against 4.6MB from a
default `go build`, which keeps its symbols. Harmless against a 50MB cap right up until
something gets vendored.

**gRPC works.** The upstream hop is h2c when the request is `application/grpc`, which is what
a tonic server needs and has no HTTP/1.1 listener to fall back to. `application/grpc-web` is
HTTP/1.1-native and was always fine.

What is genuinely absent is **interpreters** — no Python, Ruby, or JVM. Bun
(TypeScript/JavaScript) and a binary you compiled yourself are the two paths.

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

**What a fork clones is the SQLite state in `/data`.** It also copies the app's secrets
verbatim, so an app whose data lives in an external Postgres gets a fork pointing at the very
same database — see the Postgres section below before testing a migration that way.

## SQL

`sql(app, sql?, file?, create?)` runs statements against the app's database. Omit `sql` and
you get the schema — table names and their `CREATE` statements — plus every database the app
has, listed as `files`.

**Two backends answer here and the reply looks the same either way:** the SQLite file the app
keeps in `/data`, or an external Postgres when the app's secrets hold a connection string.
They share one candidate list, so `file` is how you choose and the schema call is how you see
what there is to choose from.

**SQLite is read on the host side of the bind mount, so the app does not have to be running.**
A sleeping app is not woken and a crashed app still answers — which is exactly when the data
is most worth reading. After a failed deploy, `sql` still works.

Writes and migrations are allowed and several statements in one call are fine. **There is no
undo**: for SQLite the only rollback is an operator restoring a snapshot, and for an external
Postgres there is not even that. Fork first (above) when the migration is the thing you are
unsure about — but read the Postgres section before trusting that on an external database.

Reading the response:

- A result set is `rows` plus a `count`; zero rows is an empty array, not an error.
- **A migration comes back as `output` text, not `rows`.** DDL and inserts print nothing at
  all, and two selects print two arrays with no document around them, so anything but one
  clean result set arrives as a string. A missing `rows` is not a failure.
- Capped at 256KB with `truncated: true`, and there is no pagination or cursor — put the
  `limit` in the statement instead of expecting one. Postgres is capped at 1,000 rows as well.
- BLOBs arrive as escape sequences: fine for text and numbers, useless for stored bytes.
- SQLite lists only tables in the schema — views, indexes and triggers need a
  `select … from sqlite_master` of your own. Postgres lists every non-system schema, and
  qualifies any name that is not in `public`.

**Which database, when the app has more than one.** The schema call picks the first and
returns the full list as `files`, so `sql(app)` then `sql(app, sql, file)` is the way in. A
*statement* with several databases and no `file` is refused rather than guessed at —
`this app has several databases — pass one as 'file': …`.

**The extension is load-bearing** — `.db`, `.sqlite` or `.sqlite3`. Under any other name `sql`
will not find the database, and neither will snapshots, `fork` or replication.

**It will not create a database it was not asked to create.** An unknown `file` is
`this app has no database named '…'`, and an app with none at all is `this app has no database
yet — create one at $DATA_DIR/app.db`. Normally that is the app's job at startup, and the
refusal is deliberate: the alternative makes a typo in `file` look like success.

`create: true` **together with an explicit `file`** is the way past it, for one real ordering
problem — the database is made by the app's own code, so until the app has booted once there
is nothing to migrate against:

```
sql(app, "create table todos (id integer primary key, title text)", file="app.db", create=True)
```

It is a flag and never an inference: `create` with no `file` is refused rather than named for
you, and the name still has to be usable — letters, digits, `.`, `_` and `-`, ending in one of
the three extensions above. Ordinary reads and writes need none of this.

## An external Postgres

Most backend apps keep their data somewhere else, and those answer here too. When the app's
secrets hold a Postgres URL, that database joins the same list as the files on disk, **named
after the secret holding it** — so you pass `DATABASE_URL` as `file`, never a hostname.

`DATABASE_URL` is preferred and any other key works (`POSTGRES_URL`, `PG_URL`, a one-off
name); the key is what appears in `files`. Postgres only — a `DATABASE_URL` holding a
`mysql://` URL is not offered at all, and neither is libSQL.

**It has to be reachable from the internet over TLS.** Loopback, private, CGNAT and
link-local addresses are refused, unix-socket connection strings are refused, and `sslmode`
is forced up to at least `require`. A managed database with a public endpoint (Neon, Supabase,
RDS) works; one on localhost or behind a private VPC is refused, and that is a guard rather
than a bug to report.

Three things differ from the SQLite side, and the second is the one that costs data:

- **There is no undo of any kind** — no snapshot, no rollback, no operator restore. The undo
  for a migration against somebody's Neon is their provider's.
- **`fork` does not protect it.** A fork copies the app's secrets verbatim, so it inherits the
  same `DATABASE_URL` and points at the *same* database. The fork-then-migrate recipe guards
  SQLite state and does nothing here: a migration you "test" on the fork has already run on
  production. Use the provider's own branching, or point the fork at a second database with
  `set_secret` before deploying it.
- **A failed migration has not half-run.** A write, DDL or migration cannot sit inside the
  `from` clause the row cap uses, so it fails to *plan* and is retried by a plainer path
  before anything executes — nothing is applied twice.

One collision worth knowing: if a secret is somehow named `app.db`, the file on disk wins and
the external database is not offered at all. Rename the secret.

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
| gRPC | **works**, h2c to the container. Bidirectional streaming included; the gap is WebSockets, not HTTP/2 |
| Screenshot | root path only |
| `destroy` | deletes the app, its container, its data and its URL. **Not reversible** — confirm with the user first |

## Naming

`deploy(dir, app?)` defaults the name to the directory's basename, lowercased with anything
outside `[a-z0-9-]` collapsed to `-`. Deploying the same name again updates that app in
place; a new name creates a new app with a new URL. Pass `app` explicitly when the directory
is called something like `tmp` or `src`.
