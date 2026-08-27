# Compatible Object Storage for Receipt Audit Archives: 30/90-Day Lifecycle Rules in US/EU

**Short answer:** for a customer-support system that must process receipts and retain the original file, a cheap compatible storage choice should stay private, use multipart uploads for large archives, record the retention class in a manifest, and apply lifecycle rules only to completed objects after 30 or 90 days. Treat the restore drill, region placement, and deletion evidence as part of the design; a low storage rate does not rescue an archive nobody can reconstruct.

This is an architecture decision record for receipt audit archives, not a shopping list. The difficult question is not how to write bytes to a bucket. It is how to prove, during an audit, that the original receipt was accepted intact, remained private, was held for the promised period, and can still be downloaded by an authorized operator.

## Implement: custody records around each object

The first proof is identity. A receipt object needs a deterministic key that includes the tenant, source system, receipt identifier, and capture date, while a separate manifest records the object key, digest, source event, region, and retention class. Do not make the filename carry every audit fact; object metadata is a poor incident database, and a small relational index is easier to query.

The second proof is completeness. A large original file may cross the network in parts, but an initiated multipart upload is not an accepted receipt. The application should mark the backup complete only after the final completion response, then store the digest and the resulting object identity in one durable record. If a worker dies between those two writes, the record must remain visibly pending instead of being counted as protected data.

The third proof is privacy. Buckets stay private. An operator who has passed the application's authorization check can receive a short-lived presigned download URL; the URL is a bearer capability, so it belongs in neither ordinary logs nor a ticket copied to a broad audience. AWS describes this property and the signing model in its presigned URL documentation.

The fourth proof is placement. “US” and “EU” are policy choices, not latency labels. Put the intended region in deployment configuration, record it in the manifest, and test both destinations with the same restore procedure. A successful write in one region says nothing about the other.

That's the contract. Lifecycle automation is only one part of it.

## Comparison: retention shapes before a policy review

Use two explicit retention classes, such as `support-receipts-30d` and `support-receipts-90d`, represented by prefixes or separate buckets. The choice depends on the provider's lifecycle scope and the team's permission model; the important point is that a 30-day object cannot silently inherit the 90-day policy because somebody changed a default.

Lifecycle rules should be expressed against completed objects and a documented whole-day boundary. The scheduler creates the daily archive, uploads it, writes the manifest, and alerts on missing completion. The lifecycle engine later performs deletion. Those clocks must be monitored separately: a correctly expiring bucket does not prove today's backup exists, and a successful upload does not prove that old receipts are actually leaving.

The retention table should be small enough to review in a pull request.

| Design choice | What it makes clear | Failure boundary | Use it when |
|---|---|---|---|
| One bucket with retention prefixes | Policy is visible in the object key and lifecycle filter | A bad prefix or broad rule can affect both classes | The team can review lifecycle filters and permissions carefully |
| Separate buckets by retention class | Access and deletion policy are easier to isolate | Cross-bucket discovery and restore tooling need more configuration | Audit roles or legal holds differ materially |
| Application-led deletion | The application can attach business context to each deletion | Retries, clocks, and missed jobs become application correctness problems | The storage service cannot express the needed rule |
| Storage lifecycle deletion | Expiry is detached from the backup worker | Lifecycle semantics, timing, and incomplete uploads still need testing | The retention policy is a simple age-based rule |

The catch is that lifecycle is not an immutability guarantee. If an audit or threat model requires deletion resistance, retention locks, or reversible overwrites, this design is not suitable without a storage control that explicitly provides those properties. Keep the original object and its digest under an access policy that matches the requirement; do not imply that a 90-day expiry rule means legal hold.

No policy text can repair a missing object.

## What should a daily backup retention record prove before 30/90-day deletion?

The worker needs a state machine, even if the implementation is a small daily job. A useful set of states is `received`, `uploading`, `complete`, `restore-tested`, and `expired`. Only `complete` is eligible for ordinary retention accounting, and only `restore-tested` proves that the archive format, encryption key, and download authorization still work.

Here is the shape of the write path. The storage client is deliberately an interface: its concrete SDK or HTTP adapter must be tested against the chosen S3-compatible endpoint, while the state transitions remain application-owned.

