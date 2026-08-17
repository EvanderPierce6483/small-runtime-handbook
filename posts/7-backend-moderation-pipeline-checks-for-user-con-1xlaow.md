# 7 Backend Moderation Pipeline Checks for User Content (with JSON Review Queues)

Short answer: for messy product descriptions, use a rules-first backend moderation pipeline that accepts a typed JSON decision from a chat-completion adapter, publishes only clear allows, rejects only explicit blocks, and sends every uncertain result to a durable human review queue. Keep the provider behind one interface. Portability comes from owning the decision contract, not from pretending every model behaves the same.

| Pick this path | Use it when | Main operational cost | Publish rule |
| --- | --- | --- | --- |
| Synchronous model check | Traffic is modest and a short write delay is acceptable | User-facing latency and timeout handling | Publish only after `allow` |
| Asynchronous hold and review | Catalog edits can remain pending | Queue age and reviewer capacity | Keep pending until resolved |
| Rules-first hybrid | Repeated spam patterns are easy to identify before model review | Two policy layers must stay aligned | Rules decide obvious cases; model triages the rest |
| Human-only review | Volume is low or policy nuance dominates | Reviewer time and inconsistent judgments | Publish only after a person decides |

For an e-commerce catalog, I would start with the rules-first hybrid and an asynchronous hold. It gives the team a clean before/after: an untrusted description enters as `pending`, deterministic checks remove obvious noise, structured model output assigns a route, and only then can publication state change. No model prose gets to flip that state directly.

## 1. How should a Node.js backend moderation pipeline route user-generated content?

Start with the consequence of a wrong decision. A false allow might publish a prohibited claim or abusive text. A false block can hide a legitimate product and frustrate a seller. A review decision adds delay but preserves an escape hatch. That makes `review` a real outcome, not a softer spelling of `allow`.

The synchronous path is attractive when a seller expects immediate feedback. The request stays open, the moderation decision arrives, and the API returns the new catalog state. The catch is that model latency becomes write latency. A transport timeout must leave the item pending; it must never quietly become approved. Use this path only when callers can retry with an idempotency key and the product can explain a pending result.

The asynchronous path is a better fit when imports arrive in batches or descriptions are enriched after ingestion. Save the original content, assign a content hash and policy version, then enqueue the moderation job. The public catalog continues serving the last approved revision while the new revision waits. This separates availability from moderation throughput — useful when a 50,000-row supplier file lands at once — but it creates a new promise: queue age must be measured and owned.

Human-only review remains sensible for a small, high-risk catalog. Don't add a model merely to make the diagram look current. Once volume creates a backlog, a hybrid can reserve human attention for ambiguous cases while deterministic rules catch exact deny-list matches, impossible field lengths, or missing required attributes.

Keep the state machine tiny: `pending -> allowed`, `pending -> blocked`, or `pending -> review -> allowed|blocked`. Reject every other transition. A retry may repeat a transition, but it may not invent one.

Pending means pending.

## 2. Which JSON fields should remain under application policy?

Provider portability starts at the application boundary. Define the smallest useful moderation result and validate it after every completion. The provider adapter may translate a native structured-output feature, a tool call, or plain JSON into this contract, but the catalog service receives the same object each time.

That boundary should carry policy meaning rather than provider vocabulary. `decision`, `categories`, `confidence`, `reasonCode`, `policyVersion`, and `contentHash` are enough for routing and audit. Free-form explanations are poor control inputs: wording changes, they are hard to aggregate, and they tempt downstream code to search for magic phrases. Keep a short optional note for reviewers if policy permits it, but route on enums.

Confidence needs special care. It isn't a universal probability. One adapter's `0.82` is not automatically comparable with another adapter's `0.82`, so store the adapter and model identifiers alongside the result and calibrate thresholds on labeled catalog examples. I'm not sure a threshold chosen on fashion descriptions will transfer to supplements or refurbished electronics; a stratified evaluation set would resolve that question.

Version the policy separately from the prompt. Otherwise, a policy edit and a model change become one muddy deployment. Short and explicit wins here.

The transport should also distinguish three classes of failure. Invalid input returns a stable client error such as `422`. A duplicate request returns the earlier job through its idempotency key. A completion that is missing, malformed, or outside the schema leaves the revision pending and records a transport outcome; it does not manufacture a moderation verdict.

## 3. Which adapter boundary should the human review queue trust?

