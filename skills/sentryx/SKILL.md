---
name: sentryx
description: Instrument an app with SentryX — a self-hosted Sentry-analog that unifies error tracking, distributed tracing, and product analytics/funnels, correlated by trace_id — exposed over a remote MCP server named `sentryx`. Use whenever the sentryx MCP tools are present (get_started, whoami, create_project, instrument_hint, define_event/funnel/feature, get_funnel, search_issues, get_issue, get_trace, prepare_fix_bundle) or the user mentions SentryX, "instrument my app", "error tracking", "distributed tracing", "define a funnel", "product analytics", "trace debugging", "set up Sentry / SentryX", "OTLP", or "fix this bug with SentryX". Teaches the connect-once flow, the instrument flow (detect stack → 5 product questions → create_project → snippets → define_* → verify), and the fix-bugs flow (search_issues → prepare_fix_bundle). NOT for hosted Sentry.io, Datadog, or PostHog — those are different tools.
metadata:
  short-description: Instrument an app with SentryX (errors + traces + funnels) over MCP
  version: "1.1"
---

# SentryX

SentryX is a **self-hosted observability platform** — error tracking + distributed tracing + product analytics/funnels, all **correlated by `trace_id`** — exposed over a remote MCP server named `sentryx`. This skill tells you how to take a raw app from zero to fully instrumented, and how to use the read tools to fix high-impact bugs without spelunking.

## The one rule that protects your context

**Don't read everything.** Call `get_started` first — it returns who you are, your org's projects, the tool catalogue, and the exact next step. Pull only what the task needs (`instrument_hint`, `get_funnel`, `prepare_fix_bundle`). Reading every issue or trace "to understand the app" is the failure mode this skill exists to prevent — it burns your context before you've instrumented anything or fixed a single bug.

## Connect (once)

SentryX MCP is **remote, over Streamable HTTP** at `/api/mcp`, authed by an org Bearer token. The human mints the token in the web UI; you register the server:

```
claude mcp add --transport http sentryx https://mcp.moone.dev/api/mcp \
  --header "Authorization: Bearer <token>"
```

- **Mint the token:** https://sentryx.moone.dev/settings → API keys (sha256-stored; scopes `project:create`, `define:write`, `read`).
- **Endpoints:** prod `https://mcp.moone.dev/api/mcp` (fallback `https://ingest.moone.dev/api/mcp`); local `http://localhost:8080/api/mcp`. Note: `sentryx.moone.dev` is the **web UI**, not the MCP.
- **Confirm:** call `whoami` (returns your org + scopes) and `get_started` (orients you). A `401` means a bad/missing Bearer — see `references/connect.md`.

## Onboard the user

When a user first connects (or asks to "set up SentryX"):

1. **Greet + orient** — run `get_started`; tell them what's wired (project? funnels? recent errors?).
2. **Get the token if missing** — point them at https://sentryx.moone.dev/settings to mint an org token, then run the connect command above. Confirm with `whoami`.
3. **Explain capabilities** — one platform for: errors (Sentry SDK), traces (OTLP), and product funnels (`track()`), correlated by `trace_id`. The web UI shows **/issues**, **/p/&lt;slug&gt;/funnels**, and traces.
4. **Offer the two flows** — "instrument my app" (below) or "fix a bug" (`search_issues` → `prepare_fix_bundle`).

## The instrument flow

The hero path: a raw repo → errors + traces + funnels live in the web. Follow it in order.

1. **Detect the stack** — inspect the repo: `package.json` (node/nextjs), `requirements.txt`/`pyproject.toml` (python), `go.mod` (go), an SPA entry (browser). That picks the `instrument_hint` stack.
2. **Ask the 5 product questions** (they fully parameterize analytics — keep it to these):
   1. **The ONE money/value action?** ("completed purchase") → the funnel's final step + primary success event.
   2. **The 3–5 steps that lead to it?** → the funnel `steps` (ordered event names).
   3. **Per user or per session?** → whether `track()` carries a stable `user_id` (cross-session) or a per-`session_id` distinct_id.
   4. **Which features want adoption?** → `define_feature` keys.
   5. **What conversion window is meaningful?** (1h / 24h / 7d) → funnel `window_seconds`.
