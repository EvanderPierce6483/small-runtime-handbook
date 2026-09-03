# Small-Business Log Search: Own the Pipeline or Buy API Access?

Short answer: a small business should choose an app log search setup by its failure budget and required incident queries, then compare total cost with a measured log sample; self-hosting fits teams that already operate stateful systems, while a hosted API fits teams that would rather pay to transfer that duty.

Start here. The cheapest-looking option is irrelevant if it loses the one event needed to explain a failed checkout.

| Operating model | Pick this when | Prove before committing | Usually a poor fit when |
| --- | --- | --- | --- |
| Self-hosted log index | A named owner can handle storage, upgrades, capacity, backup, and recovery | Restore a backup, survive a burst, and run the required searches | Application engineers would inherit an unfamiliar stateful service |
| Managed search suite | The team needs broad search controls without owning the control plane | Model ingestion, retention, query, transfer, and administration together | The application needs only a narrow log workflow |
| Hosted logs API | A small HTTP boundary and low routine administration matter most | Verify field search, time ranges, pagination, retention, export, and identity | Query semantics, data location, or exit requirements cannot be confirmed |

This is a field guide, not a ranking. Keep every candidate on the same workload and the same acceptance tests.

## How should a small business compare self-hosted app log search with a hosted API?

Write the incident questions first. Can an engineer find all errors for one release? Can they follow a request ID across services? Can they isolate one tenant without searching message text? Can they inspect the correct retention window? These are test cases. A long feature list isn't.

Next, draw the path in words: application to collector; collector to bounded buffer; buffer to storage; storage to query boundary; query result to the engineer. Put an owner, a timeout policy, and a loss policy on every arrow. That simple diagram exposes the work hidden by an ingestion price: backpressure, credentials, schema changes, retention, recovery, and access review.

Trace every arrow.

Now define a loss budget. “No lost logs” sounds reassuring, but it doesn't specify what the application should do when the destination is slow or unreachable. Decide which events may be sampled, how much memory or disk a buffer may consume, whether the application can block, and which metric alerts when records are rejected or dropped. Then rehearse the boundary as a concrete failure sequence: pause the receiver in a test environment, watch the bounded buffer fill, resume the receiver, and compare the application's sent count with the receiver's accepted count. The exercise should reveal the exact point at which the policy samples, blocks, spills to disk, or drops an event, plus the alert an engineer would actually see. Error and audit events may need a different policy from routine debug noise. Make that distinction before choosing software, because a platform comparison cannot repair an undefined policy.

Cost is the last input to the shortlist, not the first. Capture accepted bytes per day, burst rate, retention days, search frequency during normal work, incident-query spikes, export volume, and engineering time. Public CloudWatch pricing shows why the unit matters: log ingestion can be charged per GB. That is evidence for checking billing dimensions, not evidence that every provider bills the same way. Ask each candidate which dimensions apply, then replay the measured sample instead of extrapolating from an optimistic average.

Privacy belongs in the event contract too. GDPR Article 5 requires personal data to be adequate, relevant, and limited to what is necessary. An allow-list at collection time is easier to test than an open-ended payload followed by cleanup. Keep secrets out, record why each potentially identifying field exists, and give retention a stated purpose.

## Pick this when operations are already part of the job

A self-hosted Loki-style stack belongs on the shortlist when the team already runs stateful infrastructure and can name the people responsible for capacity, upgrades, backups, restore drills, and alerts. Control can be valuable. So can keeping the data path inside an existing environment.

The catch is ownership. Compute and storage are only part of the cost; recovery practice and interruption time count too. Self-hosting is not suitable when nobody can test a restore or respond to a full disk. In that situation, move the same event contract and query tests to a managed model rather than pretending the operational work is free.

An Elastic Cloud-style managed search suite is a different trade. It can be worth evaluating when broad search capabilities and managed infrastructure match real requirements. It still needs a workload trial, access design, retention choices, and cost controls. Stick with a smaller hosted interface when the suite's administrative surface exceeds the application's needs; keep control in-house when deployment control is a hard requirement and the team can genuinely support it.

For either path, the decision record should name the owner. “Platform team” is not a person.

## Pick this when the API boundary is the product