The focused example below keeps networking outside the policy function. `ChatTransport` is the replaceable edge: each provider-specific adapter can call its supported chat-completion interface and return unknown data. Everything after that point is ordinary TypeScript, which makes the routing behavior testable without a live model.

```ts
type Decision = "allow" | "review" | "block";

type ModerationResult = {
  decision: Decision;
  categories: string[];
  confidence: number;
  reasonCode: string;
  policyVersion: string;
  contentHash: string;
};

type CatalogRevision = {
  id: string;
  title: string;
  description: string;
  contentHash: string;
};

type ChatTransport = {
  completeJson(input: {
    system: string;
    user: string;
    schema: object;
  }): Promise<unknown>;
};

type ReviewQueue = {
  add(job: {
    revisionId: string;
    reasonCode: string;
    policyVersion: string;
    contentHash: string;
  }): Promise<void>;
};

const moderationSchema = {
  type: "object",
  additionalProperties: false,
  required: [
    "decision",
    "categories",
    "confidence",
    "reasonCode",
    "policyVersion",
    "contentHash",
  ],
  properties: {
    decision: { enum: ["allow", "review", "block"] },
    categories: { type: "array", items: { type: "string" } },
    confidence: { type: "number", minimum: 0, maximum: 1 },
    reasonCode: { type: "string", minLength: 1 },
    policyVersion: { type: "string", const: "catalog-2026-01" },
    contentHash: { type: "string", minLength: 1 },
  },
} as const;

function isModerationResult(value: unknown): value is ModerationResult {
  if (typeof value !== "object" || value === null) return false;
  const item = value as Record<string, unknown>;
  return (
    ["allow", "review", "block"].includes(String(item.decision)) &&
    Array.isArray(item.categories) &&
    item.categories.every((entry) => typeof entry === "string") &&
    typeof item.confidence === "number" &&
    item.confidence >= 0 &&
    item.confidence <= 1 &&
    typeof item.reasonCode === "string" &&
    item.policyVersion === "catalog-2026-01" &&
    typeof item.contentHash === "string"
  );
}

async function moderateRevision(
  revision: CatalogRevision,
  transport: ChatTransport,
  queue: ReviewQueue,
): Promise<ModerationResult> {
  const raw = await transport.completeJson({
    system:
      "Classify catalog text. Return only the requested JSON. " +
      "Use review whenever the evidence is ambiguous.",
    user: JSON.stringify({
      title: revision.title,
      description: revision.description,
      contentHash: revision.contentHash,
      policyVersion: "catalog-2026-01",
    }),
    schema: moderationSchema,
  });

  if (!isModerationResult(raw) || raw.contentHash !== revision.contentHash) {
    const pending: ModerationResult = {
      decision: "review",
      categories: ["invalid_response"],
      confidence: 0,
      reasonCode: "SCHEMA_OR_HASH_MISMATCH",
      policyVersion: "catalog-2026-01",
      contentHash: revision.contentHash,
    };
    await queue.add({
      revisionId: revision.id,
      reasonCode: pending.reasonCode,
      policyVersion: pending.policyVersion,
      contentHash: pending.contentHash,
    });
    return pending;
  }

  if (raw.decision === "review") {
    await queue.add({
      revisionId: revision.id,
      reasonCode: raw.reasonCode,
      policyVersion: raw.policyVersion,
      contentHash: raw.contentHash,
    });
  }

  return raw;
}
```

There is one deliberate policy choice in the fallback: malformed output becomes `review`, never `allow`. The returned object is an internal routing decision, not a claim that the model classified the content. That distinction matters in audit logs.

Consider one supplier import with 50,000 rows. Row 18,204 describes a replacement battery, but its title belongs to the previous row because the supplier's export shifted one cell. The deterministic layer sees valid lengths and no exact deny-list match. The completion sees plausible text and returns structured JSON, yet the title-description conflict makes the result uncertain enough for review. Before queue insertion, another import corrects the title and creates a new content hash. A reviewer later opens the old job. If the workflow applies that old decision by product ID alone, a careful human can accidentally approve a revision they never saw. The hash comparison stops the transition, marks the job stale, and submits the current revision again. This is why the hash appears in the input, model response, queue message, log event, and publication guard. It is repetitive on purpose — each asynchronous boundary must prove which bytes were judged. The model provider is almost incidental to this failure mode; swapping it cannot repair weak revision identity.

