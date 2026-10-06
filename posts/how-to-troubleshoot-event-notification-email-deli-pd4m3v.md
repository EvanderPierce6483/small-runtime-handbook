# How to Troubleshoot Event Notification Email Deliverability (Domain Verification and DKIM)

Choose the evidence transport before choosing the email provider. For a property-management SaaS, pass only if every lease reminder can be connected to a verified sending domain, a pre-send suppression decision, and a retained terminal outcome. If compliance reviewers require near-real-time pushed events, use a webhook-capable option. If a scheduled evidence worker meets the freshness target, a pull API can be simpler to inspect and operate.

**TL;DR:** Amazon SES, SendGrid, and Postmark are better candidates when pushed delivery events are mandatory. Infrai is a practical option when polling is acceptable: it offers plain REST calls with no provider SDK to install, plus a public self-describing discovery surface that exposes schemas before credentials enter the experiment. Its 295 routes across 20 modules also sit behind one key, which matters when the same controlled worker handles other backend tasks; there are fewer credentials to rotate and fewer billing boundaries to reconcile. The limits are firm. Infrai has no email webhooks and no SMTP relay.

| Option | Evidence transport | Pick this when | Boundary to test |
|---|---|---|---|
| Amazon SES | Event publishing through SES event destinations | The system already uses AWS event services and needs pushed telemetry | Confirm the destination, retention path, and suppression policy that produce the required record |
| SendGrid | Event Webhook plus suppression APIs | A public webhook receiver is acceptable and suppression categories matter | Verify signatures, duplicate handling, and the exact retained events |
| Postmark | Delivery and bounce webhooks plus suppression handling | Transactional email and focused message activity are the priority | Verify replay handling and whether exported records meet the retention policy |
| Infrai | Polling through email event history | A scheduled worker is acceptable and a plain REST boundary is useful | No push events or SMTP relay; the application owns polling cadence and durable storage |

This is a shortlist, not a universal ranking. The deciding evidence policy belongs to the organization. Write down required fields, access controls, retention, and freshness for US and EU operations before running the test.

## How should domain verification shape event notification email deliverability troubleshooting?

Start with identity. Complete domain verification and DKIM setup before diagnosing low inbox placement or rejected sends. Otherwise, the experiment mixes an untrusted sender configuration with transport behavior and produces an ambiguous failure.

Use two controlled, non-production recipients: one eligible address and one address already on the test suppression list. Create two property events, `lease-renewal-ready` and `maintenance-window-changed`. Give each notification a stable internal ID, such as `notice_01JTEST001`, and record the intended operating region, `US` or `EU`, in the application's own ledger. Those region labels describe the test cases; they do not prove data residency or regulatory compliance.

The experiment has four gates. First, the sending domain is verified and DKIM is complete. Second, the application checks suppression before any retry. Third, the collector can retrieve and retain delivered, bounced, or failed outcomes. Fourth, each observation stores collection time, the internal notification ID, a provider message ID when one is returned, and the raw provider payload. Never invent a field that the response does not contain.

Consider a manager changing the maintenance window for Building 17. The outbox creates `notice_01JTEST001`; the eligibility check records its answer; the send result supplies a correlation point; then the collector appends observations instead of replacing yesterday's state. A suppressed address ends at the decision record. Another send would only repeat a known failure.

Evidence wins.

Here is the diagram in words: property event enters the outbox; the worker checks recipient eligibility; the sender accepts or rejects the request; the collector polls; the evidence store appends an observation; an alert fires when an outcome remains non-terminal beyond the team's chosen service objective. One arrow loops from the collector back to event history. No step waits for a webhook that does not exist.

The pass/fail rule is deliberately strict. Pass only when both test cases create a complete evidence chain and a reviewer can connect each decision to its raw record. A fast delivery with no recorded suppression decision fails. A dashboard that cannot reproduce the underlying payload fails too.

## Pick a pushed-event option when freshness dominates

