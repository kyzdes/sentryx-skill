# Instrumentation snippets per stack

Copy-paste instrumentation for each stack. These mirror what
`instrument_hint(stack, dsn, track_url)` returns — call that tool to get the
**same snippets pre-filled** with your project's real `dsn` / `otlp_*` /
`track_url` (from `create_project`). Three pieces per stack:

1. **Errors** — Sentry SDK init with the `dsn` (+ `release`, `environment`).
2. **Tracing** — point the OpenTelemetry (OTLP) exporter at `otlp_http` / `otlp_grpc`.
3. **`track()`** — the product-events helper that POSTs to `track_url`.

> **The correlation backbone.** Every `track()` call MUST carry `distinct_id` +
> `user_id` + `session_id`, and `trace_id` **when in a request**. That is what
> ties a product event to the error/trace in the same flow — drop it and funnels
> stop deduping per user and events stop correlating to traces. Call `track()`
> at **each funnel step** and **each feature** you defined.

Placeholders: `<DSN>` = `create_project.dsn`, `<TRACK_URL>` =
`create_project.track_url`, `<OTLP_HTTP>` / `<OTLP_GRPC>` = `create_project.otlp_http` /
`otlp_grpc`.

---

## Node

```js
// errors — @sentry/node, as early as possible in startup
import * as Sentry from "@sentry/node";
Sentry.init({ dsn: "<DSN>", release: "myapp@1.0.0", environment: "production", tracesSampleRate: 1.0 });
```
```bash
# tracing — env for the OTLP exporter
OTEL_EXPORTER_OTLP_ENDPOINT=<OTLP_HTTP>
OTEL_RESOURCE_ATTRIBUTES=service.name=myapp
```
```js
// product events
async function track(event, props = {}, ids = {}) {
  await fetch("<TRACK_URL>", {
    method: "POST",
    headers: { "Content-Type": "application/json" },
    body: JSON.stringify({ events: [{
      event,
      distinct_id: ids.distinctId,
      user_id: ids.userId,
      session_id: ids.sessionId,
      trace_id: ids.traceId,        // include inside a request
      properties: props,
    }] }),
  });
}
```

## Next.js

```js
// errors — @sentry/nextjs, in instrumentation.ts / sentry.*.config.ts
import * as Sentry from "@sentry/nextjs";
Sentry.init({ dsn: "<DSN>", release: process.env.RELEASE, environment: "production", tracesSampleRate: 1.0 });
```
```bash
# tracing — env
OTEL_EXPORTER_OTLP_ENDPOINT=<OTLP_HTTP>
OTEL_RESOURCE_ATTRIBUTES=service.name=myapp,deployment.environment=production
```
```js
// product events (route handler / server action / client)
export async function track(event, props = {}, ids = {}) {
  await fetch("<TRACK_URL>", {
    method: "POST",
    headers: { "Content-Type": "application/json" },
    body: JSON.stringify({ events: [{
      event,
      distinct_id: ids.distinctId,
      user_id: ids.userId,
      session_id: ids.sessionId,
      trace_id: ids.traceId,
      properties: props,
    }] }),
  });
}
```

## Python

```python
# errors — sentry-sdk, at startup
import sentry_sdk
sentry_sdk.init(dsn="<DSN>", release="myapp@1.0.0", environment="production")
```
```bash
# tracing — env for the OTLP exporter
OTEL_EXPORTER_OTLP_ENDPOINT=<OTLP_HTTP>
```
```python
# product events
import httpx

def track(event, props=None, distinct_id=None, user_id=None, session_id=None, trace_id=None):
    httpx.post("<TRACK_URL>", json={"events": [{
        "event": event,
        "distinct_id": distinct_id,
        "user_id": user_id,
        "session_id": session_id,
        "trace_id": trace_id,       # include inside a request
        "properties": props or {},
    }]})
```

## Go

```go
// errors — getsentry/sentry-go, at startup
sentry.Init(sentry.ClientOptions{
    Dsn:         "<DSN>",
    Release:     "myapp@1.0.0",
    Environment: "production",
})
```
```bash
# tracing — env (gRPC or HTTP)
OTEL_EXPORTER_OTLP_ENDPOINT=<OTLP_GRPC>   # gRPC, e.g. host:4317
# or
OTEL_EXPORTER_OTLP_ENDPOINT=<OTLP_HTTP>   # HTTP
```
```go
// product events — POST <TRACK_URL>
// body: {"events":[{"event":"...","distinct_id":"...","user_id":"...",
//                    "session_id":"...","trace_id":"...","properties":{}}]}
//
// func track(event, distinctID, userID, sessionID, traceID string, props map[string]any) {
//   body, _ := json.Marshal(map[string]any{"events": []map[string]any{{
//     "event": event, "distinct_id": distinctID, "user_id": userID,
//     "session_id": sessionID, "trace_id": traceID, "properties": props}}})
//   http.Post("<TRACK_URL>", "application/json", bytes.NewReader(body))
// }
```

## Browser

```js
// errors — @sentry/browser
Sentry.init({ dsn: "<DSN>", environment: "production" });
```
```js
// tracing — browser RUM/OTLP is roadmap; send product events via track() below
```
```js
// product events — sendBeacon survives page unloads
function track(event, props = {}, ids = {}) {
  navigator.sendBeacon("<TRACK_URL>", JSON.stringify({ events: [{
    event,
    distinct_id: ids.distinctId,
    user_id: ids.userId,
    session_id: ids.sessionId,
    properties: props,
  }] }));
}
```

---

## After wiring it up

1. Trigger the flow (run the app / a smoke script) so events land.
2. `define_event` / `define_funnel` / `define_feature` for what you emit.
3. `get_funnel(funnel_key)` — monotonically decreasing counts confirm the
   `track()` calls fire and correlate. A flat/zero step ⇒ a missing `track()`
   at that step.
4. Errors and traces show in the web at **/issues** and via `trace_id`
   correlation; product funnels at **/p/&lt;slug&gt;/funnels**.

> Note: minified browser JS frames aren't symbolicated yet (source maps are a
> fast-follow); `prepare_fix_bundle` flags affected frames as `unsymbolicated`.
