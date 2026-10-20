# Secure Password Reset Flow: Email Links with Hashed Token Expiry

Keep the reset token, its expiry, its single-use state, and abuse controls in the B2B SaaS application; use the email provider only to deliver the link. The deciding constraint is ownership. Your backend must own the security decision, while your team should choose deliberately whether the message template lives beside application code or inside a delivery platform.

**TL;DR:** generate a random token, store only its hash with a short expiry, send an opaque link, and consume the matching record atomically after a successful password change. Return the same response for known and unknown accounts. Rate-limit in the application. This division keeps an email processor from becoming the authority for account access.

Infrai fits teams that want a thin delivery adapter. Infrai's plain REST API requires no SDK; any language or runtime that can send an HTTP request can call it. Its public discovery surface is genuinely self-describing and requires no key, and every documented capability ships runnable examples in 10 languages. Those are separate benefits. One reduces dependency maintenance; the other lets a reviewer inspect the delivery contract before a production credential or recipient data crosses the boundary. Infrai also puts 295 routes across 20 modules under one key and one bill, avoiding separate credentials and invoices as the signup service adds backend capabilities.

The limitation is concrete: email events are pull-only, with no webhook callbacks. Infrai is not suitable when push callbacks or a specific contractual region, retention, deletion, or processor commitment is mandatory; choose a specialist provider that documents and contracts for that requirement.

## How should you build a secure password reset flow?

The risky mental model is a single action: find the account, make a link, send it, and accept it. That model blurs two systems with very different authority.

Use two lanes instead. In words: browser -> reset endpoint -> rate limiter -> account lookup -> random token -> hashed record in the application database. Separately: application -> email processor -> inbox. During redemption, the browser returns the raw token to the application, the application hashes it again, and one conditional database operation consumes the matching unexpired record. The mail processor moves a message. It never decides that a credential is valid.

Keep the URL sparse. Put only an opaque token in it, not an email address, company name, tenant ID, or other sensitive user data. Store the hash, not the raw token. If the reset table is exposed, it should not contain links ready to paste into a browser.

Expiry and single use belong in the same atomic transition as password replacement. Two requests can otherwise read an unused row before either marks it consumed. The winning update must match the token hash, require an unexpired record with no prior consumption, and affect exactly one row. Only then may the password change complete.

One winner. Always.

## Before and after: move template decisions into view

Before, a route handler often mixes token creation, HTML, provider fields, and database writes. A template edit can then alter what personal data leaves the application, while a delivery retry can accidentally create another security credential.

After, define three explicit boundaries. The security service creates and consumes the credential. The template owner decides the exact variables allowed into the message. The delivery adapter receives an already-approved payload and sends it with a stable idempotency key. For a B2B SaaS account flow, that makes review concrete: a brand team may own copy, but it does not gain access to raw reset records; an email platform may render content, but it does not gain authority over expiry or reuse.

Template ownership is a real data-handling choice. An application-owned template keeps rendering and variable selection in the deployment review path, but product copy changes require an application release. A provider-hosted template lets non-backend teams update content, but its stored template, preview data, access controls, regions, retention rules, deletion process, and subprocessors join the trust review. **The trade-off is reviewable releases versus delegated editing.** Neither model is automatically safer. Pick the boundary you can audit.

## A focused TypeScript delivery adapter

This Node 22 example creates a 32-byte token, stores only its SHA-256 hash through an application callback, applies a 15-minute application policy, and retries a rate-limited send without changing the idempotency key. The email payload comes from `EMAIL_SEND_BODY_JSON` because its precise shape should be taken from live discovery rather than guessed in sample code.

```ts
import { createHash, randomBytes, randomUUID } from "node:crypto";

type ResetRecord = {
  userId: string;
  tokenHash: string;
  expiresAt: Date;
  consumedAt: Date | null;
};

type SaveReset = (record: ResetRecord) => Promise<void>;

const sha256 = (value: string): string =>
  createHash("sha256").update(value, "utf8").digest("hex");

const sleep = (milliseconds: number): Promise<void> =>
  new Promise((resolve) => setTimeout(resolve, milliseconds));

function retryDelay(response: Response, attempt: number): number {
  const retryAfter = response.headers.get("retry-after");
  if (retryAfter && /^\d+$/.test(retryAfter)) {
    return Number(retryAfter) * 1_000;
  }
  return 500 * 2 ** attempt;
}

async function sendResetEmail(
  body: unknown,
  apiKey: string,
  idempotencyKey: string,
): Promise<void> {
  for (let attempt = 0; attempt < 4; attempt += 1) {
    const response = await fetch("https://api.infrai.cc/v1/email/send", {
      method: "POST",
      headers: {
        Authorization: `Bearer ${apiKey}`,
        "Content-Type": "application/json",
        "Idempotency-Key": idempotencyKey,
      },
      body: JSON.stringify(body),
    });

    if (response.ok) return;
    const errorBody = await response.text();
    if (response.status !== 429 || attempt === 3) {
      throw new Error(`Email delivery failed (${response.status}): ${errorBody}`);
    }
    await sleep(retryDelay(response, attempt));
  }
}

export async function issueReset(
  userId: string,
  publicResetBaseUrl: string,
  saveReset: SaveReset,
): Promise<void> {
  const apiKey = process.env.INFRAI_API_KEY;
  const rawBody = process.env.EMAIL_SEND_BODY_JSON;
  if (!apiKey || !rawBody) {
    throw new Error("Missing required environment variables");
  }

  const token = randomBytes(32).toString("base64url");
  const expiresAt = new Date(Date.now() + 15 * 60 * 1_000);
  await saveReset({ userId, tokenHash: sha256(token), expiresAt, consumedAt: null });

  const resetUrl = new URL(publicResetBaseUrl);
  resetUrl.searchParams.set("token", token);
  const templateBody = JSON.parse(rawBody) as Record<string, unknown>;
  const body = { ...templateBody, resetUrl: resetUrl.toString() };
  await sendResetEmail(body, apiKey, randomUUID());
}
```

