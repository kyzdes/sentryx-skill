# Common SentryX workflows

End-to-end recipes for what agents do most with SentryX. Each assumes you've
connected and can call the `sentryx` MCP tools (see `connect.md`). The golden
rule runs through all of them: **`get_started` first, pull only what the task
needs, don't read every issue/trace to "understand the app".**

Distilled from the canonical runbook:
<https://github.com/kyzdes/sentryx/blob/main/docs/agent-onboarding.md>.

---

## 0. First contact

```
whoami           # confirm the Bearer token resolved → { org, scopes }
get_started      # orient: projects, what's instrumented, recommended next step
```
If there's no project yet, go to recipe 1. If errors/funnels already exist, you
can jump straight to recipe 2 (fix a bug) or 3 (analyze a funnel).

---

## 1. Onboard a new project end-to-end (the hero path)

Raw repo → errors + traces + funnels live in the web.

```
# a) Detect the stack (read the repo, no MCP call):
#    package.json → node/nextjs · requirements.txt/pyproject.toml → python
#    go.mod → go · SPA entry → browser

# b) Ask the 5 product questions (see SKILL.md) — money action, 3–5 steps,
#    user-vs-session, features, conversion window.

# c) Provision (idempotent on slug):
create_project(name: "ShopCo", slug: "shopco")
  -> { project_id, dsn, otlp_http, otlp_grpc, track_url, is_new }

# d) Get snippets for the detected stack:
instrument_hint(stack: "nextjs", dsn: <dsn>, track_url: <track_url>)
  -> { errors_snippet, tracing_snippet, track_snippet, notes }

# e) Apply all three to the code (see example-instrumentation.md):
#    - Sentry init with <dsn> (+ release, environment)
#    - OTel exporter → otlp_http / otlp_grpc
#    - track() at each funnel step + each feature; EVERY call carries
#      distinct_id + user_id + session_id, and trace_id when in a request.

# f) Register the definitions:
define_event(name: "visit")
define_event(name: "signup")
define_event(name: "activate")
define_event(name: "subscribe")
define_funnel(key: "activation", name: "Visit → Subscribe",
              steps: ["visit","signup","activate","subscribe"],
              window_seconds: 86400)
define_feature(key: "checkout", event_names: ["checkout_started","purchase"])

# g) Verify data flows — trigger the flow, then:
get_funnel(funnel_key: "activation")
  -> steps with monotonically decreasing counts + biggest_drop
```
When counts show up and decrease step-over-step, instrumentation is correct. The
project now appears in the web: **/issues** (errors), **/p/&lt;slug&gt;/funnels**
(funnels + drop-off), traces correlated by `trace_id`. Place a `track()` call at
**every** funnel step (Q2) and every feature (Q4) — a missing step shows as a
zero/flat segment in `get_funnel`.

---

## 2. Fix a high-impact error

```
search_issues(status: "unresolved", level: "error", limit: 20)
  -> pick the issue with the highest times_seen / affected_users

prepare_fix_bundle(issue_id: 42)
  -> summary, exception, in_app_frames + source context, breadcrumbs,
     correlated trace (with bottleneck), impact, similar_issues, suspect_commits
```
Fix **from the bundle** — it's already the in-app frames + the line that threw +
the breadcrumbs leading up to it + the trace it happened in + the commits that
likely caused it. Reach further only if needed:

```
get_issue(issue_id: 42)          # latest_event summary → grab trace_id
get_trace(trace_id: "<32-hex>")  # spans + correlated errors, end-to-end
```
Don't hand-reconstruct what `prepare_fix_bundle` already returns. Check
`flags.unsymbolicated` (minified browser frames aren't symbolicated yet) and
`flags.event_pending` (the event may still be ingesting).

### Close the loop — assisted autonomy (agent-side PR, SentryX audits)

SentryX **records + audits** the fix and tracks each issue's `fix_status`; **you**
write the patch and open a **DRAFT** PR with `gh`. SentryX never patches code —
there is no server-side patch service. Every write tool reuses the `define:write`
scope, is org-scoped, and enforces project + issue ownership. `record_fix_attempt`
and `link_pr` are idempotent on `idempotency_key`, so a retried step audits once.

```
# 0) (optional) is a fix already in flight? don't open a duplicate PR.
list_fix_attempts(issue_id: 42)
  -> { actions:[{action_type,pr_url,status,created_at}], count }

# 1) you're starting the fix — record it BEFORE touching code:
record_fix_attempt(issue_id: 42,
                   summary: "guard nil user in checkout handler",
                   idempotency_key: "fix-42-a1b2c3d")
  -> { action_id, issue_id, fix_status: "investigating", was_new }

# 2) write the patch, then open a DRAFT PR YOURSELF (never push to a protected branch):
#    git checkout -b fix/issue-42
#    git commit -am "Fix: guard nil user in checkout handler"
#    gh pr create --draft --title "Fix: guard nil user in checkout handler" \
#      --body "Fixes SentryX issue #42. Root cause: <from prepare_fix_bundle>."

# 3) link the PR you just opened:
link_pr(issue_id: 42,
        pr_url: "https://github.com/acme/shop/pull/17",
        idempotency_key: "pr-42-17")
  -> { action_id, issue_id, fix_status: "pr_open", pr_url, was_new }

# 4) a HUMAN reviews + merges the draft PR (you only opened a draft).

# 5) after merge/deploy, resolve the issue:
resolve_issue(issue_id: 42)
  -> { issue_id, status: "resolved", fix_status: "fixed" }
```

Lifecycle: `none` → `investigating` (record_fix_attempt) → `pr_open` (link_pr) →
`fixed` (resolve_issue, which also sets the canonical issue `status='resolved'`).
Always **draft** the PR and let a human merge — autonomy here means SentryX
records and audits the loop, not that the agent self-merges.

---

## 3. Analyze a funnel drop-off

```
get_started                          # which funnels exist
get_funnel(funnel_key: "activation")
  -> steps:[{name,count}], total_entered,
     biggest_drop:{from_step, to_step, lost, rate}
```
`biggest_drop` points at the step pair losing the most users. To decide whether
it's a **bug** or a **UX gap**:

```
# Same step has errors? correlate:
search_issues(q: "<the dropping step's route/handler>")
prepare_fix_bundle(issue_id: <n>)    # if there's an error on that path → fix it
```
A flat/zero step usually means a **missing `track()` call** at that step, not a
real drop — re-check the instrumentation (recipe 1e) before concluding users are
churning. A steep `rate` with no errors is a UX/product signal: prioritize that
step for the team.

---

## Notes & current limits

- **Single org per token** today; multi-org / richer human UI is roadmap.
- **Funnels** use ClickHouse `windowFunnel` (ordered, within the window). Flows
  (`sequenceMatch`), retention/cohorts, session replay, and RUM/web-vitals are
  roadmap.
- **Minified browser JS frames aren't symbolicated yet** (source maps are a
  fast-follow); `prepare_fix_bundle` flags this as `unsymbolicated`.
- `sentryx.moone.dev` is the **web UI**; the MCP lives at
  `https://mcp.moone.dev/api/mcp` (fallback `https://ingest.moone.dev/api/mcp`).
