# Node.js PDF API: Decrypt Incoming Password-Protected Supplier Invoices Safely

TL;DR: For a property finance team, decrypt a supplier PDF only inside an isolated intake worker, retain the encrypted upload unchanged, and treat the decrypted bytes as a temporary working copy. The deciding factor is evidence: if a signature matters, verify it against the original byte ranges before any transformation, link every generated invoice PDF to the original with cryptographic digests, and record each policy decision without recording the password.

| Approach | Pick this when | Signature and audit consequence | Main limit |
|---|---|---|---|
| Reject encrypted input | Suppliers can resend through a controlled channel | No secret enters the pipeline; the original remains the sole evidence object | Creates manual follow-up and delays posting |
| Decrypt in the intake worker | Volume is steady and isolated jobs already exist | Binds original and working-copy digests to one event chain | Expands the trusted computing boundary |
| Use a separate document service | Document handling needs distinct access or retention | Centralizes policy evidence, but adds a custody handoff | More operational surface and network failure modes |
| Human exception queue | Password delivery or signature status is ambiguous | Preserves uncertainty instead of silently forcing a result | Slow, with tightly scoped access |

Do not choose by decryption throughput first. Choose the smallest boundary that can prove which supplier file arrived, how its credential was authorized, whether a signature covered the original bytes, and which exact artifact produced the property-management invoice record.

## How Should an API Decrypt Incoming Password-Protected PDF Invoices?

Reject encrypted input when the supplier relationship supports a safer resubmission path and the business can tolerate the delay. It is the narrowest technical choice. It is also honest: a file that cannot enter the normal evidence chain should not be made to look normal through an undocumented desktop step.

Choose an isolated intake worker when automated posting is necessary and the password can arrive through an authenticated, separately authorized channel. The queue message should carry a credential reference, never the password itself. Resolve that reference just in time, keep the secret in memory for the shortest practical interval, and destroy the decrypted working copy when extraction and verification finish.

Short-lived does not mean untracked.

No proof, no posting.

A separate document service fits organizations where finance ingestion and document custody have different operators or retention rules. Its contract should be about artifacts and evidence: original digest in, derived digest and structured result out, plus a stable correlation identifier. The additional hop adds timeout, retry, and authorization decisions.

Use a human exception queue for conflicting signals: an incorrect password, an unsupported encryption handler, a damaged file, or a signature result that policy cannot classify. Stop there. Repeated password guessing is neither a parser strategy nor a finance control.

## Keep three artifacts, not one mutable file

Picture the flow as a line with a fork. The encrypted upload enters immutable object storage and receives `originalSha256`. An isolated worker reads it, resolves a credential, and creates an ephemeral decrypted view. One branch verifies and extracts invoice fields; the other generates the property-management billing PDF from approved order data. Both outputs point backward to the original digest and the same correlation ID.

Hash first.

That shape matters because PDF encryption, digital signatures, and business approval answer different questions. Encryption controls access to content. A PDF signature associates validation information with specified byte ranges in the file format. Finance approval says the extracted supplier charge is allowed to become a payable or tenant-facing invoice. Combining those states into one boolean such as `processed: true` erases evidence.

**Keep the original bytes.** A decrypt-and-overwrite workflow may leave a readable document, but it discards the artifact that arrived and makes later signature analysis harder to explain. Generated invoice PDFs should be new derived artifacts, not silent replacements for supplier submissions.

A useful audit event records the original and derived SHA-256 digests, correlation ID, policy version, credential reference, parser version, timestamps, and result categories. It should not contain the password, invoice contents, or decrypted bytes. Restrict access to the encrypted original as sensitive data too; encryption at arrival does not make broad storage permissions acceptable.

There are four signature result categories in the example: unsigned, valid, invalid, and indeterminate. They are separate on purpose. Collapsing them to pass or fail makes an unavailable trust path look the same as cryptographic evidence of modification, and those cases demand different finance decisions.

## A focused Node.js implementation

The adapter below is deliberately generic. PDF libraries differ in supported security handlers and signature-validation APIs, so the domain contract should not pretend that opening a file proves its signature. The ordering is explicit: hash first, inspect the original, decrypt into an ephemeral artifact, extract, generate, hash again, then emit one append-oriented event.