```python
from dataclasses import dataclass
from hashlib import sha256
from typing import BinaryIO, Protocol


class ObjectStore(Protocol):
    def start_multipart(self, bucket: str, key: str) -> str: ...
    def upload_part(self, bucket: str, key: str, upload_id: str, number: int, data: bytes) -> str: ...
    def complete_multipart(self, bucket: str, key: str, upload_id: str, parts: list[dict]) -> None: ...
    def abort_multipart(self, bucket: str, key: str, upload_id: str) -> None: ...


@dataclass
class ReceiptManifest:
    tenant_id: str
    receipt_id: str
    region: str
    retention_days: int
    object_key: str
    digest: str


def archive_receipt(store: ObjectStore, source: BinaryIO, manifest: ReceiptManifest,
                    bucket: str, part_size: int = 8 * 1024 * 1024) -> None:
    if manifest.retention_days not in {30, 90}:
        raise ValueError("retention must be 30 or 90 days")

    upload_id = store.start_multipart(bucket, manifest.object_key)
    uploaded_parts = []
    digest = sha256()
    try:
        number = 1
        while chunk := source.read(part_size):
            digest.update(chunk)
            etag = store.upload_part(bucket, manifest.object_key, upload_id, number, chunk)
            uploaded_parts.append({"PartNumber": number, "ETag": etag})
            number += 1

        if not uploaded_parts:
            raise ValueError("an empty receipt is not an accepted archive")
        store.complete_multipart(bucket, manifest.object_key, upload_id, uploaded_parts)
    except Exception:
        store.abort_multipart(bucket, manifest.object_key, upload_id)
        raise

    if digest.hexdigest() != manifest.digest:
        raise ValueError("source digest changed during upload")
    # Persist the complete manifest in the same transaction as the job state.
    mark_backup_complete(manifest)


def mark_backup_complete(manifest: ReceiptManifest) -> None:
    raise NotImplementedError
```

The example leaves `mark_backup_complete` as an application boundary rather than pretending that a storage response is a database transaction. In production, persist the expected digest before uploading, persist the completed object identity after completion, and reconcile a crash between those steps. A retry must use an idempotency key derived from the receipt event and retention class; otherwise a repeated queue delivery can create two audit records for one file.

One operational number matters here: `8 * 1024 * 1024` is a chunk-size example, not a universal throughput claim. Measure with the largest receipt bundles your support workflow actually produces, including the network path between the worker and the selected US or EU endpoint. Your mileage may vary with concurrency, encryption, and the provider's multipart limits.

## Test: the full-size archive path

Name failures before they happen. A worker timeout leaves an upload in `uploading`; the reconciler either resumes from recorded part state or aborts it. A duplicate event finds the existing idempotency key. A digest mismatch blocks completion. An expired presigned URL causes a newly authorized URL to be issued, not a public bucket to be opened. A missing lifecycle deletion is an alert and an investigation, not a reason to delete objects from an unrelated prefix.

The restore test should use a real support receipt-shaped archive, but a non-production object. Create it in each region, wait for the completion record, obtain a short-lived signed download after authorization, recompute the digest, and open the original file with the parser used by the support application. Record the result and delete the test object through the same retention-aware process. Run this after a credential rotation and after a lifecycle-policy change. If the archive is large, let the test run through the same multipart path as production; a tiny happy-path object says little about a receipt bundle that crosses a slow link, loses a worker midway, or arrives with a different encryption-key version. The evidence should name the source event, expected digest, completed object, region, retention class, authorization decision, and final parser result, so an auditor can follow one receipt from intake to expiry without relying on a dashboard's aggregate count.

I don't treat a green upload metric as backup evidence. It measures a request, not the existence of a usable audit artifact. The evidence is the manifest, the completed object, the digest match, and a restore event that a reviewer can find later.

Measure the artifact.

The rejected shortcut is to let the support API stream each receipt directly into storage and immediately return success. It looks fast, but it puts the acceptance decision outside the job ledger, makes partial transfers hard to distinguish from complete ones, and gives lifecycle cleanup no reliable notion of which object is valid. Direct streaming can be a reasonable choice for small, non-audit files where loss is acceptable; it is the wrong default for original receipts whose retention is part of the product promise.

For this workload, choose the storage option that can demonstrate these boundaries with the fewest moving parts. Change options when the required region, lifecycle semantics, multipart behavior, or deletion controls cannot be verified in a restore test. Cheap capacity is a constraint. Recoverability is the decision.

## References

- AWS S3 documentation: Presigned URLs: https://docs.aws.amazon.com/AmazonS3/latest/userguide/using-presigned-url.html
- Backblaze Cloud Storage pricing: https://www.backblaze.com/cloud-storage/pricing
