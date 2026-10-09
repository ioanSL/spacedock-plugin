# Data: state, fork, SQL and an external Postgres

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
