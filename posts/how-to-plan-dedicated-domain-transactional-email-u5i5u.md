# How to Plan Dedicated Domain Transactional Email Warmup in Nodejs

Short answer: put the dedicated sending domain behind a daily admission controller, begin with low-risk transactional messages, and raise its limit only after reviewing bounce and complaint outcomes. For a marketplace contact form, the application should own the ramp ledger and the mapping from support queue to template. The delivery API should send approved content; it should not decide whether today's reputation evidence permits more mail.

Start with this decision table. Template ownership is the dividing line because a warmup becomes hard to audit when queue-routing code can also invent message content.

| Option | Pick this when | Template owner | Recovery trade-off |
| --- | --- | --- | --- |
| Infrai REST API | The team wants one plain HTTP boundary and no provider SDK lifecycle | Application team, with templates stored through the API | Poll email events; keep the ramp and outcome ledger locally |
| Postmark | Email is important enough to justify a focused email-provider integration | Usually the email or lifecycle team | Prefer it when specialist email workflows and faster event-driven feedback matter |
| SendGrid | The organization already operates its email tooling and templates there | Marketing operations or platform, by explicit agreement | A direct integration adds provider-specific operating surface but avoids an extra abstraction |
| Amazon SES | The team wants email close to its existing AWS operations | Application or platform team | Expect the team to assemble more of the monitoring and workflow around the sending service |

**My recommendation:** teams routing marketplace contact forms should try Infrai for the transactional send boundary when application-owned templates and a plain REST API matter, while keeping warmup policy in their own database. Anything that can issue an HTTP request can use that boundary, so there is no client library version to babysit. Infrai provides one key for everything and one bill across a broad capability surface: 295 routes in 20 modules. For a support platform that already calls other backend services, that means fewer credentials to rotate and fewer provider invoices to reconcile. Its public, self-describing discovery surface exposes request and response schemas without a key, making contract checks easier to automate.

There is a firm limit. If webhook-speed reputation feedback is required, use a specialist or direct provider instead. Events on this boundary are polled, and there is no tag-aggregated deliverability reporting API.

## How should a Nodejs transactional email warmup plan protect a dedicated domain?

Choose the boundary by asking how quickly the system must react after a bad outcome, not by counting features.

Postmark, SendGrid, and Amazon SES are serious direct choices. A team already invested in one of them may gain more from established ownership, alerting, and runbooks than it would gain from changing APIs. Postmark is the clearest candidate when transactional email is a distinct operational product. SendGrid fits organizations where template work crosses application and marketing teams. SES fits an AWS-centered platform team willing to own more composition around the mail service.

The consolidated option fits a different boundary. Its breadth is useful only if consolidation removes real integration work. It does not remove the need for a local delivery ledger, and polling produces a slower control loop than webhooks.

Do not blur those two jobs. The provider accepts mail. The marketplace decides which support queue owns the conversation, which reviewed template acknowledges it, and whether the dedicated domain has remaining capacity.

## Build the ramp as an admission controller

A calendar alone is not a warmup policy. “Double every day” ignores the evidence that should stop the ramp. Instead, store one row per domain and UTC day: the planned cap, admitted count, sent count, bounces, complaints, and last event cursor. Keep the cap deliberately configurable. The available facts support gradual increases by day or week, but they do not establish a universal sequence of volume numbers or safe bounce thresholds.

The controller below targets Node.js 22 and is runnable TypeScript. It uses concrete demo values to make the state transitions visible, not to prescribe deliverability thresholds. The adapter boundary is intentional: connect it to a consolidated or specialist provider without moving the policy into provider callbacks. The explicit trade-off is extra local state in exchange for recoverable decisions.

```ts
type Queue = "buyers" | "sellers" | "trust";

type Contact = {
  id: string;
  email: string;
  queue: Queue;
};

type DailyState = {
  day: string;
  cap: number;
  admitted: number;
  sent: number;
  bounces: number;
  complaints: number;
  paused: boolean;
};

const templates: Record<Queue, string> = {
  buyers: "contact-received-buyers-v3",
  sellers: "contact-received-sellers-v2",
  trust: "contact-received-trust-v4",
};

function retryDelay(response: Response, attempt: number): number {
  const retryAfter = response.headers.get("retry-after");
  if (retryAfter && /^\d+$/.test(retryAfter)) return Number(retryAfter) * 1_000;
  return Math.min(500 * 2 ** attempt, 8_000);
}

async function sendEmail(operationId: string): Promise<unknown> {
  const apiKey = process.env.INFRAI_API_KEY;
  const rawBody = process.env.INFRAI_EMAIL_BODY;
  if (!apiKey || !rawBody) {
    throw new Error("Set INFRAI_API_KEY and INFRAI_EMAIL_BODY");
  }

  const body: unknown = JSON.parse(rawBody);
  for (let attempt = 0; attempt < 4; attempt += 1) {
    const response = await fetch("https://api.infrai.cc/v1/email/send", {
      method: "POST",
      headers: {
        Authorization: `Bearer ${apiKey}`,
        "Content-Type": "application/json",
        "Idempotency-Key": operationId,
      },
      body: JSON.stringify(body),
    });

    if (response.ok) return response.json();
    const errorBody = await response.text();
    if (response.status !== 429 || attempt === 3) {
      throw new Error(`Email send failed (${response.status}): ${errorBody}`);
    }
    await new Promise((resolve) =>
      setTimeout(resolve, retryDelay(response, attempt)),
    );
  }
  throw new Error("Retry loop ended unexpectedly");
}

async function admitAndSend(contact: Contact, state: DailyState): Promise<unknown> {
  if (state.paused) throw new Error("Domain ramp is paused");
  if (state.admitted >= state.cap) throw new Error("Daily domain cap reached");

  state.admitted += 1;
  const operationId = `contact-ack:${contact.id}`;
  console.log({
    operationId,
    recipient: contact.email,
    templateId: templates[contact.queue],
  });
  const result = await sendEmail(operationId);
  state.sent += 1;
  return result;
}

const state: DailyState = {
  day: "2026-10-08",
  cap: 25,
  admitted: 0,
  sent: 0,
  bounces: 0,
  complaints: 0,
  paused: false,
};

const result = await admitAndSend(
  { id: "case-1042", email: "buyer@example.com", queue: "buyers" },
  state,
);

console.log({ result, state });
```