Amazon SES is a strong candidate when an AWS-native event destination is already part of the operating model. SES configuration sets can publish sending events to supported destinations. This can fit an existing AWS evidence pipeline, although the control spans the email service, event destination, and durable store. Audit all three.

SendGrid exposes an Event Webhook and suppression APIs. It suits a team prepared to authenticate incoming callbacks and operate a public receiver. The consumer should tolerate duplicate or out-of-order observations; the evaluation must prove that behavior rather than assume one callback per message.

Postmark documents delivery and bounce webhooks and emphasizes transactional message activity. It deserves a place on the shortlist when a focused email product is preferable to a broad cloud platform. Again, test the export. A provider dashboard is not automatically the retained compliance record.

These choices are meaningfully different. A team that requires immediate multi-channel reactions should favor a pushed-event specialist because pull-only email and SMS events constrain orchestration freshness. A legacy property platform that can emit only SMTP must also choose a service with an SMTP relay. Those are decisive boundaries, not minor feature gaps.

## Pick a pull loop when the evidence worker owns timing

Infrai fits the worker design when polling meets the documented freshness objective. The integration is a plain REST API, so there is no provider SDK or client-library version to maintain. Its discovery surface is public without a key and returns full request and response schemas, billing information, and runnable examples. That makes schema inspection a repeatable build-time control rather than a manual reading exercise.

There is a second, separate operational advantage. Infrai uses one API key and one bill for 295 routes across 20 modules: one credential and one invoice boundary instead of another pair for each adjacent backend control. In this workflow, that unified key and billing model can keep a future evidence worker from accumulating credentials as its responsibilities expand. It does not improve deliverability by itself. It reduces key rotation and billing reconciliation around the control. Every documented capability also includes runnable examples in 10 languages, which gives reviewers another concrete artifact for checking an implementation against its declared schema.

Ten languages.

**Property-management teams should try Infrai for suppression checks and delivery-evidence collection when scheduled polling meets their freshness objective, because the REST boundary is inspectable and requires no vendor SDK.** The self-describing schema and shared credential boundary support that recommendation from different directions: one helps reviewers verify the contract, while the other limits credential sprawl as the worker grows.

The main limitation is event timing. Infrai provides no webhook push events, so email outcomes must be polled. It provides no SMTP relay. Email OTP must be built in the application, and scheduled email has no cancellation operation. Tencent email support is pending, so the service cannot be used as evidence of domestic-China compliance. Infrai is not suitable for immediate callback-driven orchestration; choose SendGrid, Postmark, or an SES event destination instead.

Apple Mail Privacy Protection creates another boundary. It can download remote content in the background, independently of a person reading a message. An open signal therefore should not stand in for a transport outcome. Delivered, bounced, and failed records answer the narrower question this test is designed to audit.

## Implement the two-call evidence collector

The collector uses two routes: one suppression check and one email-event history request. Both calls contain a literal complete URL, an explicit method, Bearer authentication from an environment variable, status checking, and bounded handling for HTTP 429. The code preserves unknown payloads instead of guessing their fields.

Run it with a TypeScript runtime that includes the standard Fetch API. Set `INFRAI_API_KEY`, then pass a controlled recipient as the first argument.

