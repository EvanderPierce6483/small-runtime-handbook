# 5 Loyalty Identity Checks: Deduplicate Game Accounts Before User Creation

For loyalty account deduplication, identity resolution must happen before user creation: when a player asks to reset a password, the dangerous question is not “does this email exist?” It is “which account, if any, may this recovery request touch?”

**Short answer:** run identity resolution as a gated, observable decision before creating or linking a loyalty user; require a high-confidence match for automatic recovery, and send ambiguous cases to a step-up path instead of guessing.

That ordering changes the system. A signup form is no longer the source of truth. It is an input to a small decision service that compares normalized signals, records why it chose a path, and only then creates a user record or starts password recovery.

## 1. Treat duplicate detection as a recovery decision

In a game membership system, one person may arrive with an old console address, a new mobile email, and a phone number shared with a family account. Exact string equality misses legitimate matches; fuzzy matching can merge two real players. The cost of a false merge is high: purchase history, rewards, and recovery authority move to the wrong identity.

Use a staged model. Normalize first (Unicode-aware email case folding, phone formatting, and provider-specific alias policy). Then compare independent signals: verified email, verified phone, console subject, loyalty card number, and recent device evidence. A match on one weak signal should never create a privileged link by itself. In one audit replay, the same household phone appeared on three gamer profiles; treating it as a primary key would have handed password recovery to the wrong person, while treating it as a supporting signal kept all three records separate and sent the collision to step-up verification.

The output should be an explicit state, not a boolean:

| State | Meaning | Recovery action |
| --- | --- | --- |
| `new` | No trustworthy account match | Create a pending identity, then verify |
| `match` | One account clears the confidence threshold | Continue recovery for that account |
| `ambiguous` | Several accounts or weak signals conflict | Step up with an independent factor |
| `blocked` | Risk policy rejects the request | Stop and log a review event |

This is identity resolution before user creation. The user table is downstream of the decision, not the place where you discover collisions.

## 2. What should a recovery gate record before creating a loyalty user?

An auditor needs to reconstruct the decision without reading application logs from five services. Store a decision event with a correlation ID, policy version, signal classes (not raw secrets), candidate count, confidence band, selected path, and reason code. Keep the original request body out of general logs; OWASP recommends careful handling of authentication data and generic responses that do not reveal account existence.

Here is a compact TypeScript shape for the boundary. It is deliberately vendor-neutral and can sit in front of any identity store.

```ts
type Resolution = "new" | "match" | "ambiguous" | "blocked";

type Candidate = {
  userId: string;
  score: number;
  signals: string[];
};

type Decision = {
  resolution: Resolution;
  userId?: string;
  reason: string;
  policyVersion: string;
};

export function resolveRecovery(
  candidates: Candidate[],
  policyVersion = "recovery-2026-01",
): Decision {
  const ranked = [...candidates].sort((a, b) => b.score - a.score);
  const top = ranked[0];
  const second = ranked[1];

  if (!top) {
    return { resolution: "new", reason: "no_candidate", policyVersion };
  }
  if (top.score < 0.85) {
    return { resolution: "ambiguous", reason: "low_confidence", policyVersion };
  }
  if (second && top.score - second.score < 0.10) {
    return { resolution: "ambiguous", reason: "close_scores", policyVersion };
  }
  if (!top.signals.includes("verified_factor")) {
    return { resolution: "blocked", reason: "no_verified_factor", policyVersion };
  }
  return {
    resolution: "match",
    userId: top.userId,
    reason: "single_high_confidence_candidate",
    policyVersion,
  };
}
```

The thresholds here are policy examples, not universal truth. Calibrate them against labelled duplicate cases, then version the policy so a later audit can explain why yesterday’s request took a different path. Your mileage may vary: a children’s game with shared household numbers needs stricter rules than a business loyalty programme.

Emit metrics for each resolution state and reason code. Alert on a sudden rise in `ambiguous`, a drop in `verified_factor`, or a spike in recovery requests per account. A dashboard that only shows successful resets hides the attack surface.

## 3. How do you test identity resolution and password recovery together?

Build a fixture set that looks like production, including diacritics, plus-addressing, recycled phone numbers, console account transfers, and two siblings sharing a device. Every fixture should assert both the resolution state and the user-visible response. The response should be deliberately generic: “If the details match an account, we’ll send next steps.” That prevents account enumeration while still giving support a correlation ID.

Run property tests around invariants: no two loyalty users may claim the same verified external subject; an ambiguous result must not create a privileged account; replaying the same idempotency key must not create a second user; and changing a policy version must leave the old decision event intact.

For operations, trace the flow as a short chain: request received -> signals normalized -> candidates scored -> policy decision -> factor challenged -> credential changed. Redact tokens and recovery links at the logger boundary. Keep audit events append-only, restrict who can query them, and define retention with your privacy team.

Provider choice affects plumbing, not these invariants. Auth0 and Clerk offer managed account workflows, while Keycloak gives teams more control over self-hosted identity data. Each can be made to fit this gate, but their extension points, export paths, and operational ownership differ. Compare those boundaries with your incident response skills and data residency needs; a familiar tool with weak event export is a poor audit fit.

## 4. The trade-offs that decide the design

The catch is that higher assurance adds friction. Requiring two independent factors reduces false merges, yet it can strand a legitimate player who lost a phone and an old email at once. A manual review queue is slower and costs staff time, but it is safer than silently joining records.

Do not use this pattern when your product has no durable account benefit and can tolerate anonymous progress; an ephemeral guest profile may be enough. Stick with a simpler signup check when there is only one authoritative identifier and no cross-device recovery. Choose a managed provider when your team cannot operate key storage, rate limits, and recovery messaging. Choose a self-hosted system when control over data shape and audit export outweighs that maintenance burden.

Start small: instrument the decision boundary, replay a month of anonymized cases, and inspect every `match` that depended on a single signal. The useful outcome is not a clever score. It is a recovery path that a support engineer, an auditor, and the player can all understand.

## References

- https://cheatsheetseries.owasp.org/cheatsheets/Authentication_Cheat_Sheet.html
- https://pages.nist.gov/800-63-3/
- https://www.w3.org/TR/2021/REC-xmlschema11-2-20210422/
- https://opentelemetry.io/docs/concepts/observability-primer/
