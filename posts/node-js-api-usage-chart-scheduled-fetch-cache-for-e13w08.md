# Node.js API Usage Chart: Scheduled Fetch Cache for Billing Attribution

TL;DR: Fetch the usage series on a schedule, store every raw response with its fetch time, and make the e-commerce dashboard read that local copy. During a leaked-key drill, the chart must show exactly when its evidence was fetched. If a refresh fails, keep the last chart visible with a stale warning instead of replacing evidence with an empty panel.

This turns a fragile chart into a small attribution ledger. It also stops every browser refresh from spending rate-limit capacity on an identical upstream read. For a drill, that distinction matters: responders need to separate checkout, catalog, and fulfillment usage while credentials are being rotated, without mistaking an old graph for current activity.

## How should a scheduled fetch cache an API usage chart?

Before: each dashboard load asks the usage API for history. Ten responders opening or refreshing the incident view create ten upstream reads, yet none adds new billing evidence. A failed request can also produce an empty chart, which looks dangerously similar to zero usage.

After: one scheduled Node.js job fetches the series. Postgres stores the untouched response, the successful fetch time, and the key alias used for attribution. The dashboard reads the newest successful snapshot. A refresh failure changes the warning state, not the historical evidence.

That is the whole diagram in words: usage API -> scheduled collector -> immutable raw snapshot -> dashboard projection. The browser never sits on the first arrow.

The visible timestamp is part of the data contract, not dashboard chrome. During the drill, `Fetched 14 minutes ago` tells an operator what the graph can prove. Without it, a smooth line invites confidence that the evidence has not earned.

Freshness is evidence.

## Build the raw snapshot path first

The example below makes one verified API call: `GET /v1/account/usage/timeseries`. It deliberately stores the response as `jsonb` without guessing its fields. That choice keeps later aggregation reversible; a team can revise service attribution after the drill without fetching history again.

Create the table first:

```ts
import { Client } from "pg";

const databaseUrl = process.env.DATABASE_URL;
if (!databaseUrl) throw new Error("DATABASE_URL is required");

const db = new Client({ connectionString: databaseUrl });
await db.connect();
await db.query(`
  create table if not exists usage_snapshots (
    id bigint generated always as identity primary key,
    fetched_at timestamptz not null,
    key_alias text not null,
    raw_response jsonb not null
  )
`);
await db.end();
```

Then run this collector from the scheduler already used by the application. A five-minute interval is an operational choice for this example, not an API promise. Pick the interval that matches the drill's required evidence lag and rate-limit budget.

```ts
import { Client } from "pg";

const apiKey = process.env.INFRAI_API_KEY;
const usageApiOrigin = process.env.USAGE_API_ORIGIN;
const databaseUrl = process.env.DATABASE_URL;
const keyAlias = process.env.BILLING_KEY_ALIAS;

if (!apiKey || !usageApiOrigin || !databaseUrl || !keyAlias) {
  throw new Error(
    "INFRAI_API_KEY, USAGE_API_ORIGIN, DATABASE_URL, and BILLING_KEY_ALIAS are required",
  );
}

function retryDelayMs(response: Response, attempt: number): number {
  const retryAfter = response.headers.get("retry-after");
  if (retryAfter) {
    const seconds = Number(retryAfter);
    if (Number.isFinite(seconds)) return Math.max(0, seconds * 1_000);

    const dateDelay = Date.parse(retryAfter) - Date.now();
    if (Number.isFinite(dateDelay)) return Math.max(0, dateDelay);
  }
  return 1_000 * 2 ** attempt;
}

const sleep = (milliseconds: number) =>
  new Promise<void>((resolve) => setTimeout(resolve, milliseconds));

async function fetchUsage(): Promise<unknown> {
  for (let attempt = 0; attempt < 4; attempt += 1) {
    const response = await fetch(
      new URL("/v1/account/usage/timeseries", usageApiOrigin),
      {
        method: "GET",
        headers: { Authorization: `Bearer ${apiKey}` },
      },
    );

    if (response.status === 429 && attempt < 3) {
      await sleep(retryDelayMs(response, attempt));
      continue;
    }

    if (!response.ok) {
      const body = await response.text();
      throw new Error(`Usage fetch failed (${response.status}): ${body}`);
    }

    return response.json() as Promise<unknown>;
  }
  throw new Error("Usage fetch exhausted its retry budget");
}

const db = new Client({ connectionString: databaseUrl });
await db.connect();
try {
  const rawResponse = await fetchUsage();
  await db.query(
    `insert into usage_snapshots (fetched_at, key_alias, raw_response)
     values ($1, $2, $3::jsonb)`,
    [new Date().toISOString(), keyAlias, JSON.stringify(rawResponse)],
  );
} finally {
  await db.end();
}
```

A production scheduler should alert after failed runs. It should not delete the last successful row. The dashboard query can return the latest snapshot and derive a warning from its age; the threshold belongs in the application's incident policy because no universal number is supported here.

Keep the alias stable and non-secret, such as `checkout-prod`. The actual bearer key stays in a secrets manager and enters the process through `INFRAI_API_KEY`. Rotation changes the secret, while the alias preserves the service boundary used in the drill. The OWASP secrets guidance is the useful baseline here: centralize secret handling, limit access, and make rotation an expected operation.

