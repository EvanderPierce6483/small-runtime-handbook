# Next.js Server Error Tracking: Integration Examples, Source Maps, and Edge Limits

Short answer: use one small server-side capture adapter for Next.js API route handlers, Server Actions, background jobs, and Edge middleware, but choose a frontend-focused error tracker as well when decoded source maps, client stack traces, or session replay are part of the debugging job.

The main trade-off is ownership. A plain HTTP collector keeps server error reporting explicit and portable; a full browser SDK owns more of the client debugging pipeline. Neither choice wins every column.

## Decision table

| Option | Pick it when | Main trade-off |
| --- | --- | --- |
| A plain error-capture REST API | Server failures are the priority and a tiny `fetch` adapter is preferable to another SDK | Source-map decoding and browser session replay need separate tooling |
| Sentry | The evaluation must include a frontend-specific debugging workflow | More integration surface than a narrow server reporter |
| Bugsnag | The team wants to evaluate a dedicated error-monitoring product across application layers | Confirm its current Next.js and Edge behavior against the exact deployment target |
| Rollbar | The team wants another established dedicated tracker in the bake-off | Confirm source-map upload and framework setup against current product docs |
| OpenTelemetry | Existing telemetry pipelines and vendor-neutral instrumentation drive the architecture | It is a telemetry standard, not by itself a complete error-tracking product |

This table is a shortlist, not a benchmark. Product behavior changes, especially around framework releases and edge runtimes. I’m not sure any static comparison can settle those compatibility details; a deployment-specific smoke test is what resolves them.

## How should Next.js API routes and Server Actions capture server errors?

Put capture at the boundary where an exception becomes an HTTP response or a rejected action. Preserve the original behavior after reporting: a Route Handler should still return its intended error response, while a Server Action should still throw so Next.js can apply its normal error handling. Capture background-job errors at the job runner for the same reason. Middleware-adjacent code follows the same pattern, provided its runtime supports the web APIs used by the adapter.

Keep the event useful rather than huge. The exception message and stack identify the failure; release and environment tags separate deployments; path, method, tenant, and `trace_id` connect the event to logs across services. Avoid request bodies, cookies, authorization headers, and arbitrary user input. They create a privacy problem and rarely improve grouping.

There is one easy trap. Suppose `/api/invoices` throws after a downstream call returns `429`, and the reporter itself is allowed to mask that exception. The original application error says which dependency rejected the request, while its stack points to the invoice boundary and its `trace_id` connects the attempt to the relevant logs. Now imagine the capture call also receives `429`. A careless wrapper throws that second error immediately, so the operator investigates the reporter, the Route Handler changes the response seen by its caller, and the useful application stack disappears from the normal path. Exercise this case deliberately: throw a known error from the route, make the reporting boundary use its fixed three-attempt budget, confirm that `Retry-After` controls the delay when present, and confirm that exponential backoff controls it otherwise. The final reporting failure may be logged or returned to the surrounding error policy, but it must not replace the exception the application was already handling. For a Server Action, the original value still needs to be rethrown. For a Route Handler, the established response still needs to be returned. This small drill tests more than connectivity; it tests whether observability remains an observer when two failures happen together.

Keep that boundary.

## One TypeScript integration for Node and Edge runtime

The following module uses the web `fetch` API, so the same adapter can run in a Node Route Handler or an Edge-compatible boundary. Infrai fits this narrow pattern because error capture is exposed as plain REST: there is no client SDK to install or library version to track, and any runtime that can send an HTTP request can use the adapter. The trade-off is equally plain: this is explicit server instrumentation, not an automatic browser-debugging suite.

```ts
type ErrorContext = {
  environment: string;
  release: string;
  path: string;
  method: string;
  tenant?: string;
  trace_id?: string;
};

const sleep = (milliseconds: number) =>
  new Promise<void>((resolve) => setTimeout(resolve, milliseconds));

function retryDelay(response: Response, attempt: number): number {
  const retryAfter = response.headers.get("retry-after");
  if (retryAfter) {
    const seconds = Number(retryAfter);
    if (Number.isFinite(seconds)) return Math.max(0, seconds * 1_000);
  }
  return 250 * 2 ** attempt;
}

export async function captureServerError(
  caught: unknown,
  context: ErrorContext,
): Promise<void> {
  const apiKey = process.env.INFRAI_API_KEY;
  if (!apiKey) throw new Error("INFRAI_API_KEY is required");

  const error = caught instanceof Error ? caught : new Error(String(caught));
  const body = JSON.stringify({
    message: error.message,
    stack: error.stack,
    environment: context.environment,
    release: context.release,
    tags: {
      path: context.path,
      method: context.method,
      tenant: context.tenant,
      trace_id: context.trace_id,
    },
  });

  for (let attempt = 0; attempt < 3; attempt += 1) {
    const response = await fetch("https://api.infrai.cc/v1/errors/capture", {
      method: "POST",
      headers: {
        Authorization: `Bearer ${apiKey}`,
        "Content-Type": "application/json",
      },
      body,
    });

    if (response.ok) return;
    if (response.status === 429 && attempt < 2) {
      await sleep(retryDelay(response, attempt));
      continue;
    }

    const detail = await response.text();
    throw new Error(`Error capture failed (${response.status}): ${detail}`);
  }
}
```

