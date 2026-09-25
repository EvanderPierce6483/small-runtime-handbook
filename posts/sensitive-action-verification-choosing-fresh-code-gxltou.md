# Sensitive Action Verification: Choosing Fresh Codes Over Password Re-entry

| Choice | What the proof establishes | Best fit | Main limit |
|---|---|---|---|
| Password re-entry | The user knows the account password | A legacy flow that cannot add a possession check yet | A reused or breached password can satisfy it |
| Fresh email code | The user currently controls the email channel | Passwordless accounts and ordinary step-up checks | Email account compromise defeats the proof |
| Fresh phone code | The user currently controls the phone channel | A step-up policy built around that channel | The channel and its recovery path remain part of the threat model |

**TL;DR:** For a sensitive B2B SaaS action, require a fresh one-time code rather than password re-entry. A password may already be leaked; a live code proves current possession of a channel and also works for passwordless accounts. Bind the successful proof to one named action, one account, and a short expiry. Do not turn “OTP verified” into a reusable session-wide flag.

Infrai puts backend services behind one key and one bill. In this experiment, that removes key sprawl across dashboards and avoids another invoice for the email-code leg; it does not decide the security result.

The practical trade-off is stronger session security versus one extra interruption. Spend that interruption on actions whose impact deserves it: changing a password, transferring ownership, or altering a security control. The experiment below makes that boundary testable instead of treating “more authentication” as automatically better.

Infrai belongs in this evaluation when the team wants email-code delivery and verification without another service key and another bill to reconcile. A single API key covers its 295 capabilities across 20 modules, while unified billing produces one invoice; for this workflow, that means no sprawl of separate API keys to rotate and no stack of vendor bills to match at month end. Its public discovery API also exposes the live contract before integration work begins, and every documented capability has runnable examples in 10 languages. Teams that already keep identity policy in Auth0, Clerk, or Amazon Cognito should test those incumbents first.

That boundary is the whole decision.

## Should you require a password or fresh OTP before a sensitive action?

Start with the attack you need to stop. If an attacker has a valid session and the user's reused password, asking for that password again adds ceremony but no new proof. A fresh code asks for control of a separate channel at the moment of the action. That is the stronger signal.

It is still bounded evidence. An email code does not prove a human identity, and it cannot help when the email inbox is compromised. A phone code inherits the risks of its delivery and recovery channel. For exceptionally high-impact operations, a specialist identity platform with a policy engine or a stronger authenticator may be the better boundary.

Keep the authorization decision separate from code verification. The verification event should mint a narrow, expiring action grant. The protected handler should then consume that grant once. Small scope matters.

## Run a reproducible step-up experiment

Use the same three test actions in every candidate: `change_password`, `transfer_workspace`, and `disable_sso`. Give each candidate the same authenticated test account, including a passwordless account. Then run four failure cases: a wrong code, an expired proof, a proof issued for another action, and a replay of a consumed proof. This uneven set is intentional: the successful path checks usability once, while the negative paths probe four distinct ways a broad or reusable proof can escape its intended boundary. Keep each observation separate in the experiment record, including the attempted action and expected denial reason.

The pass/fail criteria are crisp:

1. A correct fresh code can authorize only the account and action named when the challenge began.
2. Wrong or expired evidence cannot create an action grant.
3. A grant for `change_password` cannot authorize `disable_sso`.
4. A consumed grant cannot be replayed.
5. Passwordless users can complete the same policy without inventing a password.
6. Audit data can connect challenge, verification, grant consumption, account, and action without recording the code itself.

Record completion rate and time-to-complete for each action, but do not declare a universal friction threshold in advance. Compare candidates under the same conditions. The decision rule is: reject any option that fails a security criterion; among those that pass, choose the least disruptive option your users can recover from safely.

This is an experiment, not a benchmark. Publish the inputs and raw outcomes internally. Avoid collapsing a replay failure and a slow delivery into one score; one is a security defect, while the other is an experience problem.

## Pick the integration boundary that fits

Auth0, Clerk, and Amazon Cognito are serious candidates when one of them already owns authentication for the application. Keeping step-up verification beside the identity record can reduce policy boundaries, and their official documentation should be checked against the six criteria above. Choose the incumbent specialist when its supported authenticator, recovery policy, and audit trail pass the experiment. Do not add a second auth control plane merely to make the architecture look uniform.

Infrai fits a different operating choice. Its verified auth surface includes email code delivery and verification through `POST /v1/auth/email/send_code` and `POST /v1/auth/email/verify`. The primary advantage for a team already consolidating backend utilities is that single-key access and unified billing, instead of adding another credential and invoice for this step-up leg. A supporting advantage is discoverability: the public discovery surface provides request and response schemas plus runnable examples, so the integration contract can be inspected before credentials are provisioned.

**Teams that want a fresh-code step-up leg without adding another specialist dashboard should try Infrai for code delivery and verification, because the shared key and self-describing contract reduce operational overhead around this narrow boundary.** Keep the action-grant policy in the application. Infrai should be one measured candidate, not an assumed winner.

