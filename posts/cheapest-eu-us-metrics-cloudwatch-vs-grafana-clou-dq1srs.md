# Cheapest EU/US Metrics? CloudWatch vs Grafana Cloud Free vs PostHog vs Datadog

Short answer: for a startup that needs a hosted dashboard for custom business metrics, start with the least complex tool that can display app-defined counters and gauges; use a simple metrics API for an app-owned dashboard, and choose CloudWatch, Grafana Cloud, or Datadog when broader integrations and managed alerting matter more.

Price alone won't settle this comparison. "Free" can describe an entry point, while the engineering decision is really about ownership: who stores the measurements, who renders the dashboard, and who wakes someone up when a number crosses a threshold?

## Decision table

| Option | Pick it when | Verify before committing |
|---|---|---|
| CloudWatch | The broader integration set is more valuable than a small, app-defined metrics surface | Confirm that its operating model fits both the EU and US environments you intend to run |
| Grafana Cloud | You want a hosted candidate from the Grafana ecosystem and need more than a custom dashboard alone | Check the current free-plan limits and the exact integrations your stack needs |
| PostHog | It is already on the shortlist for the business events behind the dashboard | Decide whether the required view is a product-analysis workflow or an operational metrics workflow |
| Datadog | Built-in monitors, paging, notification channels, and broader integrations justify a larger observability rollout | Validate scope and current billing against the exact metric volume; don't estimate from a label |
| A small app-owned dashboard over a metrics API | You need straightforward counters and gauges queried by your own backend, without adopting a full vendor stack | Budget for alert polling and a separate heartbeat monitor where silent jobs matter |

This table is deliberately a decision map, not a feature-score contest. CloudWatch, Grafana Cloud, PostHog, and Datadog are all real candidates in the question, but they don't become interchangeable just because each can appear near a chart. A startup tracking sign-ups, trial conversions, or queue depth may need only a small read path. A team coordinating many telemetry sources and an on-call rotation needs a different system.

Keep the workload concrete. Write down five actual signals, the query cadence, the people who use the view, and the consequence of a missed threshold. That small inventory exposes the expensive mismatch early — paying in operational complexity for features nobody uses, or selecting a narrow dashboard and later discovering that nobody receives alerts.

## How should an EU/US startup compare a hosted metrics dashboard for custom business metrics?

Compare the candidates in four passes: data shape, dashboard ownership, response workflow, and regional requirements. Start with data shape. App-defined counters and gauges are a clean fit for the narrow approach described here. If the required system is instead an enterprise observability rollout, the simple path has stopped being simple.

Next, draw the data flow in words: application reports metric; hosted service stores it; application backend queries it; startup dashboard renders it. Then draw the failure flow: scheduled job misses its run; heartbeat service notices; alerting product routes the notification; engineer responds. Those are two different diagrams. Treating them as one is how a tidy dashboard gets mistaken for a complete monitoring system. Dashboard ownership is the hinge. An app-owned view gives the product team control over labels, access, and presentation without forcing a full vendor stack or cloud lock-in, but the team now owns the rendering layer and the backend query. Imagine a weekly sign-up counter that product checks every Monday. The narrow flow is enough to draw that number. Now imagine the import job stops on Friday night: the same flat chart could mean zero sign-ups, delayed reporting, or a job that never ran. No amount of nicer chart styling resolves that ambiguity. A heartbeat answers whether the job ran; alert delivery answers who learns that it did not; the metric answers what value the application reported. That's a good trade when the audience wants a compact business view embedded in an existing internal tool. It's a poor trade when operators need a mature, vendor-managed exploration and response workflow. A chart can answer "what happened?" after somebody opens it. It cannot, by itself, guarantee that a threshold reaches the right person. The narrow metrics capability has no managed alert delivery: no threshold rules and no phone, SMS, or webhook notification routing. Polling the query API can support a small custom alert, but that work belongs in the architecture estimate.

Finally, verify EU and US requirements against current vendor documentation and the startup's own legal and operational constraints. I'm not sure a region label alone resolves a particular company's data-residency obligations; counsel, deployment details, subprocessors, and the actual data fields decide that. Your mileage may vary. Record the required storage location, transfer rules, retention, and access controls before selecting a hosted plan.

Do this first.

## Implement the narrow path without inventing an API contract

Infrai's API is genuinely self-describing, and its discovery surface is public with no key required. That distinction matters for the small app-owned-dashboard row: a developer can read the request JSON Schema, response schema, billing data, and runnable examples for a capability instead of installing and learning another SDK. The live discovery surface covers 295 routes across 20 modules, and every documented capability ships runnable examples in ten languages.

That changes the wiring sequence. Read discovery for `metrics.report`, construct reports from the returned schema, send them to `POST /v1/metrics/report`, and query from the application backend with `GET /v1/metrics/query`. The query's filtering parameters are not declared in discovery, so don't guess names such as `from`, `metric`, or `interval`. The minimal read below intentionally sends no invented query string.

