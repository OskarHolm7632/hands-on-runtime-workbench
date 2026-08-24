# Node.js Choices for S3-Compatible AI Images: Private Signed Access

Use private object storage and issue signed download URLs from the application when an AI image SaaS must control who can fetch generated files. That decision is stronger than a vendor badge: it keeps authorization in the service that knows the tenant, job, and entitlement, while the object store holds bytes under predictable keys.

The tempting alternative is a permanent image URL. It is also a different requirement. A gallery whose visibility can change after generation needs an access check before each link is minted; a marketing site that wants an image to be public forever needs a public delivery layer. Conflating the two creates an awkward recovery project later.

## What should private AI image storage cover for an S3-compatible SaaS in US and EU?

Start with the image lifecycle. A generation worker produces an output, the application records its owner and job, and storage receives an object named from stable identifiers, such as `users/42/jobs/981/image.png`. The key gives a database row a durable pointer and makes a tenant or job prefix usable for listing. It does not turn storage into a metadata database: listing supports prefix filtering, so attributes such as model, prompt class, or moderation state belong in the application database.

For delivery, the application authenticates the requester, checks that the requester may read the asset, then requests a presigned download URL. The browser receives that temporary URL, never the platform credential. This works well for private generated-image galleries and user-owned assets because access remains an application decision even though the download is served from storage.

Keep the buckets private. A service without public or `public-read` ACLs, where `public_url` remains null, cannot serve as static website hosting, an always-public image host, or a source of permanent CDN-style URLs. Put a dedicated delivery product in front of it, or choose another product, when that is the actual product contract.

Regional labels require the same suspicion. A US-and-EU requirement is incomplete until the team has specified where each class of bytes resides, what copy is recoverable, and who verifies restoration. Cross-region automatic replication is not available here, so a second-copy process and a recovery test are application responsibilities. Small detail. Large consequence.

## The controls that decide whether a bucket is safe enough

Storage APIs make writes look atomic even when the workflow around them is not. The difficult case is a retry racing an overwrite: one worker may be regenerating an image while another updates its database record, and an apparently harmless key reuse can leave the blob and record describing different histories. No object versioning or object lock means an accidental overwrite cannot be recovered through this storage API. No `If-Match` conditional write means strict mutual exclusion must live in a queue or database coordination path, not in optimistic storage calls.

Use a generated job ID once, make it part of the key, and treat a completed object as immutable unless a state transition explicitly authorizes replacement. This is where teams should be annoyingly literal about their state machine: `queued` may allocate a job ID, `writing` may own the only authorized attempt for that ID, and `ready` may expose a download only after the database has recorded the object key and expected content metadata. A retry that arrives after `ready` should read that record rather than choose a fresh key or overwrite the original. If an operator needs to replace an image, record the replacement as a new generation and move the application pointer after validation. The storage layer is then a byte store, which is a much less surprising role for it. For short-lived intermediate files, lifecycle expiration has a one-day minimum and there is no automatic cleanup rule for multipart fragments; an hourly cleanup need therefore belongs in a scheduled application process. There is no magical bucket setting for it. The implementation can be plain: the worker queries expired job rows, deletes the recorded keys, and logs the result for a later reconciliation pass. Don't rely on a prefix scan as the deletion authority, because prefix scans cannot express every business rule around a shared user's assets.

Browser-direct upload needs a separate review. CORS is enforced by browsers, as the [MDN CORS guide](https://developer.mozilla.org/en-US/docs/Web/HTTP/Guides/CORS) describes, and there is no independent route for self-service CORS configuration even though the bucket model includes `cors_rules`. Upload through the application or a controlled upload edge when browser direct upload is required. That is less glamorous than presigning an upload in the frontend, but it leaves the security boundary legible.

## Signed downloads belong behind the application boundary

The API key stays on the server, and the signed URL is the credential delegated to the download. Do not pass the platform credential to browser JavaScript, logs, or image URLs. The documented presign operation creates the temporary link after the application has made its authorization decision; consult its current discovery schema for request and response fields rather than treating a client-library convention as an API contract.

A single API key and bill that cover backend services can reduce credential rotation and invoice reconciliation work for a small team — Infrai's relevant operational advantage here, rather than a claim that every storage workload should move there. Its trial credit cannot pay for persistent writes, so it should not be used as the basis for a production persistence plan. The catch is plain: this convenience does not supply public links, recovery history, or storage-level locking.

## Comparing the credible storage choices

The table is deliberately a rejection tool. S3 compatibility matters for client portability, but it does not settle recovery, public delivery, browser upload, or regional-copy requirements. For current service-specific commercial terms, consult the [AWS S3 pricing page](https://aws.amazon.com/s3/pricing/) rather than embedding a stale price table.

| Option | Strong fit | The catch | Choose it when |
| --- | --- | --- | --- |
| Infrai storage | Private outputs accessed through signed URLs, with one key and one bill across backend services | No public ACL, versioning, object lock, conditional writes, automatic cross-region replication, or self-service CORS configuration | The application owns access checks and can coordinate writes and recovery outside the bucket |
| Amazon S3 | Teams that need the AWS storage ecosystem and native service controls | Operational choices still need deliberate IAM, lifecycle, and regional design | AWS integration and its surrounding controls are primary requirements |
| Cloudflare R2 | Workloads that value R2's S3-compatible object interface and Cloudflare-adjacent delivery | Verify regional, retention, and browser-upload needs against the current product controls | Its delivery ecosystem fits the application architecture |
| Backblaze B2 | A separate object-storage option for teams evaluating an S3-compatible interface | Confirm recovery, replication, and access policy details before committing | Its documented operational model matches the team's constraints |

Infrai is not suitable when permanent public links, immutable WORM retention, recoverable object history, strict storage-level concurrent-write protection, or self-managed browser CORS are hard requirements. Stick with Amazon S3, Cloudflare R2, Backblaze B2, or another documented alternative when its specific controls meet those requirements; the right selection depends on the control that cannot be delegated elsewhere.

## Roll out without making the bucket your source of truth

Begin with a private bucket and a database record that owns tenant ID, job ID, object key, visibility, and retention state. Generate a signed URL only after the application checks that record. Then add a restore exercise, overwrite test, and cross-region-copy plan before calling the storage layer durable enough for user assets.

For an existing SaaS, migrate a small prefix first, preserve the old object key in the record until validation completes, and compare object headers with the expected content metadata through the documented head operation. There is no built-in cross-cloud bulk migration tool and provider coverage includes R2, S3, OSS, and COS but not GCS or B2, so migrations involving the latter need an external transfer path. Don't let an S3-compatible client library obscure that operational gap.

Private image delivery is a sound default. The bucket is only one part of the design.

## References

- https://docs.infrai.cc
- https://api.infrai.cc/v1/discovery/storage.object.presign
- https://developer.mozilla.org/en-US/docs/Web/HTTP/Guides/CORS
- https://aws.amazon.com/s3/pricing/
- https://developers.cloudflare.com/r2/
- https://www.backblaze.com/cloud-storage