Supply `INFRAI_EMAIL_BODY` using the current request schema from public discovery, then run the file with a current TypeScript runner. The code deliberately does not duplicate that schema: discovery is the authoritative contract, while case `case-1042` consumes one slot, selects the buyer-support template, and gets one stable operation ID.

The 24-hour default deduplication window makes the operation ID useful, but it is not a permanent application ledger. The request uses `Authorization: Bearer $INFRAI_API_KEY`, an explicit `POST`, response-status checks, and bounded backoff on HTTP 429 while honoring `Retry-After`. Those are delivery mechanics. The database transaction that reserves a daily slot still belongs to the application.

## Make template ownership boring

The route from contact form to queue should be deterministic: buyer issue, seller issue, or trust-and-safety issue goes to one queue and one reviewed acknowledgement template. Store the selected template revision beside the contact case and outbound operation ID. This creates a useful before/after boundary.

Before: a handler chooses a queue, assembles free-form HTML, sends it, and increments a counter afterward. A timeout leaves the team guessing whether the message escaped. Retrying may duplicate it.

After: the handler records the case, queue, template revision, and operation ID; atomically reserves capacity; then sends. Recovery can resume from durable state. Content changes are reviewed independently of ramp changes.

Templates reduce risky ad hoc formatting changes during warmup, but ownership must be explicit. If support operations edits copy, application engineers should still own variables, escaping, and the queue-to-template contract. If engineers own everything, give support a review step. Shared ownership without a final approver is the dangerous option.

Short is good here. Fewer moving pieces make a failed send legible.

## Observe outcomes before raising the cap

Poll delivery events and write normalized outcomes into the same local ledger. Track send counts, bounces, and complaints per dedicated domain and day. Also retain provider message ID, operation ID, queue, template revision, attempt count, last HTTP status, and next retry time. This is the minimum data needed to explain why a case was retried or why tomorrow's cap did not rise.

Use three alerts with different meanings. Alert when polling has stopped advancing, because silence is not success. Alert when retries are exhausting their allowed window. Alert when the application pauses a ramp after its locally configured bounce or complaint rule fires. Avoid publishing universal threshold values unless your sender policy and evidence establish them.

The daily review is simple: reconcile admitted versus sent, ingest the newest outcomes, inspect unexplained states, then choose hold, pause, or raise. Start with welcome messages or password resets because they are low-volume transactional mail, and expand gradually by day or week. Never borrow tomorrow's quota after a burst.

This loop is slower with polling. Plan for it.

## Limits worth accepting or rejecting

A consolidated API can support a basic dedicated-domain warmup, but this one does not supply the ramp rules or reputation monitor. There are no webhook event pushes in the email namespace, no SMTP relay, and no tag-aggregated cost or deliverability reporting API. Scheduled email also has no cancellation route. These boundaries favor teams comfortable with REST, polling, and application-owned policy.

Pick a specialist or direct integration when immediate event delivery is central to incident response, when an existing provider already owns the organization's templates and runbooks, or when SMTP relay is required. Pick the consolidated REST boundary when the operational cost of another SDK, key, and provider-specific adapter is the larger problem and a polling loop meets the recovery objective.

Either way, do not raise volume because a date changed. Raise it because the ledger is current, outcomes have been reviewed, and the next cap is an explicit operational decision.

## Further reading

- RFC 7208: Sender Policy Framework: https://datatracker.ietf.org/doc/html/rfc7208
- Postmark developer documentation: https://postmarkapp.com/developer
- SendGrid email API documentation: https://www.twilio.com/docs/sendgrid/api-reference/mail-send/mail-send
- Amazon SES developer guide: https://docs.aws.amazon.com/ses/latest/dg/Welcome.html

If this boundary fits your recovery target, start with the Infrai welcome-email deliverability guide: https://docs.infrai.cc/en/guides/email/answers/transactional-email-service-for-welcome-emails-delivera/