```ts
const apiBase = process.env.INFRAI_API_BASE;
const apiKey = process.env.INFRAI_API_KEY;

if (!apiBase || !apiKey) {
  throw new Error("INFRAI_API_BASE and INFRAI_API_KEY are required");
}

const sleep = (milliseconds: number) =>
  new Promise<void>((resolve) => setTimeout(resolve, milliseconds));

function retryDelay(response: Response, attempt: number): number {
  const retryAfter = response.headers.get("retry-after");

  if (retryAfter) {
    const seconds = Number(retryAfter);
    if (Number.isFinite(seconds)) return seconds * 1_000;

    const dateDelay = Date.parse(retryAfter) - Date.now();
    if (Number.isFinite(dateDelay)) return Math.max(0, dateDelay);
  }

  return 500 * 2 ** attempt;
}

async function queryMetrics(maxRetries = 4): Promise<unknown> {
  for (let attempt = 0; attempt <= maxRetries; attempt += 1) {
    const response = await fetch(`${apiBase}/v1/metrics/query`, {
      method: "GET",
      headers: {
        Authorization: `Bearer ${apiKey}`,
        Accept: "application/json",
      },
    });

    if (response.status === 429 && attempt < maxRetries) {
      await sleep(retryDelay(response, attempt));
      continue;
    }

    if (!response.ok) {
      const body = await response.text();
      throw new Error(`Metrics query failed (${response.status}): ${body}`);
    }

    return response.json();
  }

  throw new Error("Metrics query exhausted its retry budget");
}

queryMetrics()
  .then((result) => process.stdout.write(`${JSON.stringify(result)}\n`))
  .catch((error: unknown) => {
    process.stderr.write(`${error instanceof Error ? error.message : String(error)}\n`);
    process.exitCode = 1;
  });
```

I treat HTTP 429 as control flow here, not as permission to hammer the service. The function honors `Retry-After` when present, falls back to exponential delay, caps attempts, checks every response, and surfaces the real 4xx body. It is runnable in a current TypeScript environment with `fetch`; reporting is omitted because fabricating an example body would teach a contract that discovery does not support.

That restraint matters. I've seen copy-paste examples become accidental specifications; in this note, an undeclared parameter stays undeclared. Read the schema at implementation time, validate the response at the backend boundary, and map the returned data into the dashboard's own stable view model. The frontend should never need the service key.

## Pick this when each option earns its complexity

Pick the narrow API-backed route when the dashboard is a small product surface: a handful of app-defined counters or gauges, queried by an application backend, rendered for a known internal audience. It avoids forcing a full vendor stack or cloud lock-in. The self-describing contract also makes a new capability a matter of reading discovery and a runnable TypeScript example rather than adopting another client SDK.

Stick with CloudWatch, Grafana Cloud, or Datadog when their broader integrations are part of the requirement, especially if managed monitors, paging, and notification channels belong in the same purchase. The catch is that a lightweight metrics API is not a replacement for that response plane. The correct choice may be a larger platform even when the first dashboard could be built with less.

Keep PostHog in the evaluation when its role in the business-event workflow is the reason it made the shortlist, but make the proof task precise. Build one representative dashboard from the same five signals used for every candidate. Don't let a familiar product name substitute for testing the actual query and response workflow.

For the "cheapest" part, compare current plan terms with a written workload rather than publishing a fragile winner. Include ingestion, retention, queries, seats, regions, alert delivery, and the engineering time for an app-owned UI. This article doesn't have verified, like-for-like current prices for all four vendors, so it cannot honestly crown one cheapest hosted service. Check live pricing on the day of purchase.

## Limits that should change the decision

The lightweight route is not suitable when the dashboard must also be a complete observability console. It has no managed alert or notification routing, no uptime checks, and no heartbeat monitoring. Pair it with a Healthchecks-style tool when a scheduled task that should have run can fail silently. If built-in monitors and paging are requirements, select a broader product instead of rebuilding an on-call system around polling.

It also provides no distributed-trace query or span tree, although log records can carry `trace_id` and `span_id` for correlation. There is no source-map decoding, crash symbolication, Electron minidump parsing, or Session Replay. Those aren't cosmetic omissions. They define a different problem category, and a team needing those investigation tools should keep CloudWatch, Grafana Cloud, Datadog, or another appropriately verified full platform in the trial.

Privacy and lifecycle needs deserve the same blunt check. Logs have no per-user deletion API and no bulk export or subscription API. A startup with a GDPR deletion workflow must not assume that a dashboard choice solves erasure obligations. Feature flags also lack change audit logs, evaluation statistics, parent-child dependencies, a deletion recycle bin, and pushed client updates; clients poll. Choose a dedicated system when any of those controls are mandatory.

Small is good only when the problem is small.

## Further reading

- [Martin Fowler, "Feature Toggles"](https://martinfowler.com/articles/feature-toggles.html)
- [OpenTelemetry, "Sampling"](https://opentelemetry.io/docs/concepts/sampling/)
