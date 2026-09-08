# Polling Error Events for Node.js SaaS Alerting: Query API, Webhook, and Cron Design

Short answer: for a Node.js SaaS, use a webhook as the fast path, a cursor-based error events query API as the recovery path, and a scheduled check as proof that both paths are still alive.

Do not ask Slack or email to be the monitoring system. They are destinations. The actual system needs to capture structured failure events, preserve enough state to replay them, and detect silence when no event arrives. That distinction is the difference between an alerting pipeline and a notification script.

Here is the before/after model.

Before: application error -> one outbound webhook -> one chat message. If that request gets a `429`, the chain ends unless somebody designed a retry.

After: application error -> durable event store -> webhook delivery attempt -> query-based reconciliation -> deduplicated Slack or email notification. One route is quick. The other repairs gaps.

That is the design to evaluate, regardless of which service or self-hosted components provide it.

## How should a Node.js SaaS combine an error events query API, cron, and webhooks?

Start with one canonical event, not three destination-specific payloads. OpenTelemetry defines a log record as a timestamped record with attributes, and its logs data model includes fields such as event time, observed time, severity, body, trace context, resource, and attributes. You don't need every field on day one, but using that shape prevents an early Slack integration from becoming the permanent schema.

A practical event envelope has an immutable event ID, the time the failure occurred, a stable fingerprint, a severity, a service name, a short message, and trace context when available. Put high-cardinality request IDs in attributes rather than in the fingerprint. Otherwise every occurrence becomes a new alert group.

Then split ingestion from delivery. The request handler records the event and returns; a separate delivery worker sends notifications. A webhook can provide low-latency delivery, while a query API exposes events ordered by a stable cursor. The scheduled reconciler asks, "Which events after cursor X have not produced a notification?" It advances the cursor only after processing succeeds.

Tiny detail. Huge consequence.

Offset pagination is a poor fit for a changing event stream because inserts can move page boundaries while a reader is paging. A stable cursor tied to an ordering key makes the recovery loop easier to reason about. The consumer should also tolerate seeing the same event twice. Exactly-once delivery across storage, a webhook, and a chat or mail provider is not a useful assumption; idempotent handling is the useful requirement.

The diagram in words is straightforward: **write once, attempt push, recover by pull, notify once**. The event ID connects all four stages. Store a notification record keyed by `(destination, eventId)` or use an equivalent uniqueness constraint, and a retry cannot create a second page.

Cron is only the clock here. It is not the source of truth. A platform scheduler, a container scheduler, or a long-running worker can trigger the same reconciliation function; what matters is that concurrent runs cannot corrupt the cursor and that a delayed run catches up instead of dropping a time window.

## A copyable TypeScript reconciliation loop

The following example keeps vendor details behind two narrow interfaces. It assumes the query endpoint returns events in ascending cursor order and that `saveIfNew` atomically claims an event for a destination. Those are contract requirements to verify during evaluation, not hidden implementation details.

```ts
type ErrorEvent = {
  id: string;
  occurredAt: string;
  fingerprint: string;
  severity: "warning" | "error" | "critical";
  service: string;
  message: string;
  traceId?: string;
};

type EventPage = {
  events: ErrorEvent[];
  nextCursor?: string;
};

interface EventSource {
  query(after?: string): Promise<EventPage>;
}

interface AlertState {
  loadCursor(): Promise<string | undefined>;
  saveIfNew(destination: string, eventId: string): Promise<boolean>;
  commitCursor(cursor: string): Promise<void>;
}

interface Notifier {
  send(event: ErrorEvent): Promise<void>;
}

export async function reconcile(
  source: EventSource,
  state: AlertState,
  notifier: Notifier,
  destination: string,
): Promise<void> {
  let cursor = await state.loadCursor();

  for (;;) {
    const page = await source.query(cursor);

    for (const event of page.events) {
      const claimed = await state.saveIfNew(destination, event.id);
      if (claimed) await notifier.send(event);
    }

    if (!page.nextCursor || page.nextCursor === cursor) return;
    await state.commitCursor(page.nextCursor);
    cursor = page.nextCursor;
  }
}
```

There is an important trade-off in the claim-before-send sequence. It gives at-most-once notification behavior if the process stops after the claim but before the send. Claiming after the send gives at-least-once behavior and can duplicate a notification if the process stops between those operations. A production design can model `pending`, `sent`, and retryable delivery states, but it still cannot erase uncertainty around an external side effect unless the destination accepts an idempotency key.

For paging, I would choose at-least-once and deduplicate visibly by event ID; for a low-priority daily email, at-most-once may be acceptable. That is an operational choice, not a universal answer. I'm not sure a destination is safe to retry until its documentation states the idempotency behavior explicitly.

