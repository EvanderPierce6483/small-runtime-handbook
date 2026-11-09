# How to Suppress International SMS Alerts in Node.js — Marketplace Evidence Before Retries

| Evidence available | Pick this policy when | Dispatch decision |
| --- | --- | --- |
| An authenticated event explicitly identifies an invalid destination | The event can be tied to the recipient and channel | Suppress that destination and retain the reason |
| An authenticated event identifies a temporary delivery condition | Another attempt is allowed by the marketplace's policy | Schedule a bounded retry |
| A failure is ambiguous, unverified, or unmapped | The evidence cannot justify either classification | Hold for review |

International reach creates a tempting trade-off: retry more aggressively to improve alert delivery, or stop earlier to avoid repeatedly contacting invalid marketplace recipients. Resolve it with evidence. **Short answer:** separate confirmed invalid destinations from temporary and unknown failures, retain the reason for every suppression, and check that decision before dispatching another SMS.

For a marketplace sending seller alerts, the useful starting point for a Plivo versus Sinch pricing and international scalability comparison is the same acceptance exercise: can the team explain why a particular seller was contacted, retried, or suppressed? This note does not establish either provider's coverage, throughput, or current charges. Those comparisons need destination-specific evidence. It implements the recipient decision that an evaluation can inspect, using a deliberately small local example.

## 1. What evidence should stop international SMS alerts?

Pick permanent suppression when a trusted event explicitly supports an invalid-destination decision. Keep the channel in the key. An email bounce should not silently become a phone-number suppression, and a failed phone destination should not disable every communication attached to a seller account. This is a proposed marketplace policy, not a claim that one provider exposes a particular status or that one retention period satisfies every jurisdiction.

Pick bounded retries when the normalized evidence says the condition is temporary and your messaging policy still permits another attempt. A retry budget is an operational choice. The example below uses 2 retries so its boundary is easy to test; that number is illustrative, not a carrier requirement. Recheck eligibility when the retry actually runs.

Pick a review hold when the evidence is unknown. That choice costs delivery opportunities and adds review work, but it avoids inventing a permanent invalid-recipient finding from an unfamiliar error. A rejected submission, a delivery report, and proof that a human read a message are different questions. Define what your evidence supports before naming a normalized outcome.

Keep uncertainty visible.

Diagram in words: authenticated receipt → normalized evidence → recorded decision → recipient eligibility → dispatch. The normalization boundary owns the interpretation of external codes. The dispatch boundary owns enforcement. If the two disagree, the evidence record should make that disagreement inspectable.

## 2. Record a decision before changing eligibility

Use an append-only decision history and a separate current eligibility view in the production design. The history answers why a seller's destination was blocked; the current view answers whether a worker may send now. Combining those concerns into one overwritten status loses the earlier reasons. The code below models both in memory so the decision rule stays readable. It is a local executable example, with synthetic seller identifiers and no provider calls.

Save the TypeScript snippets together as `recipient-policy.ts` and run them with your project's TypeScript runner. The event contract is defined here. It is not an external webhook schema. Only pass events to this function after your ingress has authenticated the request, checked its shape, and established its marketplace scope.

```ts
type Channel = "sms" | "email";
type Outcome = "invalid" | "temporary" | "unknown";
type Decision = "suppress" | "retry" | "review";

type Evidence = {
  eventId: string;
  marketplaceId: string;
  recipientId: string;
  channel: Channel;
  outcome: Outcome;
  reason: string;
  receivedAt: string;
};

type RecordEntry = Evidence & { decision: Decision };
const history: RecordEntry[] = [];
const seen = new Map<string, RecordEntry>();
const eligibility = new Map<string, "suppressed" | "held">();

function recipientKey(e: Evidence): string {
  return JSON.stringify([e.marketplaceId, e.recipientId, e.channel]);
}

function recordEvidence(e: Evidence): RecordEntry {
  const eventKey = JSON.stringify([e.marketplaceId, e.eventId]);
  const prior = seen.get(eventKey);
  if (prior) {
    if (JSON.stringify(prior) !== JSON.stringify({ ...e, decision: prior.decision })) {
      throw new Error("Event identity reused with different evidence");
    }
    return prior;
  }

  const decision: Decision = e.outcome === "invalid"
    ? "suppress"
    : e.outcome === "temporary" ? "retry" : "review";
  const entry: RecordEntry = { ...e, decision };
  history.push(entry);
  seen.set(eventKey, entry);

  const key = recipientKey(e);
  if (decision === "suppress") eligibility.set(key, "suppressed");
  if (decision === "review" && !eligibility.has(key)) {
    eligibility.set(key, "held");
  }
  return entry;
}

function canDispatch(e: Evidence, retriesUsed: number): boolean {
  if (!Number.isInteger(retriesUsed) || retriesUsed < 0) return false;
  return !eligibility.has(recipientKey(e)) && retriesUsed < 2;
}
```

Notice the asymmetry: recording a temporary failure never clears a suppression or review hold. A later-arriving event should not reopen a destination merely because it looks less severe. Reinstatement needs a separate, authorized decision with its own evidence; this example does not implement that workflow. Consider the synthetic seller fixture below: `receipt-002` establishes invalidity, and `receipt-003` carries a temporary outcome for the same destination. Treating the last event received as the complete current truth would clear the block without evidence that the destination became valid. The safer rule here is monotonic suppression until an explicit reinstatement decision, with the temporary event retained in history so an investigator can still reconstruct what arrived.