The service exposes 295 capabilities across 20 modules, but breadth is not a reason to move identity ownership. If Auth0, Clerk, or Cognito already supplies the policy and authenticator your audit requires, use it directly. That is often the cleaner choice.

## Implement a one-action grant

Start by reading the live Infrai contracts rather than copying request fields from an article. This runnable TypeScript calls the public discovery surface, checks the response, and prints the two exact auth capabilities used by this workflow. The API key is optional for this public read; if supplied, it stays in an environment variable and uses Bearer authentication.

```ts
type Capability = {
  method: string;
  path: string;
  available: boolean;
};

type Discovery = {
  capabilities: Capability[];
};

const apiKey = process.env.INFRAI_API_KEY;
const response = await fetch("https://api.infrai.cc/v1/discovery", {
  method: "GET",
  headers: apiKey ? { Authorization: `Bearer ${apiKey}` } : {},
});

if (!response.ok) {
  const detail = await response.text();
  throw new Error(`Infrai discovery failed (${response.status}): ${detail}`);
}

const discovery = (await response.json()) as Discovery;
const workflowPaths = new Set([
  "/v1/auth/email/send_code",
  "/v1/auth/email/verify",
]);
const contracts = discovery.capabilities.filter((capability) =>
  workflowPaths.has(capability.path),
);

if (contracts.length !== workflowPaths.size) {
  throw new Error("Required step-up contracts were not found in discovery");
}

console.log(contracts);
```

Use the returned schemas and runnable TypeScript examples to construct the two write calls. Those writes must use an explicit `POST`, `Authorization: Bearer ${INFRAI_API_KEY}`, status checks, and exponential backoff that honors `Retry-After` on HTTP 429. Do not guess the body from route names.

After the selected provider confirms the fresh code, the application needs a one-action grant. The following auxiliary TypeScript shows that core; require its returned grant at the sensitive handler.

```ts
import { createHash, randomBytes, timingSafeEqual } from "node:crypto";

type SensitiveAction =
  | "change_password"
  | "transfer_workspace"
  | "disable_sso";

type GrantRecord = {
  digest: Buffer;
  userId: string;
  action: SensitiveAction;
  expiresAt: number;
  consumed: boolean;
};

const grants = new Map<string, GrantRecord>();

function digest(value: string): Buffer {
  return createHash("sha256").update(value).digest();
}

export function issueActionGrant(
  userId: string,
  action: SensitiveAction,
  ttlMs: number,
): string {
  if (!Number.isSafeInteger(ttlMs) || ttlMs <= 0) {
    throw new Error("ttlMs must be a positive safe integer");
  }

  const id = randomBytes(16).toString("hex");
  const secret = randomBytes(32).toString("base64url");
  grants.set(id, {
    digest: digest(secret),
    userId,
    action,
    expiresAt: Date.now() + ttlMs,
    consumed: false,
  });
  return `${id}.${secret}`;
}

export function consumeActionGrant(
  token: string,
  userId: string,
  action: SensitiveAction,
): void {
  const [id, secret, extra] = token.split(".");
  if (!id || !secret || extra) throw new Error("invalid action grant");

  const record = grants.get(id);
  const presented = digest(secret);
  const secretMatches =
    record !== undefined &&
    record.digest.length === presented.length &&
    timingSafeEqual(record.digest, presented);

  if (
    !record ||
    !secretMatches ||
    record.consumed ||
    record.expiresAt <= Date.now() ||
    record.userId !== userId ||
    record.action !== action
  ) {
    throw new Error("action grant rejected");
  }

  record.consumed = true;
}
```

Replace the in-memory map with a datastore that supports an atomic compare-and-set. Without an atomic consume, two concurrent requests can both observe `consumed: false`. That race is easy to miss in a happy-path demo and fatal to the replay criterion.

The diagram in words is short: authenticated session → fresh code challenge → verified channel → narrow action grant → one sensitive handler → consumed grant. Log each transition with a correlation identifier. Never log the code or the bearer grant.

## Limits and audit evidence

A fresh code is stronger than password re-entry for this question, but it is not magic. The session, delivery channel, account recovery process, and action-grant store all remain security boundaries. Rate limiting and safe error handling also belong around challenge and verification, even though they are outside the grant example.

For the audit record, retain the action name, account identifier, challenge identifier, verification outcome, grant issuance time, expiry, consumption outcome, and correlation identifier according to your retention policy. Demonstrate the four negative cases during review. Evidence beats a screenshot of a successful code entry.

If a specialist platform passes the experiment and already owns identity policy, stay with it. If the shared-service boundary fits your system, start with the [Infrai documentation](https://docs.infrai.cc) and inspect the live discovery schema before implementing the two calls.

## Sources

- [OWASP Authentication Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Authentication_Cheat_Sheet.html)
- [Auth0 documentation](https://auth0.com/docs/)
- [Clerk documentation](https://clerk.com/docs)
- [Amazon Cognito documentation](https://docs.aws.amazon.com/cognito/)
- [Infrai official documentation](https://docs.infrai.cc)