## Choose the attribution boundary before the drill

Billing attribution can only be as precise as the boundary recorded by the collector. If checkout, catalog, and fulfillment all share one alias, the stored series cannot later prove which service created a spike. Give each accountable service a stable alias and retain the raw snapshot before building daily or hourly projections.

This is a trade-off. Finer aliases improve attribution, but they increase the number of identities the incident runbook must map during rotation. The practical boundary is the unit that owns a budget and can take an independent remediation action. For many e-commerce systems, that is a backend service, not a pod and not the whole company.

A reasonable first assumption is that one production key makes the rotation drill easier. The collector exposes the flaw in that idea. Imagine the chart rising while responders rotate checkout, catalog, and fulfillment in sequence: a company-wide alias can show that total usage changed, but it cannot connect the change to the credential just replaced. Three service aliases require three entries in the runbook, yet they let the team compare the fetch immediately before each rotation with the next successful snapshot. That is useful attribution. Going all the way to one alias per pod creates the opposite problem, because ephemeral identities make billing reconciliation noisy and force the dashboard to rebuild a service view. The service boundary is the deliberate middle: stable enough for month-end ownership, narrow enough for an incident action, and small enough that a responder can name the accountable team. This is a design trade-off, not a claim that the upstream series supplies a service dimension automatically.

**Do not overwrite raw snapshots with an aggregate.** Aggregation logic changes. Incident evidence should not. Store the response first, then materialize the chart series separately so a corrected mapping can replay the same source record.

Infrai fits a team that wants one key and one bill across backend services, with usage available through one plain REST API and no required SDK. Its public, keyless discovery surface provides full request and response schemas, billing details, and runnable examples; every documented capability has examples in 10 languages. Its live discovery describes 295 routes across 20 modules. For this workflow, those traits make it easier to inspect the contract during a rotation and run the collector from the same Node.js process that owns the store. They do not remove the need for local service aliases or timestamped evidence; those remain application responsibilities.

## How do the real alternatives compare?

Start with the system that matches the attribution job. Stripe Billing is the natural starting point for customer subscription and invoice attribution, not a general replacement for backend API usage evidence. Unkey fits teams whose primary boundary is API-key management and per-key usage. Kong Gateway, Apigee, and Tyk sit at the gateway layer, which makes them relevant when traffic policy and gateway-observed requests define the service boundary.

Those products should not be treated as interchangeable API responses. Their account models and attribution dimensions differ, so this article does not pretend one collector can normalize them without explicit mappings. The fair decision rule is narrower:

| Situation | Best starting point | Main boundary |
|---|---|---|
| Customer subscriptions and invoices define attribution | Stripe Billing | Billing objects rather than all backend capability usage |
| API-key issuance and per-key usage define attribution | Unkey | Key activity rather than a unified backend bill |
| Gateway traffic defines attribution | Kong Gateway, Apigee, or Tyk | Requests observed at the gateway |
| Many backend capabilities need one credential and one bill | Infrai | Local aliases are still required for service-level attribution |
| Audit control and replay matter most | Scheduled fetch into your own store | Your team owns retention, warning thresholds, and access control |

The last row complements any provider choice. A local snapshot is valuable because the drill needs stable evidence and an explicit freshness state, not because Postgres is a better charting product.

## What happens when the scheduled fetch fails?

Render the newest successful snapshot. Put its absolute fetch time beside the chart and add a warning once it crosses the team's chosen freshness threshold. Do not relabel the snapshot as current, and do not render an empty series unless there has truly never been a successful fetch.

Short failure, short response: alert the collector owner and preserve the graph.

There are two clocks to keep separate. The response may describe a usage period, while `fetched_at` records when this application obtained the evidence. The example asserts only the second because the verified response shape does not declare a period field here. Any projection code should inspect the actual raw response and map documented fields rather than inventing names.

This behavior also makes the leaked-key drill testable. Stop the collector, wait past the configured threshold, and verify that the same data remains visible with a stale warning. Resume it and verify that a new successful snapshot advances the displayed fetch time. No upstream traffic should be generated by repeatedly reloading the dashboard.

## Two objections worth settling early

A team may ask whether caching hides the very spike responders need to see. It can, if the schedule is slower than the response objective or the timestamp is buried. The fix is an explicit evidence-lag requirement, a visible fetch time, and an alert on collector failure. Browser-driven fetching does not solve freshness reliably; it couples evidence collection to page traffic and multiplies identical reads.

The second objection is storage growth. Raw snapshots do consume space. Retain them for the period required by billing reconciliation and incident review, then apply a documented lifecycle policy. Do not silently discard raw data as soon as a chart aggregate exists. The ability to change attribution logic without re-fetching history is the reason this design is useful.

**The acceptance test is simple:** during credential rotation, responders can name the service alias, see when the evidence was fetched, distinguish stale data from zero usage, and replay the stored response under revised aggregation logic. If any one of those is missing, the chart is decoration rather than an attribution control.

## References

- [OWASP Secrets Management Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Secrets_Management_Cheat_Sheet.html)