Test the loop with failure injection. Return the same page twice and expect one notification. Throw on the second page, rerun, and verify that processing resumes from the committed cursor. Make the notifier return a retryable `429`, then check that backoff honors the destination's published rate-limit contract. Finally, run two reconcilers together. If the test sends two alerts for one event, the claim isn't atomic.

Also test emptiness.

An empty page can mean "nothing failed," but it can also mean that ingestion, credentials, networking, or the query itself stopped working. Emit a heartbeat metric from the reconciler with its last-success time and cursor age. Alert on an overdue heartbeat through a route that does not depend on the event query being healthy. Otherwise the recovery path can fail silently while reporting a perfectly calm system.

## Isn't polling an alternative to an exception tracker?

No. Polling is a delivery and reconciliation technique; exception tracking is a larger workflow that may include stack capture, source mapping, grouping, release association, ownership, and issue state. A generic event query can be enough when the requirement is narrowly "alert on failures," but it is not a substitute when developers need rich debugging context and regression triage.

The reverse is also true. An in-process exception SDK sees failures that reach the runtime and its instrumentation. It cannot infer a business outcome that never happened unless the application records an expectation or exposes a separate health signal. A job that was never scheduled, a webhook that was accepted but never processed, and a batch that stopped producing output are absence problems. They need a deadline, a heartbeat, queue-age monitoring, or an expectation ledger.

So define the failure classes before comparing APIs:

- Thrown exceptions need stack and trace context.
- Rejected background work needs queue state and retry metadata.
- Missing scheduled work needs a heartbeat or deadline.
- Failed outbound delivery needs attempt history and response classification.
- User-visible business failures need an outcome signal tied to the affected operation.

If most incidents are thrown exceptions in a single Node.js service, a mature exception-tracking workflow may be the smaller operational burden. Stick with it. If the problem is cross-service delivery and missed side effects, an event store plus reconciliation offers a clearer boundary. Many SaaS teams need both signals, joined by trace IDs and service attributes rather than forced into one tool.

## Should alerts go to Slack, email, or both?

Route by urgency and ownership. Chat is useful for a shared, time-sensitive operational stream, but channels become noisy and messages scroll away. Email is durable and searchable, but it is a weak paging mechanism. Neither should be the only record of alert state.

A small policy table is enough:

| Condition | Destination | Grouping | Expected action |
| --- | --- | --- | --- |
| One isolated retryable failure | Event store only | Fingerprint | Review in normal triage |
| Repeated failures crossing a defined service objective | On-call route plus chat | Service and fingerprint | Acknowledge and mitigate |
| Daily low-priority summary | Email | Service and day | Review trends |
| Reconciler heartbeat overdue | Independent on-call route | Reconciler identity | Restore detection coverage |

Avoid hard-coding "error means Slack" in application code. Put routing after grouping, because ten thousand identical events are one incident, not ten thousand conversations. Apply a quiet period, retain the count and oldest occurrence, and send an update when severity changes. Don't discard the underlying events just because the notification was grouped.

Email also changes the data-handling question. Stack traces and log bodies can carry personal data, secrets, or request content. Redact before storage when possible, restrict what enters notification payloads, and set retention intentionally. OpenTelemetry's guidance makes the correlation benefit clear: logs can carry trace and resource context. It does not make every attribute appropriate for every destination.

## The selection checklist and its limits

The best simple API is the one whose failure semantics you can test. Before adopting one, verify cursor stability, retention, filtering by service and severity, documented rate limits, webhook retry behavior, event IDs, timestamps, trace correlation, and an export path. Check authentication rotation and auditability too. A five-line happy-path request tells you almost nothing about operating the integration at 03:00.

There is a real cost to this architecture. You own cursor state, deduplication, retry policy, heartbeat monitoring, and tests across two delivery paths. It is not suitable when a small team wants stack traces and basic notifications with no appetite for maintaining alert state; an integrated exception tracker is then the more honest choice. It is also unnecessary for a low-volume internal script where one monitored scheduled run and an email failure handler satisfy the impact and recovery requirements.

On the other hand, webhook-only alerting is too fragile when missing one notification has material impact, and query-only alerting is too slow when response time matters. Pair them when both fast awareness and eventual recovery are requirements. Keep one canonical event envelope. Keep the cursor durable. Make every consumer idempotent.

Then pull the network cable in staging and watch what happens.

## References

- OpenTelemetry, "Logs signal concepts": https://opentelemetry.io/docs/concepts/signals/logs/
- OpenTelemetry, "Logs data model": https://opentelemetry.io/docs/specs/otel/logs/data-model/
- OpenTelemetry, "Trace context in non-OTLP log formats": https://opentelemetry.io/docs/specs/otel/compatibility/logging_trace_context/
- MDN, "HTTP response status codes": https://developer.mozilla.org/en-US/docs/Web/HTTP/Reference/Status
- OWASP, "Logging Cheat Sheet": https://cheatsheetseries.owasp.org/cheatsheets/Logging_Cheat_Sheet.html
