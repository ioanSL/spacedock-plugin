# Secrets and tags

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

## Tags

`set_tags(app, tags)` sets the labels the console groups and filters the app list by. Nothing
on the box reads them — not the reaper, not the router, not the quota — so they are for
whoever is looking at that list, which is the user rather than you.

**It replaces the whole set.** `set_tags(app, ["prod"])` on an app already tagged `api` leaves
it tagged `prod` and nothing else. To add one, read `list_apps` first — every app it returns
carries its `tags` — and send the union. `set_tags(app, [])` clears them.

Names are lowercased and slugified on the way in, so `Client X` is stored as `client-x`: the
value that comes back is not always the value you sent, and it is the one to repeat to the
user. 8 per app, 24 characters each — past either the call fails rather than quietly dropping
or truncating.