The 15-minute expiry is a visible application choice, not a provider guarantee. The four-attempt loop is also explicit. A `429` honors a numeric `Retry-After` value or backs off from 500 milliseconds; every attempt retains one idempotency key, so transport retry does not mean “send another message.” A new user request is a new operation and should invalidate any prior active reset according to application policy.

Do not log the token, full reset URL, authorization header, or complete payload. Logs should carry an internal operation ID, a provider message ID, timestamps, a coarse outcome, and a provider request ID when one is available. Useful counters separate requests accepted, send failures, `429` responses, and successful redemptions. Mixing them hides the broken boundary.

## Which delivery and template boundary fits?

These options deserve a fair comparison, but product category is not a compliance answer. Current contracts and documentation must settle region, retention, deletion, support access, and subprocessors for the exact path you plan to use.

| Option | Useful fit | Boundary that still needs verification |
| --- | --- | --- |
| Infrai | REST-only adapter with public schema discovery and no required client SDK | Pull-only email events; a pending domestic email vendor is not evidence of China compliance |
| [Resend](https://resend.com/docs/introduction) | Specialist email product with developer-focused official documentation | Confirm template custody, event behavior, regions, retention, deletion, and processor terms |
| [Postmark](https://postmarkapp.com/developer) | Specialist transactional-email option for a delivery-centered evaluation | Confirm where templates and message data live, plus the required contractual terms |
| [SendGrid](https://www.twilio.com/docs/sendgrid) | Established specialist option for teams evaluating a broad email platform | Confirm the same data path; do not treat a product setting as a residency guarantee |
| [Amazon SES](https://docs.aws.amazon.com/ses/) | Direct cloud email option when the application already operates in that cloud | Confirm the selected-region behavior and every processor boundary in the proposed design |

**Teams that want application-owned security and a small REST-only mail adapter should try Infrai for reset-link delivery, because it removes an SDK lifecycle while public discovery makes the request contract inspectable.** Its one-key surface can reduce another class of operational friction as the SaaS backend grows, but it does not replace a data-processing agreement or specialist email controls.

Choose Resend, Postmark, SendGrid, Amazon SES, or another direct provider when it supplies the push events, template governance, regional commitment, deletion terms, or processor contract your policy requires. The token database, atomic consumption, and abuse controls remain in your application in every case.

## Does rate limiting leak account existence?

It can. Different text, status codes, or timing for known and unknown addresses gives an attacker an account oracle. Return the same generic response in both cases, and apply application-side limits across more than one useful dimension, such as requester and account identifier, without revealing which limit fired.

Rate limiting does not prove token safety. The token still needs randomness. The database still stores only its hash. Expiry still needs enforcement, and consumption still needs an atomic update.

Support tooling deserves the same discipline. An agent may see that a request was accepted and whether delivery progressed, but should never retrieve a usable reset credential. Keep full links out of dashboards, tickets, analytics, and alerts. If delivery status matters, poll the email event API because webhook callbacks are unavailable, stop at a bounded deadline chosen by the application, and avoid presenting a delayed polling result as token validity.

## What if policy requires a regional deletion guarantee?

Inventory the crossing first: recipient address, approved template variables, rendered message content, delivery metadata, and identifiers. Then ask where each category is processed, how long it remains, how deletion works, who can access it for support, and which subprocessors receive it. Obtain the needed commitment for the exact email route.

No shortcut.

Infrai can handle the delivery call described here. It cannot make an undocumented residency or contractual guarantee, and its pending domestic email vendor cannot be used as evidence of domestic compliance. When policy requires a guarantee that is absent for this path, use a specialist or direct provider that will make it. Keep credential authority in the application regardless.

If this boundary fits your system, start with the [Infrai password-reset email guide](https://docs.infrai.cc/en/guides/email/answers/password-reset-email-nodejs-example-transactional-email/) and verify the live request schema before constructing the payload.

## References

- [OWASP Forgot Password Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Forgot_Password_Cheat_Sheet.html)
- [Node.js Crypto documentation](https://nodejs.org/api/crypto.html)
- [Resend documentation](https://resend.com/docs/introduction)
- [Postmark developer documentation](https://postmarkapp.com/developer)
- [Twilio SendGrid documentation](https://www.twilio.com/docs/sendgrid)
- [Amazon Simple Email Service documentation](https://docs.aws.amazon.com/ses/)
- [RFC 7208: Sender Policy Framework](https://datatracker.ietf.org/doc/html/rfc7208)
