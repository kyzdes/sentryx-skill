# Connecting to the SentryX MCP

SentryX MCP is **remote, over Streamable HTTP** at `/api/mcp`, authed by an org
Bearer token. One token scopes every call to one org. Connect once; the server
is then available to any agent session.

---

## 1. Mint an org token (web UI → /settings)

1. Open the web UI: **<https://sentryx.moone.dev/settings>** (the human signs in).
2. Go to **API keys** and create a new org token. Give it a name (e.g.
   `claude-code`). Copy it **now** — it's shown once; the server stores only its
   `sha256` (in the `api_keys` table), so it can't be re-displayed later.
3. The token carries scopes: **`project:create`**, **`define:write`**, **`read`**.
   - `read` — `search_issues`, `get_issue`, `get_trace`, `get_funnel`,
     `prepare_fix_bundle`, `instrument_hint`, `get_started`, `whoami`.
   - `project:create` — `create_project`.
   - `define:write` — `define_event`, `define_funnel`, `define_feature`.
   For a fix-bugs-only agent, a `read`-scoped token is enough.

## 2. Register the server

```
claude mcp add --transport http sentryx https://mcp.moone.dev/api/mcp \
  --header "Authorization: Bearer <token>"
```

Endpoints:

| Env | MCP URL |
|-----|---------|
| Prod | `https://mcp.moone.dev/api/mcp` |
| Prod (fallback) | `https://ingest.moone.dev/api/mcp` |
| Local | `http://localhost:8080/api/mcp` |

> `sentryx.moone.dev` is the **Next.js web UI**, *not* the MCP. The Go app that
> serves the MCP is reachable at `ingest.moone.dev`; `mcp.moone.dev` is its
> friendly alias. If `mcp.moone.dev` doesn't resolve in your environment, use
> the `ingest.moone.dev` fallback.

## 3. Confirm it worked

```
whoami        # -> { org, scopes }  ← the token resolved
get_started   # orients you: projects + what's instrumented + next step
```

If `whoami` returns your org and scopes, you're connected. Hand off to the
instrument flow or the fix-bugs flow (see `workflow.md`).

---

## Troubleshooting

### 401 / "unauthorized" / "SENTRYX_TOKEN: …"
The Bearer is **missing or wrong**. Check:
- The header is exactly `Authorization: Bearer <token>` (note the `Bearer ` prefix and the space).
- You pasted the **full** token, no surrounding quotes/whitespace, not truncated.
- The token hasn't been revoked/rotated in `/settings`. Mint a fresh one and re-run `claude mcp add` (remove the old server first: `claude mcp remove sentryx`).
- You're hitting `/api/mcp` (not the web UI host). Try the `ingest.moone.dev` fallback.

Confirm a fix with `whoami` — it's the cheapest round-trip that proves auth.

### "api token lacks required scope `project:create`" (or `define:write`)
Your token is `read`-only (or missing that scope). Mint a new token in
`/settings` with the write scopes and re-register the server.

### "no project yet — create one with create_project"
The org has no project. Run `create_project(slug: "…")` (needs `project:create`),
then proceed.

### Rotate / revoke a token
In **<https://sentryx.moone.dev/settings>** → API keys: revoke the old key and
create a new one. Tokens are independent, so you can rotate without downtime —
add the new server, confirm with `whoami`, then revoke the old key. Because only
the `sha256` is stored, a leaked token is mitigated solely by revoking it.