3. **Provision** — `create_project(name, slug)` → `{project_id, dsn, otlp_http, otlp_grpc, track_url, is_new}`. Idempotent on slug; capture `dsn`, `otlp_*`, `track_url`.
4. **Get the snippets** — `instrument_hint(stack, dsn, track_url)` → `{errors_snippet, tracing_snippet, track_snippet, notes}`.
5. **Apply all three** to the code:
   - **Errors** — Sentry SDK init with the `dsn` (+ `release`, `environment`).
   - **Tracing** — point the OTel exporter at `otlp_http` / `otlp_grpc`.
   - **Product events** — drop in `track()` and call it at each funnel step (Q2) and each feature (Q4). **Every `track()` must carry `distinct_id` + `user_id` + `session_id`, and `trace_id` when in a request** — that backbone is what correlates a product event to the error/trace in the same flow. Copy-paste per stack: `references/example-instrumentation.md`.
6. **Register the definitions** — `define_event(name)` for each event you emit, `define_funnel(key, name, steps, window_seconds)`, `define_feature(key, event_names)`.
7. **Verify data flows** — trigger the flow (run the app / a smoke script), then `get_funnel(funnel_key)`. Monotonically decreasing step counts + a `biggest_drop` = instrumentation is correct. The project now shows in the web.

Full end-to-end recipe with example calls: `references/workflow.md`.

## The fix-bugs flow

Any agent with the org token gets the full read picture:

1. `search_issues(status?, level?, q?)` — list issues, newest activity first → pick an `id`.
2. `prepare_fix_bundle(issue_id)` — **the hero tool.** One call returns a ~250–600-token, fix-ready bundle: summary, exception, in-app frames + source context, breadcrumbs, the correlated trace (with bottleneck), impact (times_seen / affected_users), similar issues, and suspect commits. Fix from this; don't reconstruct it by hand.
3. Need more? `get_issue(issue_id)` for the latest-event summary, `get_trace(trace_id)` to drill into spans + correlated errors.
4. Prioritize with `get_funnel` — where users drop off tells you what to fix or build next.

### Close the loop: the assisted-autonomy fix flow

SentryX **records and audits** the fix; **you** do the actual work — read the bundle, write the patch, and open a **DRAFT** PR with `gh`. SentryX never patches code (there is no server-side patch service). The loop, with idempotency so a retried step audits exactly once:

1. `record_fix_attempt(issue_id, summary, idempotency_key)` — call this the moment you start from `prepare_fix_bundle`, **before** opening a PR. Sets the issue's `fix_status='investigating'` and logs an audit entry. Use a stable key like `fix-<issue_id>-<short-sha>` so retries don't double-log.
2. **Write the fix and open a DRAFT PR yourself** with the GitHub CLI — never push to a protected branch:
   ```
   git checkout -b fix/issue-<issue_id>
   git commit -am "Fix: <summary>"
   gh pr create --draft --title "Fix: <summary>" --body "Fixes issue #<issue_id> in SentryX. <root cause from prepare_fix_bundle>"
   ```
3. `link_pr(issue_id, pr_url, idempotency_key)` — call this **after** `gh pr create` returns the URL. Sets `fix_status='pr_open'` + stores `pr_url`. Use a stable key like `pr-<issue_id>-<pr_number>`.
4. **A human reviews and merges** the draft PR (the human, not you — you only opened a draft).
5. `resolve_issue(issue_id)` — call this **only after** the PR is merged/deployed. Sets the issue `status='resolved'`, `fix_status='fixed'`, and logs the resolve.

Use `list_fix_attempts(issue_id)` before starting to check whether a fix is already in flight (don't open a duplicate PR). Full step-by-step recipe: `references/workflow.md` (recipe 2).

## Tool reference

Every MCP tool — meta (`get_started`, `whoami`), write (`create_project`, `define_*`, `instrument_hint`), and read (`search_issues`, `get_issue`, `get_trace`, `get_funnel`, `prepare_fix_bundle`) — with inputs, outputs, and when to use: `references/tools.md`.

See also:
- `references/tools.md` — every MCP tool with I/O shapes and when to use.
- `references/workflow.md` — end-to-end recipes: onboard a project, fix a high-impact error, analyze a funnel drop-off.
- `references/connect.md` — token mint walkthrough + connect troubleshooting (401, whoami, rotate/revoke).
- `references/example-instrumentation.md` — copy-paste snippets per stack (Node/Next, Python, Go, browser): Sentry init, OTLP env, and the `track()` helper.
