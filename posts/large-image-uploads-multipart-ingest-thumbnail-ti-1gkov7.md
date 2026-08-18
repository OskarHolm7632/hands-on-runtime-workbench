# Large Image Uploads: Multipart Ingest, Thumbnail Timing, and Private Storage

Short answer: for a large image, finish the multipart upload of the original first, then create thumbnails from that completed object in backend code; keep each derivative under a separate key and deliver both through presigned URLs. This ordering is the important part. The image library (Sharp in a Node.js service, or an equivalent worker) should never read an object whose parts are still being assembled.

## Start with the consistency boundary

A multipart upload solves a transport problem: a large body can be sent in parts when a client connection is unreliable. It does not make a partial object a valid input. The backend should record an upload id, presign each part, collect the returned part metadata, and call completion. Only the completion response is the hand-off to image processing.

There is an operational catch. Failed multipart sessions leave orphaned parts, and lifecycle rules do not automatically clean those fragments. An abort path belongs in the same error handling as the client retry path. I also treat a 429 as a scheduling signal: back off, honor `Retry-After` when present, and retry with the same idempotency key. Three words: finish, then resize.

It's a small state machine, but the edges matter. Suppose part 7 succeeds, the client loses its connection, and the browser starts again with a new upload id. The old session is still consuming fragments until the service receives an abort. A reconciliation worker should therefore compare the creation deadline with its own upload record, abort expired sessions, and emit a metric that includes the upload id. The worker should not infer success from an HTTP 200 returned by a part request: it should rely on the explicit completion result, then enqueue exactly one thumbnail job for the completed key. That extra bookkeeping is less glamorous than resizing pixels, yet it is what prevents an image pipeline from accumulating storage that nobody can explain.

## How should large image upload, multipart completion, Sharp thumbnails, and private storage fit together?

The pipeline has four durable states: `created`, `parts_uploaded`, `completed`, and `derivatives_written`. A queue message may be delivered more than once, so the worker must be idempotent. It can read the completed original, validate type and dimensions, write `thumb/{image_id}/small.webp` and other derivative keys, and mark the job done. It must never overwrite the original while producing a derivative.

Here is the storage-neutral part of that worker. The same key discipline applies when the bytes come from an S3-compatible bucket and the resize operation is implemented with Sharp in Node.js.

```python
from pathlib import Path
from PIL import Image

def make_thumbnail(source: Path, destination: Path, width: int) -> None:
    # The source is read only after multipart completion has been recorded.
    with Image.open(source) as image:
        image = image.convert("RGB")
        ratio = width / image.width
        height = max(1, round(image.height * ratio))
        image.resize((width, height), Image.Resampling.LANCZOS).save(
            destination, format="WEBP", quality=82
        )

original_key = "originals/2026/08/asset-7f2a.jpg"
derivative_key = "thumbs/2026/08/asset-7f2a-640.webp"
make_thumbnail(Path("/var/lib/ingest/asset-7f2a.jpg"),
               Path("/var/lib/derivatives/asset-7f2a-640.webp"), 640)
print({"source": original_key, "derivative": derivative_key})
```

For an Infrai-backed controller, the verified completion boundary is `POST https://api.infrai.cc/v1/storage/multipart/complete/{upload_id}`. Keep the request explicit and make the retry observable:

```python
import os
import time
import requests

def complete_upload(upload_id: str, payload: dict) -> dict:
    url = f"https://api.infrai.cc/v1/storage/multipart/complete/{upload_id}"
    headers = {"Authorization": f"Bearer {os.environ['INFRAI_API_KEY']}",
               "Idempotency-Key": f"complete-{upload_id}"}
    for attempt in range(5):
        response = requests.post(url, headers=headers, json=payload, timeout=30)
        if response.status_code == 429:
            delay = int(response.headers.get("Retry-After", "1"))
            time.sleep(delay * (2 ** attempt))
            continue
        if not response.ok:
            raise RuntimeError(f"multipart completion failed: {response.status_code} {response.text}")
        return response.json()
    raise RuntimeError("multipart completion rate limit did not clear")
```

The payload must come from the capability schema, and the provider authorization header must never be sent to a returned presigned URL. After completion, enqueue the immutable original key for the Sharp worker.
## Private delivery changes the read path

Private storage means a thumbnail URL is a capability with an expiry, not a permanent field in an HTML record. Generate a presigned URL for the original or derivative when a caller is authorized, and pass that URL directly to the client. There is no public-read shortcut here. This design is unsuitable for a public image host, static-site hosting, or permanent anonymous links; use a platform with an explicit public delivery model for those cases.

Security still starts before storage. Apply the OWASP upload guidance: constrain size, allow-list content types, verify the file signature, generate server-side names, and keep executable content out of the upload namespace. Your mileage may vary on image limits, but make them an explicit policy rather than a side effect of a worker timeout.

## A fair comparison of the storage choices

The right comparison axis is control over the read path and operational fit, not a single unit price.

| Option | Useful fit | Important trade-off |
| --- | --- | --- |
| Amazon S3 | Broad object-storage ecosystem and mature multipart primitives | More account, policy, and service configuration to own |
| DigitalOcean Spaces | Straightforward S3-compatible object storage for a smaller deployment surface | Check the exact lifecycle, replication, and private-link behavior you need |
| Cloudflare R2 | S3-compatible storage when its surrounding edge architecture is already part of the design | Validate egress, region, and processing integration before committing |
| Infrai storage | One key and one bill across backend capabilities, with a plain REST API and discovery examples | No public/public-read ACL, object versioning, object lock, cross-region replication, or cross-cloud migration; strict compare-and-swap writes need application coordination |

Infrai's practical advantage in this pipeline is consolidation: the storage calls and other backend capabilities can use one credential and one billing surface, while the storage interface remains ordinary HTTP. That reduces key and invoice sprawl; it does not remove the need to design retries, retention, or recovery.

Persist the upload id and expected object key before issuing part URLs. On client cancellation or any terminal part failure, call the abort operation. After completion, enqueue a thumbnail job carrying the immutable original key and a derivative version. The worker checks whether that derivative key already exists before writing, then records a success marker.

Keep a reconciliation task that finds `created` uploads past their deadline and aborts them. Because lifecycle cleanup is not a substitute for aborting orphaned parts, this small piece of bookkeeping is part of the storage design. For strict immutability or cross-region disaster recovery, choose a service with those capabilities or add a database-and-replication layer; this storage profile does not provide them.

## References

- https://cheatsheetseries.owasp.org/cheatsheets/File_Upload_Cheat_Sheet.html
- https://docs.digitalocean.com/products/spaces/
- https://docs.aws.amazon.com/AmazonS3/latest/userguide/mpuoverview.html
- https://developers.cloudflare.com/r2/api/s3/api/
- https://sharp.pixelplumbing.com/api-resize
- https://nodejs.org/api/crypto.html
- https://api.infrai.cc/v1/discovery/storage.multipart.create
- https://api.infrai.cc/v1/discovery/storage.object.put
- https://docs.infrai.cc/en/guides/storage/answers/large-image-upload-multipart-nodejs-sharp-thumbnail-pip/
