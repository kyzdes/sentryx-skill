# sentryx — Claude Code plugin

Instrument **error tracking + distributed tracing + product funnels** into your app
with [SentryX](https://sentryx.moone.dev) (a self-hosted Sentry-analog), and
triage/fix issues — all over a remote MCP, from inside Claude Code.

## Install

```
/plugin marketplace add kyzdes/claude-skills
/plugin install sentryx@claude-skills
```

## Connect (once)

1. Mint an org API token at **https://sentryx.moone.dev/settings** (copy it — shown once).
2. Register the MCP:
   ```
   claude mcp add --transport http sentryx https://mcp.moone.dev/api/mcp \
     --header "Authorization: Bearer <YOUR_TOKEN>"
   ```
3. Confirm — ask the agent to run `whoami` / `get_started`.

## What it does

- **Instrument** — detects your stack, asks a few product questions, then
  `create_project` → `instrument_hint` (errors + OTLP traces + a `track()` helper)
  → `define_funnel` / `define_event`.
- **Fix bugs** — `search_issues` → `prepare_fix_bundle`: a ~250–600 token, fix-ready
  bundle (in-app frames + breadcrumbs + correlated trace + suspect commit + impact).
- **Analyze** — `get_funnel` to see where users drop off.

The skill teaches the work cycle (always call `get_started` first to orient cheaply).
Full details live in `skills/sentryx/references/` (tools, workflows, connect, example
instrumentation).
