# Compiled binaries and gRPC

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