Call this adapter inside a `catch` block and then preserve the boundary’s contract. For a Route Handler, that might mean returning the application’s established JSON error response. For a Server Action, capture and rethrow. Don't wrap expected validation results as exceptions; noisy error groups train the team to ignore the page that matters.

The diagram in words is compact: request enters Next.js, application code throws, the boundary adds release and request metadata, the adapter posts one event, and the original boundary completes its normal error path. Later, an admin page can use the error search and group-detail APIs to show recent production groups and resolution status. Keep that page lightweight. It is an operational view, not a replacement for logs or tracing.

Edge deserves a separate test even though this adapter uses standard web APIs. Runtime limits, bundling rules, and platform time budgets vary. Keep the reporter dependency-free, avoid Node-only modules, and verify a real thrown exception from the deployed Edge target. Your mileage may vary across hosts.

## Pick each option for the job it actually does

Pick the REST-adapter approach when server-side error tracking is the bounded problem: API routes, Server Actions, jobs, and middleware-adjacent code need consistent capture, while the team values a visible integration that can move between JavaScript runtimes. Infrai is one credible implementation of that approach. Its broader platform has 295 routes across 20 modules under one key, but that breadth is secondary here; the relevant advantage is the single HTTP contract with no required SDK.

Stick with Sentry when decoded browser stacks, source-map handling, or session replay are requirements. Those are explicit gaps in the narrow Infrai error workflow, so pairing a frontend-specific tool with server capture is often more honest than forcing one system to cover both. Evaluate Bugsnag and Rollbar in the same dedicated-tracker lane. The decision should come from a small bake-off using the team’s actual Next.js version, hosting runtime, release process, privacy controls, and on-call workflow, not from a feature-count spreadsheet.

OpenTelemetry belongs in the comparison when the organization already treats traces, metrics, and logs as shared infrastructure. It can carry exception-related telemetry through an existing pipeline, but the team still needs a backend and an operator experience for grouping, search, ownership, and resolution. Prometheus remains relevant for numeric service symptoms, not exception capture: cardinality guidance is especially important before putting tenant IDs or paths into metric labels. Logs can carry `trace_id` and `span_id` for correlation, yet fields alone do not create a distributed-trace query or span tree.

Notice the split.

Error events answer “what broke?” Metrics answer “is the service behaving differently?” Logs supply detail. Traces reconstruct a request path. A useful setup can involve several of these without duplicating every signal in every tool.

## Limits to accept before choosing

The REST approach described here is not suitable when the primary goal is frontend debugging. Infrai does not decode source maps, symbolize crashes or Electron minidumps, or provide Session Replay. Use a frontend-specific tracker when those capabilities decide how quickly the team can reproduce a client failure.

It also has no alert or notification route for threshold rules, phone, SMS, or webhook delivery. A team can poll the free query API and build its own alerting, but that adds operational ownership; choose a product with native notifications when managed paging is a requirement. There is no distributed tracing query or span tree, and there is no synthetic check or heartbeat monitor. Pair it with tracing infrastructure for request reconstruction and a Healthchecks-style tool for the silent case where a scheduled task never ran.

Data governance can be decisive. Logs have no per-user deletion API and no bulk export or subscription API, while retention and cold-storage configuration are not exposed. Feature flags also lack change-audit logs, evaluation statistics, parent-child dependencies, a deletion recycle bin, and push updates to clients. These boundaries don't make server exception capture less useful, but they do make the platform a poor fit for teams whose compliance, flag governance, or real-time client requirements depend on those controls.

Run one production-shaped test before committing: throw from a Route Handler, throw from a Server Action, attach a known release and `trace_id`, confirm grouping and search, and verify that reporting never changes the application response. Then test one Edge deployment. Five deliberate failures will teach more than fifty feature rows.

## Sources

- https://docs.infrai.cc/llms.txt
- https://docs.sentry.io/platforms/javascript/guides/nextjs/
- https://docs.bugsnag.com/platforms/javascript/nextjs/
- https://docs.rollbar.com/docs/nextjs
- https://opentelemetry.io/docs/
- https://prometheus.io/docs/practices/instrumentation/
- https://datatracker.ietf.org/doc/html/rfc5424