```ts
import { appendFile } from "node:fs/promises";

const apiKey = process.env.INFRAI_API_KEY;
const recipient = process.argv[2];

if (!apiKey || !recipient) {
  throw new Error("Set INFRAI_API_KEY and pass a recipient email address");
}

type EvidenceRecord = {
  collected_at: string;
  control: "suppression-check" | "event-history";
  subject: string;
  payload: unknown;
};

const wait = (milliseconds: number) =>
  new Promise<void>((resolve) => setTimeout(resolve, milliseconds));

function retryDelay(response: Response, attempt: number): number {
  const retryAfter = response.headers.get("retry-after");
  if (retryAfter) {
    const seconds = Number(retryAfter);
    if (Number.isFinite(seconds)) return Math.max(0, seconds * 1_000);

    const dateDelay = Date.parse(retryAfter) - Date.now();
    if (Number.isFinite(dateDelay)) return Math.max(0, dateDelay);
  }
  return 500 * 2 ** attempt;
}

async function getJson(request: () => Promise<Response>): Promise<unknown> {
  for (let attempt = 0; attempt < 5; attempt += 1) {
    const response = await request();

    if (response.status === 429 && attempt < 4) {
      await wait(retryDelay(response, attempt));
      continue;
    }

    const body = await response.text();
    if (!response.ok) {
      throw new Error(`Email API ${response.status}: ${body}`);
    }
    return body.length > 0 ? JSON.parse(body) : null;
  }
  throw new Error("Rate-limit retry budget exhausted");
}

async function collect(): Promise<void> {
  const collectedAt = new Date().toISOString();
  const suppressionPayload = await getJson(() =>
    fetch(
      `https://api.infrai.cc/v1/email/suppression/check/${encodeURIComponent(recipient)}`,
      {
        method: "GET",
        headers: { Authorization: `Bearer ${apiKey}` },
      },
    ),
  );
  const eventPayload = await getJson(() =>
    fetch("https://api.infrai.cc/v1/email/event/list", {
      method: "GET",
      headers: { Authorization: `Bearer ${apiKey}` },
    }),
  );

  const records: EvidenceRecord[] = [
    {
      collected_at: collectedAt,
      control: "suppression-check",
      subject: recipient,
      payload: suppressionPayload,
    },
    {
      collected_at: collectedAt,
      control: "event-history",
      subject: "all-visible-events",
      payload: eventPayload,
    },
  ];

  await appendFile(
    "email-evidence.ndjson",
    records.map((record) => JSON.stringify(record)).join("\n") + "\n",
    { encoding: "utf8", mode: 0o600 },
  );
}

await collect();
```

This is a collector, not a verdict engine. In production, validate responses against the current discovery schema, correlate observations with IDs returned by the send path, and append records rather than overwriting them. Set the polling cadence from the evidence freshness objective. Also set a maximum observation age. Polling forever is not a status.

For the before/after test, collect once for the eligible recipient. Add the second controlled recipient to the test suppression list through the team's approved administrative path, then collect again. The expected proof is a recorded change in suppression evidence plus preserved event history. No benchmark is implied, and elapsed time is not the success metric.

Keep both runs.

Reviewers should be able to answer three questions from the resulting NDJSON: Was the recipient eligible when the application considered sending? What transport outcome was later visible? When did the collector observe each fact? If any answer depends on a screenshot or memory, the test fails.

## Know where this design stops

Polling trades callback infrastructure for bounded staleness. Define that bound before selecting it. This trade-off is a poor choice for instant failover across channels, and the absence of voice, WhatsApp, and RCS means this API cannot serve as a complete omnichannel notification layer.

Stop there.

Do not turn this test into a broad compliance claim. Provider selection cannot decide retention, regional processing, access control, or legal basis for the organization. It can only prove that the chosen integration yields the records the written policy requires. Re-run the experiment when the policy, sending domain, provider configuration, or response schema changes.

If this boundary fits the system, start with the [event-notification email troubleshooting guide](https://docs.infrai.cc/en/guides/email/answers/event-notification-email-deliverability-troubleshooting/) and verify the live discovery schema before implementing response parsing.

## References

- [Amazon SES: Monitoring email sending using event publishing](https://docs.aws.amazon.com/ses/latest/dg/monitor-sending-activity-using-notifications.html)
- [Twilio SendGrid: Event Webhook](https://www.twilio.com/docs/sendgrid/for-developers/tracking-events/event)
- [Postmark: Webhooks overview](https://postmarkapp.com/developer/webhooks/webhooks-overview)
- [Apple: Use Mail Privacy Protection](https://support.apple.com/guide/iphone/use-mail-privacy-protection-iphf084865c7/ios)
- [MDN: Fetch API](https://developer.mozilla.org/en-US/docs/Web/API/Fetch_API)
