# SpaceDock for Claude Code

Deploy a directory, get a live HTTPS URL in ~1s — and get the whole outcome back, errors
first, so an agent can fix and redeploy without a human pasting anything between steps.

This repo is the Claude Code plugin: the `spacedock` skill plus the MCP server wiring. The
server itself is [`spacedock-mcp`](https://www.npmjs.com/package/spacedock-mcp) on npm, which
the plugin runs via `npx` — there is nothing to build.

## Install

```
/plugin marketplace add ioanSL/spacedock-plugin
```

```
/plugin install spacedock@spacedock
```

Then export your key and restart the session — MCP servers are read at startup:

```bash
export PLATFORM_API_KEY=sk_your_key
```

Mint the key at [spacesagents.com](https://spacesagents.com) under **Account**. Point
`PLATFORM_API_URL` at your own box if you self-host; it defaults to
`https://api.spacesagents.com`.

**The key is a bearer token with no scopes**: whoever holds it can deploy, read logs, fork
and **destroy** every app in the account. Mint one per agent so a revoke costs you one agent.

## Check it

Ask the agent to list your apps. It should call `list_apps` and come back with JSON — an
empty `{"apps": []}` on a fresh account. Then hand it a directory and ask it to deploy.

## The tools

| tool | arguments | returns |
|---|---|---|
| `deploy` | `dir`, `app?` | `url`, `status`, `startup_ms`, `entry_cmd`, `errors`, `log_tail` |
| `logs` | `app`, `since?`, `level?`, `limit?` | `count` and `lines` of `{ts, stream, msg}`, newest last |
| `screenshot` | `app`, `path?`, `viewport?` | the PNG inline, plus `console_errors` and `failed_requests` |
| `sql` | `app`, `sql?`, `file?` | `rows` + `count`, or `output` for a migration. Omit `sql` for the schema |
| `fork` | `app` | a full deploy result for `<app>--fN`, code + data + secrets copied |
| `promote` | `fork_app` | `promoted`, `urls`, `rolled_back_to` |
| `set_secret` | `app`, `key`, `value` | write-only; takes effect on the **next** deploy |
| `list_apps` | — | every app with `url`, `state`, `tier`, `last_deploy` |
| `destroy` | `app` | not reversible |

`status` is `healthy`, `crashed` or `timeout`, and only `healthy` means the URL serves. A
crashed deploy comes back **as a crash** and traffic is never moved onto it, so the previous
version keeps serving — which is what makes the loop safe to run unattended.

## Without the plugin

Any MCP client can run the server directly — Kimi Code CLI reads the same shape at
`~/.kimi/mcp.json`, but note that Kimi has no documented `${VAR}` expansion, so the key sits
there in plaintext:

```json
{
  "mcpServers": {
    "spacedock": {
      "command": "npx",
      "args": ["-y", "spacedock-mcp"],
      "env": {
        "PLATFORM_API_URL": "https://api.spacesagents.com",
        "PLATFORM_API_KEY": "sk_your_key"
      }
    }
  }
}
```

The skill in `skills/spacedock/` is plain markdown — paste it into a system prompt or an
`AGENTS.md` for a client that does not read Claude Code skills.

## Troubleshooting

| symptom | cause |
|---|---|
| server never connects, no tools | `PLATFORM_API_KEY` unset, or the session was not restarted |
| `unknown api key` | key revoked, or an unexpanded `${PLATFORM_API_KEY}` reaching the server |
| tools appear but every call 401s | key from a different box than `PLATFORM_API_URL` |
| `fetch failed` | `PLATFORM_API_URL` unreachable, or a self-signed certificate on a self-hosted box (add `"NODE_TLS_REJECT_UNAUTHORIZED": "0"`) |
| `bundle is N MB, cap is 50MB` | client-side cap. Add large files to `.gitignore` — bundling is gitignore-aware inside a repo |
| `deploy` says `healthy`, the URL 502s with `Connect` | the app bound `127.0.0.1`. The probe runs inside the container; traffic does not. Bind `0.0.0.0` |

## License

MIT
