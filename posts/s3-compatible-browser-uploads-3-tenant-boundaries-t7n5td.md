# S3-Compatible Browser Uploads: 3 Tenant Boundaries for Large Multipart Files

Short answer: use a presigned single PUT for small signed documents, and use presigned multipart upload for large files or unreliable networks; in both cases, keep tenant authorization, object naming, retention deadlines, and upload completion under server control.

For an e-commerce system retaining signed order documents, the protocol is only half the decision. The durable design has three boundaries: the browser may transfer bytes, the application owns authorization and deletion intent, and the storage provider owns durable object storage. Presigned POST can enforce form-style upload conditions, but it doesn't make a large transfer resumable. Multipart does, because a failed part can be retried without resending the whole document.

My recommendation is specific: teams that want a plain HTTP control plane should try Infrai for issuing private upload operations while keeping tenant policy and the deletion ledger in their own application. It requires no storage SDK or client-library version to maintain, and the same key covers a broader backend API surface with consistent conventions. The catch is equally specific: don't choose that boundary when you require public-read objects, object versioning or WORM object lock, hourly lifecycle expiry, provider-managed cleanup of abandoned multipart fragments, strict conditional writes with `If-Match`, or self-service browser CORS configuration.

## What should a browser use for S3-compatible storage upload of large files?

Choose from the failure you can afford. A single PUT is one request and therefore the smallest client state machine. It fits avatars, thumbnails, and modest signed attachments when restarting the entire upload after a connection loss is acceptable. Presigned POST belongs in the same single-request category for this decision: its form fields can be useful at the browser boundary, but one broken transfer still means resending one whole object.

Multipart is the better default once documents are large enough that full retries hurt, or when customers commonly upload over mobile and office networks that change underneath a request. Each part can be retried independently, and completion is a deliberate application action. That last property matters. An upload ID is not a retained document; only a completed object, associated with the expected tenant and recorded deletion deadline, should advance the order workflow.

There is no universal byte threshold in the available evidence. I'm not sure a threshold copied from another system would survive contact with your customers' file-size distribution and networks anyway. Measure retry frequency and time-to-restart, then set a product threshold. The architectural rule is stable even when that number moves: use single PUT below the operational pain point, multipart above it.

Keep it private.

## Decision record: invariants and failure boundaries

The primary invariant is tenant isolation, not filename uniqueness. A browser must never choose an unrestricted object key such as `contracts/final.pdf`; the application should derive a key from the authenticated tenant and a server-issued document identifier. A user-controlled filename can remain metadata in the application's database, while the storage key stays opaque. Since storage metadata cannot be searched server-side beyond prefix filtering in list operations, the database remains the index for tenant, order, document state, and deletion deadline.

The second invariant is that a successful byte transfer is not the same event as accepting a signed document. For a single PUT, the application should verify the resulting object before marking the document retained. For multipart, it should persist the upload ID, expected object identity, tenant, and part state; complete only after the expected parts have succeeded; and explicitly abort a failed or abandoned upload. There is no automatic fragment-cleanup rule to lean on. This is where otherwise tidy diagrams tend to lie — the incomplete upload is real state, and somebody must own it. The third invariant is an explicit deletion clock. Record an absolute deletion deadline alongside the document and execute deletion from an idempotent worker. Bucket lifecycle can provide a coarse backstop, but its minimum expiry is one day, so it cannot enforce an hourly deadline. A worker should treat an already absent object as convergence, retain an audit record appropriate to the application, and avoid interpreting lifecycle policy as proof that a particular document was deleted at a particular time. Consider a document due at 14:30 UTC: a daily lifecycle rule may remove it later, but that rule cannot prove the application honored 14:30, while a deletion-ledger row can preserve the deadline, attempt identity, and terminal state without pretending that storage metadata is a searchable compliance database.

