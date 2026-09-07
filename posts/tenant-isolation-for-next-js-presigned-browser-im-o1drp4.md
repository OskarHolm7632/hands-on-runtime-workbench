# Tenant Isolation for Next.js Presigned Browser Image Uploads

Short answer: for a logistics system uploading delivery photos, let the browser send image bytes directly to private object storage with a presigned upload only after the backend has bound the key to a tenant; then let a backend worker create thumbnails from the verified original. Proxy the upload when the browser policy cannot be tested or when the application must inspect every byte before storage.

The important boundary is tenant isolation, not shaving one network hop. A Next.js frontend can request permission, but it must never choose an arbitrary bucket path. A Node.js backend should authenticate the operator, allocate an immutable tenant-scoped key, and treat the browser's completion message as untrusted. Presigning solves credential distribution. It does not prove that the object belongs to the right customer, that the file is an image, or that every thumbnail belongs to the same original.

## The tenant contract is the first artifact

Consider a driver uploading a damaged-pallet photo for tenant `north-hub`. The application creates a record before it creates a URL. That record contains the authenticated tenant, an image identifier, an upload generation, an expected media policy, and a state such as `awaiting_upload`. The object key can then be derived from server-owned values:

```python
from dataclasses import dataclass


@dataclass(frozen=True)
class UploadTarget:
    tenant_id: str
    image_id: str
    generation: str

    @property
    def original_key(self) -> str:
        return (
            f"tenants/{self.tenant_id}/images/"
            f"{self.image_id}/{self.generation}/original"
        )


target = UploadTarget("north-hub", "img-8f2d", "g-01")
assert target.original_key == (
    "tenants/north-hub/images/img-8f2d/g-01/original"
)
```

The browser receives a presigned request for that exact key and an expiry chosen by the backend. It does not receive a storage credential, and it does not submit a tenant ID that the storage layer must trust. On completion, the backend checks the authenticated image record, verifies that the expected private object exists, and queues work using the record's generation. A repeated completion call should return the same state rather than create another image job.

This is where direct upload often gets oversold. CORS is evaluated by the browser, so a server-side test can pass while the real preflight fails. Test the actual origin, method, and headers in a browser-like integration test. If that path is not compatible with the deployed policy, proxy the bytes through the backend and apply the same key and state rules there. Direct transfer is a transport choice; the authorization model remains yours.

One short rule: the client proposes; the backend decides.

## How can tenant data move from browser upload to object storage and backend thumbnails?

The worker should not start because a JavaScript callback said “done.” It should start after the backend has verified the original object and moved the database record to a durable processing state. Validation belongs before decoding: inspect the bytes, enforce the application's input and decoded-pixel limits, and reject content that cannot be decoded as an allowed image. A filename ending in `.jpg` is not evidence.

Keep the original and its variants in one generation. For example, `original`, `thumb-320`, and `thumb-1280` are separate objects but one logical image. Write variants under deterministic keys, and make the database's `ready` transition conditional on that generation after every required variant is present. If a worker dies after writing one thumbnail, the retry can safely continue. If a replacement begins, a new generation prevents an old thumbnail from being paired with a new original.

The transform itself can remain independent of the storage provider. This small Python function shows the boundary a Node.js worker needs to implement, without pretending that a thumbnail library is an authorization system:

```python
import io

from PIL import Image, ImageOps


def make_thumbnail(source: bytes, width: int) -> bytes:
    if width <= 0:
        raise ValueError("width must be positive")

    with Image.open(io.BytesIO(source)) as image:
        image.verify()

    with Image.open(io.BytesIO(source)) as image:
        corrected = ImageOps.exif_transpose(image).convert("RGB")
        corrected.thumbnail((width, width * 8))
        output = io.BytesIO()
        corrected.save(output, format="JPEG", quality=85, optimize=True)
        return output.getvalue()
```