Order isn't proof.

The event identity is scoped to the marketplace. Production normalization should also incorporate the source account or namespace if event identifiers are only unique within that source. Here, exact duplicate evidence returns the existing record, while a reused identity with changed evidence throws. Avoid acknowledging that conflict as a successful new decision.

## 3. Prove the retry boundary with seller alerts

Append this fixture to the same file. All identifiers are synthetic. The assertions exercise the policy rather than any supplier's delivery behavior. In particular, a late temporary event must leave an earlier suppression in place.

```ts
function check(condition: boolean, message: string): void {
  if (!condition) throw new Error(message);
}

const sellerAlert: Evidence = {
  eventId: "receipt-001",
  marketplaceId: "market-demo",
  recipientId: "seller-042-phone-1",
  channel: "sms",
  outcome: "temporary",
  reason: "Synthetic temporary delivery condition",
  receivedAt: "2026-11-07T08:00:00Z",
};

recordEvidence(sellerAlert);
recordEvidence(sellerAlert);
check(history.length === 1, "Duplicate event added another record");
check(canDispatch(sellerAlert, 1), "First retry should be eligible");
check(!canDispatch(sellerAlert, 2), "Retry budget was exceeded");

recordEvidence({
  ...sellerAlert, eventId: "receipt-002", outcome: "invalid",
  reason: "Synthetic confirmed invalid destination",
});
recordEvidence({ ...sellerAlert, eventId: "receipt-003" });
check(!canDispatch(sellerAlert, 0), "Late event reopened suppression");

const unknownSeller = {
  ...sellerAlert, eventId: "receipt-004",
  recipientId: "seller-043-phone-1", outcome: "unknown" as const,
  reason: "Synthetic unmapped result",
};
recordEvidence(unknownSeller);
check(!canDispatch(unknownSeller, 0), "Unknown evidence bypassed review");
console.log("Recipient policy checks passed");
```

The retry function only evaluates this example's recipient state and budget. A real dispatcher must also check consent, message purpose, destination rules, and any applicable sending restrictions.

Don't confuse eligibility with permission.

For an international SMS evaluation, run the same acceptance cases against each candidate's documented event model. Ask what evidence distinguishes invalid destinations from temporary delivery conditions in each intended market, how event authentication works, and how ambiguous outcomes surface. Capture the documented answer and a reproducible test result. No destination coverage or comparative performance follows from the local fixture.

Treat pricing as an input to that exercise. Specify the intended destinations, message content, traffic shape, and retry policy before requesting comparable terms. Without those assumptions, an advertised unit figure cannot answer this marketplace's operational question. This note makes no price or savings claim.

## 4. Make the evidence useful during operations

A suppression counter alone cannot explain which policy produced it. Emit a structured decision log that points back to the retained evidence, then count outcomes by channel and decision. Keep recipient identifiers out of metric labels. They belong in access-controlled records where an investigator can follow a particular case without turning every destination into a separate metric series.

Append this small log projection to inspect the fixture's decisions. It excludes recipient identifiers and free-form reasons from the operational log; those remain in the illustrative history.

```ts
for (const entry of history) {
  console.log(JSON.stringify({
    event: "recipient_policy_decision",
    marketplaceId: entry.marketplaceId,
    evidenceId: entry.eventId,
    channel: entry.channel,
    outcome: entry.outcome,
    decision: entry.decision,
    receivedAt: entry.receivedAt,
  }));
}
```

Before deployment, replay representative evidence through the classifier and inspect the resulting decisions with the people responsible for messaging policy. During rollout, compare the proposed decisions with current dispatch eligibility before enforcing new suppressions. Define alerts for growing review holds, missing decision records, and attempted sends to suppressed recipients. Set thresholds from the marketplace's traffic and response capacity; the fixture cannot supply a meaningful production threshold.

There is a storage trade-off here. Keeping evidence makes decisions explainable, while retaining unnecessary destination data expands the data the team must protect. Decide which fields are required, who can access them, and how retention and deletion apply before treating the history as a compliance artifact. An append-only application interface by itself does not prove legal compliance or tamper resistance.

## 5. Limits to settle before enforcing suppression

Memory disappears on restart. Replace the maps with durable storage, make duplicate detection and decision updates transactional, and serialize conflicting updates for the same destination. Test a worker racing with a new suppression. A check performed when an alert was queued can become stale before the actual send; closing that gap needs a coordinated dispatch design.

Also scope phone destinations independently from seller accounts. Replacing a seller's phone number should create a distinct destination identity and an explicit eligibility decision, rather than accidentally inheriting or clearing an unrelated suppression. Keep manual reinstatement auditable.

Finally, email documentation cannot establish SMS delivery semantics, international availability, or comparative pricing. The email references below provide channel-specific background; external SMS status mappings still need their own verified evidence. The useful finish line is modest: given a seller alert, the team can reproduce why another attempt was allowed, held, or blocked.

## Sources

References:

- https://www.rfc-editor.org/rfc/rfc3463 — Enhanced mail system status codes; email-specific context for distinguishing failure classes.
- https://www.rfc-editor.org/rfc/rfc3464 — Delivery status notifications for email; background for retaining delivery evidence.
- https://resend.com/docs/introduction — Email documentation; does not substantiate an SMS provider comparison.
