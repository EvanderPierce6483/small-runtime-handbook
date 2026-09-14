# Delete a Single DNS Record Without Guessing — Evidence Before a Destructive Change

Delete a single DNS record only after discovery returns one immutable record identity and its current version. For an e-commerce platform that lets merchants connect their own domains, that constraint separates removing an obsolete verification record from deleting a live storefront or mail-policy record.

TL;DR: do not build a deletion from a hostname alone. Read the candidate record set, make the selected record explicit, delete that identity once, then verify what the DNS data plane serves. A name can carry several record types, and a TXT owner name can carry multiple strings.

The audit event should say what changed and what evidence failed: `DNS change verification failed for _dmarc.shop.example TXT after change 8f1c`. An operator who sees only `delete failed` has to reconstruct the risk while a merchant's domain is in question. An operator who sees the record name, type, record ID, version, and failed check can decide whether to retry, roll back, or stop the release.

## What makes a single-record delete safe?

DNS users think in names. DNS control planes commonly identify a record with a provider-assigned ID, or with the complete record-set identity. Those are different things. `www.shop.example` can legitimately have A, AAAA, TXT, and CNAME data; a TXT owner name can have several independent values. Deleting by name alone turns an ordinary support request into a broad mutation.

For domains that send mail, begin with visible evidence. A DMARC policy is a TXT record at `_dmarc.<domain>`; RFC 7489 defines that location and how receivers use its policy and reporting data. A merchant may be removing an old verification value while SPF, DKIM, and DMARC still need a carefully managed change window. The interface should display the exact value and type before it permits a destructive action. The record may be obsolete, but that conclusion should be recorded, not inferred from its label.

A deletion request needs four things: the zone chosen by a suffix match, the immutable `recordId` returned by discovery, an `expected` snapshot containing name, type, value, and version when available, plus a `reason` and `changeId` for audit context. The version check matters because a record can change after the console loads it. A delayed browser tab must not erase a replacement another operator just created.

This is the before/after model:

- Before: a click says "remove `_verify.shop.example`" and leaves the service to guess.
- After: discovery produces one record ID, the UI shows the selected value, and the mutation carries the version it read.

Small distinction. Large blast radius.

## How should a merchant-domain console choose the target?

Work backward from the evidence a later investigation will need. A periodic verifier reports that the expected DMARC TXT value is no longer present along the resolver path used by the service. The first response is to identify the change, not to issue more writes. Fetch the audit entry, inspect the before-image, and compare it with the current authoritative answer.

The generic interface below keeps safety policy separate from any particular DNS control-plane HTTP shape. It refuses zero matches and multiple matches. It also makes a retry stable with an idempotency key, so a lost response does not make the caller guess whether the first request reached the control plane.

```ts
type RecordSnapshot = {
  id: string;
  name: string;
  type: string;
  value: string;
  version: string;
};

interface RecordStore {
  find(
    zone: string,
    name: string,
    type: string,
    value: string,
  ): Promise<RecordSnapshot[]>;
  deleteIfVersion(
    zone: string,
    recordId: string,
    version: string,
    idempotencyKey: string,
  ): Promise<void>;
}

export async function deleteExactRecord(
  store: RecordStore,
  input: {
    zone: string;
    name: string;
    type: string;
    value: string;
    idempotencyKey: string;
  },
): Promise<void> {
  const matches = await store.find(
    input.zone,
    input.name,
    input.type,
    input.value,
  );

  if (matches.length !== 1) {
    throw new Error(
      `Refusing delete: expected one exact record, found ${matches.length}`,
    );
  }

  const record = matches[0];
  if (!record.id || !record.version) {
    throw new Error("Refusing delete without record identity and version");
  }

  await store.deleteIfVersion(
    input.zone,
    record.id,
    record.version,
    input.idempotencyKey,
  );
}
```

Zero matches may mean an earlier attempt succeeded, the selected zone was wrong, or the caller's input is stale. Those outcomes need different handling. Two matches means the requested identity was incomplete. Neither outcome authorizes choosing a record on the caller's behalf.

Emit structured fields for discovery count, selected record ID, version match, idempotency key, control-plane outcome, and the later resolver check. Keep a sensitive TXT payload out of broad logs when it contains internal verification material; a digest or access-controlled audit store is enough to correlate the event.

## Why is a successful delete response not enough?

It proves that a control plane accepted a request. It does not prove that authoritative nameservers and recursive resolvers now serve the desired answer. DNS caching makes those observations occur at different times.

After deletion, query the authoritative path and a resolver path from the network boundary that matters to the storefront. Record the observed RRset, query timestamp, and change ID. For a mail-policy change, test the exact policy condition the delivery workflow depends on instead of merely checking that a lookup returned an answer.

Do not convert one cached observation into an automatic rollback. First distinguish an expected cached value from a mismatch at authority. The saved before-image is rollback material, but restoration is a new audited change subject to the same validation. A blind restore can overwrite a legitimate concurrent update.

One target. One delete.

## What about retries and alerts?

The alert should fire on evidence that needs human action. Paging for every resolver disagreement teaches people to ignore normal cache-expiry behavior. Waiting until every downstream mail signal is missing can leave a harmful DMARC or DKIM change undiscovered for too long.

Set the alert around the contract the platform owns: a deletion must produce the expected authoritative RRset and must not remove a record the change request marked as protected. Keep resolver discrepancies as lower-severity observations until they persist beyond the documented cache behavior for that record. Review that boundary after a noisy alert and after an escaped defect.

The trade-off is deliberate. Aggressive verification can catch mistakes sooner, while a poorly chosen threshold creates operator fatigue and retry storms. The durable answer is a change path that preserves identity, evidence, and a clear result when a write is retried.

## References

- https://datatracker.ietf.org/doc/html/rfc7489