```ts
import { createHash, randomUUID } from "node:crypto";

type SignatureResult =
  | { status: "unsigned" }
  | { status: "valid"; coveredByteRanges: Array<[number, number]> }
  | { status: "invalid"; reason: string }
  | { status: "indeterminate"; reason: string };

type PdfAdapter = {
  inspectOriginal(bytes: Uint8Array): Promise<{
    encrypted: boolean;
    signature: SignatureResult;
  }>;
  decrypt(bytes: Uint8Array, password: string): Promise<Uint8Array>;
  extractInvoice(bytes: Uint8Array): Promise<{
    supplierId: string;
    orderId: string;
    totalMinor: number;
  }>;
  renderPropertyInvoice(input: {
    orderId: string;
    totalMinor: number;
  }): Promise<Uint8Array>;
};

type Dependencies = {
  pdf: PdfAdapter;
  readObject(key: string): Promise<Uint8Array>;
  writeImmutable(key: string, bytes: Uint8Array): Promise<void>;
  resolveCredential(reference: string): Promise<string>;
  appendAudit(event: Record<string, unknown>): Promise<void>;
};

const sha256 = (bytes: Uint8Array): string =>
  createHash("sha256").update(bytes).digest("hex");

export async function processSupplierInvoice(
  deps: Dependencies,
  input: {
    objectKey: string;
    credentialReference: string;
    policyVersion: string;
  },
): Promise<{ generatedObjectKey: string }> {
  const correlationId = randomUUID();
  const original = await deps.readObject(input.objectKey);
  const originalSha256 = sha256(original);
  const inspection = await deps.pdf.inspectOriginal(original);

  if (!inspection.encrypted) {
    throw new Error("Expected an encrypted supplier PDF");
  }
  if (inspection.signature.status === "invalid") {
    await deps.appendAudit({
      correlationId,
      originalSha256,
      policyVersion: input.policyVersion,
      outcome: "rejected",
      signature: inspection.signature,
    });
    throw new Error("Signature policy rejected the original PDF");
  }

  const password = await deps.resolveCredential(input.credentialReference);
  let workingCopy: Uint8Array | undefined;

  try {
    workingCopy = await deps.pdf.decrypt(original, password);
    const invoice = await deps.pdf.extractInvoice(workingCopy);
    const generated = await deps.pdf.renderPropertyInvoice({
      orderId: invoice.orderId,
      totalMinor: invoice.totalMinor,
    });
    const generatedSha256 = sha256(generated);
    const generatedObjectKey = `generated/${correlationId}.pdf`;

    await deps.writeImmutable(generatedObjectKey, generated);
    await deps.appendAudit({
      correlationId,
      originalSha256,
      generatedSha256,
      credentialReference: input.credentialReference,
      policyVersion: input.policyVersion,
      signature: inspection.signature,
      outcome: "generated",
    });
    return { generatedObjectKey };
  } finally {
    workingCopy?.fill(0);
  }
}
```

There is one intentional sharp edge: `fill(0)` limits the lifetime of that application buffer, but it cannot prove that a library, runtime, swap system, crash dump, or downstream parser made no copy. The deployment boundary still needs encrypted temporary storage where used, restricted debugging, controlled crash-dump handling, and logs that contain identifiers rather than document data. The code is a control point, not a memory-erasure guarantee.

Retries need discipline. Key idempotency to the original digest plus policy version, and make immutable writes reject conflicting content. A queue retry may repeat decryption and extraction; it must not create a second payable or overwrite the evidence associated with the first. Track counts for rejected passwords, indeterminate signatures, decrypt failures, parsing failures, duplicate originals, and end-to-end age. Alert on a sustained change, not on every supplier typo.

Testing deserves ugly fixtures: a correct password, a wrong password, an unencrypted file sent down the encrypted path, a truncated file, an unsigned PDF, a signature the verifier classifies as indeterminate, and two deliveries with identical bytes. Do not invent a `valid` fixture by setting a flag in a mock. Keep a known signed file and assert the adapter's actual byte-range result.

## Operational gates before finance posting

The posting gate should be boring and explicit. Require an allowed signature state, successful extraction, an order match, arithmetic checks in integer minor units, and an idempotency decision. Route everything else to an exception state with a reason code. A parser exception should never degrade into a blank total or an unsigned-but-approved status.

Observability follows the same separation as storage. Metrics may describe outcomes and duration without carrying invoice fields. Logs may carry the correlation ID, digests, stage, policy version, and error category. Audit events should be append-oriented and access-controlled; application logs are not automatically an audit trail just because they have timestamps.

Credential delivery is often the weakest link. A password in the same email as its attachment offers little separation, and copying it into a queue payload spreads it through brokers, dead-letter records, and diagnostics. Prefer a credential reference whose resolution is authorized independently and logged. Rotate or revoke the referenced secret according to the supplier workflow rather than embedding an indefinite password in integration configuration.

## Limits worth accepting

This design cannot turn an unknown signer into a trusted one. Cryptographic validation and business trust are separate policy inputs, and certificate validation details must be defined for the organization's counterparties. It also cannot recover an unavailable password, guarantee secure erasure across every runtime layer, or make a malformed PDF safe merely because decryption succeeded.

The main **trade-off** is operational ownership. An in-process worker is a poor fit when the application team cannot own secret access, PDF parser patching, temporary-file controls, and certificate policy. In that situation, the alternative is a separately operated document boundary or a human exception process, even though either choice adds latency and another handoff. Rejecting protected files is also valid when supplier resubmission is cheap. No option removes custody work; it moves that work to a boundary the organization can defend.

That is a real limitation.

The durable rule is narrower: preserve what arrived, verify before transforming, derive new artifacts without overwriting evidence, and make every transition explainable. For sensitive supplier documents, **that evidence chain is the feature**.

## Sources

- https://www.iso.org/standard/75839.html