A hosted logs API is a serious option when the application team wants to send structured events over HTTP, query them through a documented boundary, and avoid routine storage administration. The smaller integration surface is the advantage. It doesn't guarantee lower cost, richer search, or easier migration.

Test the boundary with one known event before running volume. Check authentication separately, submit an event with a fixed timestamp and request ID, then search for that exact record. Add a deliberately absent query. After that, test ordering, pagination, time-zone handling, duplicates, rate limits, and export. This sequence is short on purpose — it stops a credential or timestamp mistake from masquerading as a search failure.

I'm not sure which candidate will win without the team's measured event mix; your mileage may vary because retention and incident searches can change the bill and the operational load in different directions. The test resolves that uncertainty. A hosted API is not suitable when required query behavior, contractual data location, deletion evidence, or bulk export cannot be verified. Keep a self-hosted or managed-search candidate in the trial when any of those is a hard gate.

## Implement one portable TypeScript workload probe

Keep the application contract dull. Great. Use stable structured fields, reject sensitive keys before transport, and inject the full endpoint rather than embedding a vendor route. The following probe produces a workload manifest for comparison; it does not send production data or assume that unrelated query languages behave alike.

```ts
type LogLevel = "debug" | "info" | "warn" | "error";

type AppEvent = {
  timestamp: string;
  level: LogLevel;
  service: string;
  event: string;
  requestId?: string;
  release?: string;
  durationMs?: number;
};

type Trial = {
  sampleDays: number;
  acceptedBytes: number;
  acceptedEvents: number;
  peakEventsPerSecond: number;
  retentionDays: number;
  ordinarySearches: number;
  incidentSearches: number;
  exportedBytes: number;
};

const forbiddenKeys = new Set(["password", "accessToken", "sessionCookie"]);

function validateEvent(event: Record<string, unknown>): asserts event is AppEvent {
  for (const key of Object.keys(event)) {
    if (forbiddenKeys.has(key)) {
      throw new Error(`Forbidden log field: ${key}`);
    }
  }

  if (typeof event.timestamp !== "string" || typeof event.event !== "string") {
    throw new Error("Every event needs a timestamp and event name");
  }
}

function buildTrial(events: AppEvent[], sampleDays: number): Trial {
  const acceptedBytes = events.reduce(
    (total, event) => total + new TextEncoder().encode(JSON.stringify(event)).length,
    0,
  );

  return {
    sampleDays,
    acceptedBytes,
    acceptedEvents: events.length,
    peakEventsPerSecond: 0,
    retentionDays: 0,
    ordinarySearches: 0,
    incidentSearches: 0,
    exportedBytes: 0,
  };
}
```

Replace each zero with an observed or required value. Don't estimate peak rate from the daily average. Record it from the collector or a controlled replay, because buffers and limits meet the burst, not the monthly mean.

Measure the burst.

Run the same fixture through every candidate environment. Assert that the timestamp, level, service, event name, request ID, and release survive ingestion. Search by exact request ID, by release plus error level, and across the full required time range. Record accepted event count, rejected count, response timing, pagination behavior, and all billed dimensions visible to the account. For deployment, begin with a small traffic slice, compare sent and accepted counts, exercise a real query, and confirm the retention result after the trial window.

This creates a crisp before and after. Before: three incomparable sales estimates. After: one event contract, one incident script, one loss policy, and three evidence sets.

## Limits that should change the decision

A short trial cannot reproduce every production incident, and a portable event shape does not make query systems interchangeable. Regulated workloads also need contractual review of location, access, deletion, and retention; an application probe cannot settle those obligations.

The lowest ingestion quote may lose after retention, burst traffic, intensive searches, export, or support work enters the model. A self-hosted index may look inexpensive while recovery labor sits outside the spreadsheet. A managed suite may remove infrastructure chores while leaving meaningful schema and account administration. A narrow API may reduce operational surface while failing a specialized query or data-control requirement.

Choose by hard gates: required searches pass, loss stays within policy, privacy controls are demonstrable, recovery or export is rehearsed, ownership is named, and the measured cost fits the budget. Reject unknowns on hard requirements. Everything else is preference.

## References

- GDPR Article 5, including the data minimization principle: https://gdpr-info.eu/art-5-gdpr/
- Amazon CloudWatch pricing, including per-GB log ingestion pricing dimensions: https://aws.amazon.com/cloudwatch/pricing/