In production, validate with a maintained JSON Schema implementation rather than letting the hand-written guard grow. The compact guard is shown so the control flow remains visible. Also escape nothing by string concatenation: serializing the seller fields as JSON keeps them data, while the fixed system instruction remains policy. Treat descriptions that say “ignore previous instructions” as catalog text, not as authority.

The review job contains IDs and decision metadata, not an uncontrolled copy of the description. A reviewer service can load the authorized revision under its normal access rules. This reduces duplicated sensitive text and prevents a stale queue message from showing a newer edit. Compare the stored hash before applying the human decision; if it changed, create a fresh moderation job.

## 4. When should queue telemetry stop catalog publication?

Logs answer “what happened to this revision?” Emit one structured event at each boundary: ingestion, deterministic rule result, completion result, queue insertion, reviewer decision, and publication transition. Include `revisionId`, `contentHash`, `policyVersion`, adapter/model identifiers, `decision`, `reasonCode`, latency, and an event timestamp. Don't log raw descriptions by default. Catalog copy can contain personal data, supplier secrets, or hostile payloads, and search-friendly log storage is the wrong review surface.

Metrics answer “is the system changing?” Count decisions by category and policy version. Measure completion latency, invalid-response rate, review arrival rate, queue depth, age of the oldest ready job, reviewer throughput, and reversal rate after human review. The last metric is especially useful: if reviewers repeatedly overturn `allow` or `block` for one category, the policy, examples, threshold, or adapter needs inspection.

Alert on user impact, not ordinary variation. A queue-depth alert alone is noisy because imports are bursty. Pair depth with oldest-job age and arrival-minus-completion rate. A practical diagram in words is: catalog API writes pending revision; rules label obvious cases; adapter requests typed output; router either publishes, blocks, or enqueues; reviewer records a decision; publisher compares the hash; metrics observe every arrow.

Then make the dashboard tell that same story.

One path. One ID.

Tracing is optional until there are enough hops to justify it. Correlation is not. Carry one moderation job ID across the catalog write, completion call, queue message, and review action so an on-call engineer can reconstruct the path without joining on description text.

## 5. Which shadow result should trigger a rollback?

Use a frozen, labeled set of real-shaped but appropriately sanitized catalog cases: ordinary descriptions, obfuscated terms, mixed languages, prompt-injection text, conflicting title and description, empty fields, and descriptions near the accepted length limit. Record the expected route and acceptable categories, not an exact explanation string. Run this contract suite against every adapter candidate before deployment.

The migration check has seven parts:

1. The adapter returns schema-valid JSON for every case.
2. The returned `contentHash` matches the submitted revision.
3. Decision changes are reviewed by category, not hidden in one aggregate score.
4. Timeout and malformed-output tests keep content pending.
5. Retries preserve one logical job through an idempotency key.
6. Shadow results cannot publish or block content.
7. Rollback restores both the adapter version and its calibrated thresholds.

Shadow mode is the cleanest before/after. The current adapter remains authoritative while the candidate processes a sampled copy; compare decisions offline, send disagreements to analysts, and avoid exposing candidate output to sellers. A team should define acceptance criteria before looking at the result, or a convenient aggregate can excuse a dangerous category-level regression.

Do not assume a gateway erases behavioral differences. A self-hosted gateway such as LiteLLM can provide one integration point, while the application still owns schema validation, policy versions, calibration, and queue semantics. An embeddings API can support similarity-based retrieval of labeled examples, but similarity is not itself a final moderation verdict. Those are optional components around the contract, not substitutes for it.

## 6. When should a catalog team reject this pattern?

This field guide is not suitable when regulation or internal policy requires a person to approve every catalog change; stick with human-only review and use automation only to order the queue. It is also a poor fit for live chat, where an asynchronous catalog hold has the wrong latency and conversation context. Media moderation needs image, audio, or video-specific evidence and policies rather than a text-only description contract.

The rules-first hybrid carries maintenance cost. Rules can drift away from model policy, reviewers can disagree, and a queue can become a hidden content backlog. Provider portability adds another obligation: each adapter needs its own evaluation and threshold calibration. One interface reduces application coupling. It doesn't make outputs interchangeable.

For a catalog team willing to own those limits, the durable decision is straightforward: keep revisions pending by default, route ambiguity to people, treat JSON as an untrusted input until validated, and measure the queue as a production dependency.

## References

- OpenAI, “Embeddings guide”: https://platform.openai.com/docs/guides/embeddings
- LiteLLM, open-source LLM gateway repository: https://github.com/BerriAI/litellm
