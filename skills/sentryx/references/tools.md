# SentryX MCP tool reference

All tools on the `sentryx` MCP server, served **remotely over Streamable HTTP**
at `/api/mcp` and authed by an `Authorization: Bearer <org token>` header. Every
call is **scoped to your org**; `project_id` defaults to your org's project when
omitted. Write tools are **scope-gated** (`project:create`, `define:write`); read
tools need only a valid token. Results come back as JSON.

> **Start with `get_started`.** It returns who you are, your projects, and the
> next step — reach for the granular tools only when the task needs them.

---

## Meta & onboarding

| Tool | Inputs | Output / when to use |
|------|--------|----------------------|
| `get_started` | `project_id?` | **START HERE.** Orients you: your org, projects, what's instrumented (project? funnels? recent errors?), the tool catalogue, and the recommended next step. Run it first every session. |
| `whoami` | — | The authenticated actor `{ org, scopes }`. Use to confirm the Bearer token resolved (after `claude mcp add`) and to check which write scopes you hold. A `401`/auth error means the token is bad or missing — see `connect.md`. |

## Write — provision & define (scope-gated)

| Tool | Inputs | Output / when to use |
|------|--------|----------------------|
| `create_project` | `slug` (required, url-safe), `name?` | `{project_id, slug, dsn, otlp_http, otlp_grpc, track_url, is_new}`. **Idempotent on slug** (re-running returns the same DSN). Put `dsn` in the Sentry SDK, point the OTel exporter at `otlp_*`, send product events to `track_url`. Needs scope `project:create`. |
| `instrument_hint` | `stack` (`node\|nextjs\|python\|go\|browser`, required), `dsn?`, `track_url?` | `{stack, errors_snippet, tracing_snippet, track_snippet, notes}`. Copy-paste instrumentation for the detected stack: Sentry init, OTLP exporter env, and the `track()` helper. Read-only. Pass the `dsn`/`track_url` from `create_project` so the snippets are ready to paste. |
| `define_event` | `name` (required), `project_id?` | `{id, project_id}`. Register a product event name you emit via `track()`. Call once per distinct event. Needs scope `define:write`. |
| `define_funnel` | `key` (required), `steps` (ordered event names, 2–8, required), `name?`, `mode?` (`ordered`), `window_seconds?` (default 86400), `project_id?` | `{funnel_id, project_id, key, steps}`. Define a conversion funnel. Query it later with `get_funnel`. Needs scope `define:write`. |
| `define_feature` | `key` (required), `event_names?`, `project_id?` | `{id, project_id}`. Map a feature key to the events that constitute it (for adoption). Needs scope `define:write`. |

## Read — analytics

| Tool | Inputs | Output / when to use |
|------|--------|----------------------|
| `get_funnel` | `funnel_key` (required), `project_id?` | `{key, name, window_seconds, steps:[{name,count}], total_entered, biggest_drop:{from_step,to_step,lost,rate}}`. Computes a defined funnel via ClickHouse `windowFunnel`. Step counts are **monotonically non-increasing**. Use it to **verify instrumentation** (counts show up and decrease) and to **find drop-off** (prioritize what to fix/build). |

## Read — errors & traces

| Tool | Inputs | Output / when to use |
|------|--------|----------------------|
| `search_issues` | `project_id?`, `status?` (`unresolved\|resolved\|ignored`), `level?` (`error\|fatal\|warning\|info`), `q?` (title substring), `limit?` (default 20, max 100) | `{issues:[{id,title,culprit,level,status,times_seen,last_seen,group_hash}], count}`, newest activity first. Your entry point to triage — pick an `id`, then `prepare_fix_bundle`. |
| `get_issue` | `issue_id` (required), `project_id?` | `{issue:{...}, latest_event:{event_id,timestamp,trace_id,platform,environment,release,exception_type,exception_value}\|null}`. One issue plus a summary of its latest event. Use to grab the `trace_id` for `get_trace`. |
| `get_trace` | `trace_id` (required, 32-hex W3C id) | `{trace_id, spans:[{service_name,span_name,duration_ms,status_code,span_id,parent_span_id}], errors:[{event_id,exception_type,exception_value,issue_id}]}`. Drill into a distributed trace: its spans and any **correlated errors**. Use to understand a slow/failing request end-to-end. |
| `prepare_fix_bundle` | `issue_id` (required), `project_id?` | **THE HERO TOOL.** One call → a ~250–600-token, fix-ready bundle: `{summary, issue, exception{type,value}, in_app_frames[{function,filename,lineno,context_line,in_app}], breadcrumbs[{ts,category,level,message}], trace{trace_id,span_count,total_ms,bottleneck,error_count,spans}\|null, impact{times_seen,affected_users}, similar_issues[], suspect_commits[], flags{has_trace,source_context,unsymbolicated,event_pending,truncated}, token_estimate}`. Fix from this; **don't** reconstruct it by hand from `get_issue`/`get_trace`. |

---

## Notes

- **Tool naming.** On the wire the tools are namespaced (e.g. `sentryx.search_issues`); your client surfaces them under the `sentryx` server. This reference uses the bare names.
- **`project_id` defaults** to your org's single/first project. Pass it explicitly only when your org has multiple projects.
- **Scopes.** Tokens carry `project:create`, `define:write`, and `read`. If a write tool returns "api token lacks required scope", mint a token with the right scopes in `/settings` (see `connect.md`).
- **The `track()` backbone.** Product events posted to `track_url` must carry `distinct_id` + `user_id` + `session_id`, and `trace_id` when in a request — that's what correlates a funnel event to the error/trace in the same flow. See `example-instrumentation.md`.