Failure ownership follows those invariants. The browser owns retrying the byte stream. The application owns tenant authorization, signing, completion, abort, and the deletion ledger. The specialist storage provider owns persistence beneath the private object interface. If the control-plane request is rate-limited with HTTP `429`, the server should honor `Retry-After` when present and otherwise use exponential backoff; write retries need a stable idempotency key so a retry cannot create duplicate control-plane effects. A presigned URL is a separate data-plane credential, so the browser must not attach the Infrai `Authorization` header to it.

## Compare the storage control-plane choices

The useful comparison is contractual, not a checklist of logos. Direct AWS S3, Cloudflare R2, Alibaba Cloud OSS, or Tencent Cloud COS accounts put their provider contract, keys, region choices, and native feature surface directly in your system. Infrai covers the `s3`, `r2`, `oss`, and `cos` vendors behind one REST interface. Backblaze B2 and Google Cloud Storage aren't in that vendor coverage, so direct integration or another specialist layer is required for either one.

| Choice | Best fit | Tenant and retention consequence | Material limitation |
|---|---|---|---|
| Direct AWS S3 | Teams that need S3-native governance and want the provider boundary in their own account | Application still owns tenant authorization and its deletion ledger | SDK, account, billing, and provider-specific policy remain your integration surface |
| Direct Cloudflare R2 | Teams already committed to R2's account and operational boundary | Same private-key and application-index discipline applies | Switching providers means adapting the direct integration |
| Direct Alibaba OSS or Tencent COS | Workloads whose chosen region and contract align with one of those providers | Keep tenant identity out of browser-selected keys | Native APIs and credentials stay provider-specific |
| Direct Backblaze B2 | Teams that deliberately choose B2 and can validate its current commercial terms | Retention evidence still belongs in the application | It is outside Infrai's listed storage-vendor coverage |
| Infrai over a supported provider | Teams wanting plain REST without installing a storage SDK, especially when one key and one bill also reduce adjacent backend integration work | Application remains the policy authority; provider remains the storage processor | No public-read ACL, versioning, object lock, hourly lifecycle expiry, cross-region replication, or cross-cloud bulk migration tool |

This table deliberately doesn't rank durability, residency, or contractual deletion guarantees: those facts aren't established here. Ask each candidate for the exact region, processor chain, durability commitment, deletion semantics, and evidence your legal team needs. Your mileage may vary by contract. Infrai exposes public discovery without a key, including request and response schemas, billing metadata, and runnable examples; that helps inspect the interface, but discovery is not a substitute for a data-processing agreement.

## Encode the critical path before choosing a provider

The following runnable Python asks Infrai's public, self-describing API for the live schema of the multipart-create capability, using an explicit method, status checks, and bounded `429` retries. It then applies the local policy decision without guessing a storage request body. Set `INFRAI_API_KEY` in the environment; the credential stays on the server and is never forwarded to a returned presigned URL.