The height bound is deliberately generous because width is the product constraint in this example; it is not a universal safety limit. The service still needs an explicit byte limit and a decoded-pixel limit suited to its workload. I'm not sure what those limits should be for your fleet, and anyone giving one number without knowing the camera mix is guessing.

## Evaluate a failure matrix for retries, isolation, and visibility

Object storage is a byte store, not the system of record for tenant membership, moderation state, or the active thumbnail generation. Keep those searchable attributes in the application database. The database should also record the expected key, content state, and a job identifier. Listing by a tenant prefix can support reconciliation, but it cannot authorize a request or replace a query over image ownership.

The failure table is more useful than a promise that uploads are “simple.”

| Failure mode | Invariant to enforce | Observable check |
|---|---|---|
| A client retries completion | One image generation maps to one processing job | Repeated completion returns the existing state |
| A worker stops after one variant | `ready` requires every required variant | A partial set never becomes visible as complete |
| A user replaces an image during processing | New writes use a new generation | The active pointer changes only after publication |
| A key is guessed across tenants | Keys and database lookups are scoped to the authenticated tenant | Cross-tenant reads fail before object retrieval |
| A browser upload is abandoned | Unfinished records and objects have a cleanup policy | A scheduled reconciliation reports stale state |

Presigned download URLs should be issued only after the same tenant authorization check. Cache policy then follows key lifetime. Immutable generation keys can tolerate long-lived caching; mutable aliases need revalidation or a short policy. MDN's `Cache-Control` reference is the right place to confirm the response directives instead of treating a CDN cache as an access-control layer.

Don't use a cached URL as proof of authorization.

## How can direct and proxied paths be compared against tenant isolation?

Choose a direct presigned browser upload when the production CORS request passes, the backend can constrain the target key, and the team can operate completion, validation, retries, cleanup, and private reads. Choose backend proxying when browser policy is outside your control, when admission control requires inspecting bytes before they reach storage, or when the extra hop is acceptable in exchange for a simpler browser boundary. Both paths can feed the same thumbnail worker and the same generation state machine.

| Decision question | Direct browser upload | Backend proxy |
|---|---|---|
| Who moves the original bytes? | Browser to private storage | Browser to application, then application to storage |
| Where is tenant authorization decided? | Before presigning and again before reads | At the application upload endpoint and before reads |
| What can a CORS failure affect? | The upload path itself | The browser-to-application path instead |
| What must be tested? | Preflight, signed upload, completion, and signed read | Admission, streaming limits, storage write, and the same completion flow |

The catch is operational ownership. Direct upload is not suitable when the organization cannot monitor abandoned uploads or verify the browser request in each deployed environment. Proxying is not suitable when the application cannot absorb the byte volume or when its request limits are lower than the original media. Stick with the simpler path until the larger-media requirement is real and measured.

Price is a secondary filter. A published storage pricing page can help estimate the object and transfer budget, but it cannot establish tenant isolation, CORS behavior, signing semantics, or thumbnail correctness. Those are acceptance tests, not marketing claims.

## Roll out direct uploads from measured evidence

Start with one tenant, one original, and one thumbnail width. Record the image state and object key at each transition. Exercise a duplicate completion, a worker retry, a cross-tenant read attempt, and a replacement while processing is still running. The test is successful only when the old generation remains coherent and the new one is invisible until complete.

Then add more variants, retention, and multipart handling as separate changes. Instrument presign issuance, upload completion, validation rejection, queue delay, transform duration, variant publication, and cleanup age. Alert on records that remain in `awaiting_upload` or `processing` beyond an application-defined window. Your mileage may vary on the window; camera connectivity and warehouse workflows matter more than a fashionable default.

The decision rule is compact: presign only a server-chosen, tenant-scoped key; verify before processing; publish a complete immutable generation; and authorize every read. When direct browser upload cannot meet those conditions, proxy it. The thumbnail architecture does not need to change.

## References

- https://developer.mozilla.org/en-US/docs/Web/HTTP/Reference/Headers/Cache-Control
- https://www.backblaze.com/cloud-storage/pricing
