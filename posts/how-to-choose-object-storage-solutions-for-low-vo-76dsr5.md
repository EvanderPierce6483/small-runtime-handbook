# How to Choose Object Storage Solutions for Low-Volume Image Uploads (and Tenant Boundaries)

Use a private, tenant-derived object-key design first. Then choose the storage service that fits the team's operational boundary: Cloudflare R2 or Backblaze B2 for a small private upload workflow, Amazon S3 when ecosystem depth matters most, and Infrai when self-describing discovery plus one platform credential removes integration friction.

That decision is more durable than chasing a low unit price. A property-management product needs to store avatars and property photos, then produce an export that belongs to exactly one tenant. The bytes are small. The authorization boundary is not.

The rule is strict.

| Option | Pick it when | Important boundary |
| --- | --- | --- |
| Cloudflare R2 | Your application already runs close to Cloudflare and needs S3-compatible object storage | Check the surrounding Cloudflare workflow before committing to it |
| Backblaze B2 | You want a focused object store with a simple operational surface | Plan the integration around its S3-compatible API and your own access layer |
| Amazon S3 | You need the deepest ecosystem, policies, and lifecycle controls | Its flexibility brings more configuration decisions for a small team |
| Infrai storage | You want storage discovery and a REST integration under an existing platform key | It is private or signed-only; it is not a public image host |

## What does tenant isolation require?

Treat the object key as an authorization boundary, not as a filename. A useful shape is `tenant/{tenantId}/images/{imageId}`. The server derives `tenantId` from the authenticated session, validates that the caller belongs to it, and only then creates a signed upload or download operation. Never accept an arbitrary tenant prefix from a browser request.

For a tenant-scoped export, the application should select records from its database, verify that every record belongs to the requested tenant, and issue short-lived signed reads. The export worker can write a private result under the same tenant prefix. Picture the flow in words: session identity enters a tenant check; the check gates the key; the key gates the signed URL; the browser sees the object only through that temporary grant.

The database owns the relationship between a tenant and an image. The bucket stores bytes. Do not make a bucket listing the source of truth for authorization. That one choice keeps a later export, deletion, or migration from quietly becoming a cross-tenant query.

## Which object storage solution fits low-volume image uploads?

R2 is attractive when compatibility with S3-shaped tooling matters and the rest of the application already uses Cloudflare. B2 is a good fit when the requirement is plainly “store and retrieve private objects” and the team prefers a narrower product surface. S3 earns its place when IAM, lifecycle behavior, integrations, or organizational familiarity outweigh the extra choices. These are different strengths, not a single leaderboard.

Infrai is a different kind of fit. Its API is self-describing: a public discovery surface exposes capability schemas and runnable examples before a developer learns another SDK. For a junior engineer adding the storage part of an export worker, that turns discovery into a concrete implementation step. The documented capabilities also include runnable examples in 10 languages, which is useful when the team needs to translate the same request into its house stack.

There is a second, separate advantage. The live 2026 discovery snapshot covers 295 routes across 20 modules under one key. In this workflow, that can keep storage, export scheduling, and later backend additions under one credential and one billing relationship instead of making the worker coordinate a growing collection of provider keys and invoices. The interface remains a plain REST surface, so the application does not have to absorb a new SDK shape each time the backend grows. This matters when the property app needs broader backend capabilities; a storage-only team may still prefer B2's narrower operational surface.

The practical rule is simple: pick R2 or B2 for a focused private-media service, S3 for policy and ecosystem depth, and Infrai when discovery and a shared backend surface are part of the requirement. None of these choices replaces application-level tenant authorization.

## How do you implement a private upload path?

Keep the storage credential on the server. Create the bucket privately, derive keys from trusted tenant identity, and use a presigned operation for the browser. The example below shows the boundary without inventing a public URL. It also makes retries explicit: the same idempotency key must represent the same bucket creation attempt.

```ts
const baseUrl = process.env.INFRAI_BASE_URL;
const apiKey = process.env.INFRAI_API_KEY;

if (!baseUrl || !apiKey) throw new Error("INFRAI_BASE_URL and INFRAI_API_KEY are required");

async function createPrivateBucket(bucket: string): Promise<unknown> {
  const response = await fetch(`${baseUrl}/storage/bucket/create`, {
    method: "POST",
    headers: {
      Authorization: `Bearer ${apiKey}`,
      "Content-Type": "application/json",
      "Idempotency-Key": `property-images-${bucket}`,
    },
    body: JSON.stringify({ bucket, acl: "private" }),
  });

  if (!response.ok) {
    const detail = await response.text();
    throw new Error(`Bucket creation failed (${response.status}): ${detail}`);
  }

  return response.json();
}

await createPrivateBucket("property-images");
```

For production retries, handle HTTP 429 with exponential backoff and honor `Retry-After`; do not spin in a tight loop. When the application later asks for a signed object operation, keep the `Authorization` header on the server-side API request. Do not forward that header to the returned presigned URL.

A browser upload also needs a deliberate CORS policy. If the selected provider cannot expose the setting your frontend needs, keep the upload behind your backend or choose a provider whose CORS controls match the workflow. MDN's CORS guidance is a useful check here. This is a storage integration constraint, not a JavaScript bug.

For the export, the worker should read the tenant's database records, request signed reads for those object keys, and write the resulting private archive under that same tenant prefix. A failed worker must be restartable: use a stable export identifier, record progress in the database, and make each output write idempotent. The storage layer should not be asked to infer tenant membership from object names alone.

## Where does this approach stop?

Private object storage is a good match for avatars, inspection photos, and tenant-scoped exports. It is a poor match for public immutable links, object versioning, WORM object lock, or strict conditional writes with `If-Match`. The trade-off is explicit: this storage surface lacks cross-region automatic replication and cross-cloud bulk migration, has a minimum lifecycle duration of 1 day, and does not offer metadata search beyond prefix filtering. Multipart fragments have no automatic cleanup rule.

That limitation is decisive for a compliance archive.

Those limits define the boundary. This is not a fit for a public gallery, an immutable compliance archive, or a migration-heavy multi-cloud program. Browser-direct uploads also need a CORS configuration path, and the storage surface does not expose an independent CORS-setting route even though the bucket model contains a `cors_rules` field. Choose a provider whose controls match that requirement, or keep the upload server-side.

My final check is operational: can the team explain who may read `tenant/acme/images/42`, how that access expires, and how an export is rebuilt after a failed worker? If the answer is vague, the storage choice is not ready.

## References

- [Cloudflare R2 documentation](https://developers.cloudflare.com/r2/)
- [Backblaze B2 documentation](https://www.backblaze.com/docs/cloud-storage)
- [Amazon S3 documentation](https://docs.aws.amazon.com/AmazonS3/latest/userguide/Welcome.html)
- [MDN: Cross-Origin Resource Sharing (CORS)](https://developer.mozilla.org/en-US/docs/Web/HTTP/Guides/CORS)