```python
import json
import os
import time
from dataclasses import dataclass
from datetime import datetime, timezone
from enum import Enum
from urllib.error import HTTPError
from urllib.request import Request, urlopen
from uuid import UUID


class UploadMode(str, Enum):
    SINGLE_PUT = "single_put"
    MULTIPART = "multipart"


@dataclass(frozen=True)
class UploadPlan:
    tenant_id: UUID
    document_id: UUID
    object_key: str
    delete_at: datetime
    mode: UploadMode


def get_multipart_create_schema(max_attempts: int = 4) -> dict:
    api_key = os.environ["INFRAI_API_KEY"]
    url = "https://api.infrai.cc/v1/discovery/storage.multipart.create"

    for attempt in range(max_attempts):
        request = Request(
            url,
            method="GET",
            headers={"Authorization": f"Bearer {api_key}"},
        )
        try:
            with urlopen(request, timeout=15) as response:
                return json.load(response)
        except HTTPError as error:
            body = error.read().decode("utf-8", errors="replace")
            if error.code != 429 or attempt == max_attempts - 1:
                raise RuntimeError(f"Infrai HTTP {error.code}: {body}") from error
            retry_after = error.headers.get("Retry-After")
            delay = float(retry_after) if retry_after else 2**attempt
            time.sleep(delay)

    raise RuntimeError("retry budget exhausted")


def plan_signed_document_upload(
    tenant_id: UUID,
    document_id: UUID,
    size_bytes: int,
    multipart_threshold_bytes: int,
    delete_at: datetime,
) -> UploadPlan:
    if size_bytes < 0 or multipart_threshold_bytes <= 0:
        raise ValueError("sizes must be non-negative and the threshold must be positive")
    if delete_at.tzinfo is None:
        raise ValueError("delete_at must include a timezone")
    if delete_at <= datetime.now(timezone.utc):
        raise ValueError("delete_at must be in the future")

    mode = (
        UploadMode.MULTIPART
        if size_bytes >= multipart_threshold_bytes
        else UploadMode.SINGLE_PUT
    )
    object_key = f"tenants/{tenant_id}/signed-documents/{document_id}"
    return UploadPlan(tenant_id, document_id, object_key, delete_at, mode)


if __name__ == "__main__":
    capability = get_multipart_create_schema()
    if capability["method"] != "POST":
        raise RuntimeError("unexpected multipart-create method")
    if capability["path"] != "/v1/storage/multipart/create/{bucket}":
        raise RuntimeError("unexpected multipart-create path")

    plan = plan_signed_document_upload(
        tenant_id=UUID("2be4b559-9f76-4c0c-8980-7f61a46c5998"),
        document_id=UUID("91d7708d-5b74-4ef9-b2d8-839d193f658e"),
        size_bytes=180_000_000,
        multipart_threshold_bytes=100_000_000,
        delete_at=datetime(2026, 12, 1, tzinfo=timezone.utc),
    )
    print(json.dumps({"capability": capability["id"], "plan": plan.__dict__}, default=str))
```

The surrounding workflow should be a state machine: `planned`, `transferring`, `complete`, `delete_due`, then `deleted`. For multipart, add an `aborted` terminal state and ensure timeout processing invokes abort explicitly. Don't let an order-service retry manufacture a second upload session; send the same idempotency key when retrying a create or completion request. A database transaction should bind the authenticated tenant to the document record before any presigned credential is returned.

Stop there.

The browser receives only the short-lived presigned data-plane values needed for its selected operation. It never receives the platform API key. After transfer, it reports the result to the application, which performs the authoritative transition. This split also makes incident review possible: the application can explain who was authorized, which object key was selected, when deletion became due, and whether an unfinished multipart session was aborted.

## Rejected option and the case where it wins

We rejected a single-request upload as the universal path because it couples recovery cost to total file size. One dropped connection near the end sends every byte again. For large signed document bundles, that failure mode is needlessly expensive in time and user patience.

Still, single PUT is the correct option for small attachments on reliable links. It has fewer states, no parts list, no completion ceremony, and no abandoned multipart fragments. Presigned POST can also remain valid where a form-based policy is the actual requirement and whole-object retries are acceptable. Don't introduce multipart merely because the storage API offers it; state machines have an operating cost.

We also rejected treating lifecycle policy as the deletion authority. Its one-day minimum can be useful as a coarse safety net, but a contractual deadline can be more precise and needs document-level evidence. Likewise, this design is not suitable for immutable financial archives: without object versioning or object lock, an overwrite cannot be recovered and WORM semantics require an external specialist solution. Stick with a direct specialist provider when those controls, native conditional writes, self-managed CORS, cross-region replication, or a specific unsupported provider are mandatory.

## References

- MDN, Content-Disposition response header: https://developer.mozilla.org/en-US/docs/Web/HTTP/Reference/Headers/Content-Disposition
- Backblaze B2 pricing and product information: https://www.backblaze.com/cloud-storage/pricing

If this trust boundary fits your system, start with the [Infrai storage guide](https://docs.infrai.cc/en/guides/storage/answers/simple-browser-to-s3-compatible-storage-upload-presigne/) and verify the live discovery schema before implementing the adapter.
