# Tenant-Isolated Exports: Secure Browser Uploads for Generic Files in Object Storage

Short answer: for a property-management product that stores generic files and later creates a tenant-scoped export, direct browser upload to private object storage is the simplest secure shape when the application controls a short-lived, object-specific permission; the hard part is tenant isolation and export correctness, not finding the cheapest upload widget.

An apartment photo, inspection PDF, and signed lease may all be bytes, but they do not have the same consequences when they cross a tenant boundary. A browser should never receive a storage account credential. It should receive permission for one object, for one operation, for a short interval. The application server should decide which tenant owns that object before it creates that permission.

That design leaves a less glamorous question in plain view: what exactly does an export mean? If a manager asks for Tenant A's files at 10:00, the result must not acquire Tenant B's objects because a prefix was assembled from an untrusted filename, a stale database row, or a permissive list operation. This is where a cheap-looking shortcut becomes an incident. Imagine a manager exporting a move-out packet while a leasing agent uploads a new inspection photo: if the worker queries by a human-readable property name, accepts a client-supplied tenant value, and reads whatever the bucket lists during the job, the packet can contain the wrong tenant's image without any storage error. The durable fix is an application-owned manifest built from authorized records, followed by object-key checks and an explicit policy for files created after the manifest. That policy may be “exclude them,” “queue a new export,” or “fail and ask for confirmation,” but silence is not a policy.

Trust boundaries come first.

## Can a secure direct browser upload alternative keep generic files tenant-isolated?

The upload flow needs two separate authorization decisions. First, the authenticated application checks that the user may add a file to a particular property and tenant. Second, the storage request is constrained to a server-derived key, such as `tenant-42/property-19/file-01JXYZ/lease.pdf`; the browser may supply a display name, but it does not choose the ownership prefix.

The object store should remain private. A signed upload URL delegates a narrow write, while a later signed download delegates a narrow read. Those URLs are credentials until they expire, so they do not belong in ordinary request logs, analytics events, or error messages. A 403 is useful evidence, not a state transition: it may mean an expired signature or a request whose signed headers no longer match. The application should record the upload as pending until it has independently established that the expected object exists.

I would test the boundary with two tenants and deliberately similar names: `tenant-42` and `tenant-420`. A prefix comparison that is correct for one file can still leak the other if the implementation treats strings as ownership rather than checking a structured tenant ID. Small detail. Big consequence.

The database remains the source of application ownership. Store an internal file ID, tenant ID, property ID, object key, size, media type, checksum if available, and lifecycle state. The state machine might be `pending`, `available`, `quarantined`, and `deleted`; the exact names matter less than refusing to show a row as available merely because a browser reported success. A retry after a lost response must preserve the same logical file or be rejected by an idempotency key. Otherwise, the export will contain duplicates that look like distinct documents.

Exports are permissions, too.

The key layout should serve deletion as well as writes. A tenant prefix makes a bounded cleanup query possible, but deletion still needs an application record and an audit event. GDPR Article 17 describes erasure as a right to have personal data erased, so a deletion job should enumerate the intended tenant scope, compare candidates against application records, delete the selected objects, and retain evidence of what it attempted. It should also define the race: an upload that starts while erasure is running needs an explicit queue or database ordering rule.

## How should generic files become a tenant-scoped export?

An export is a read-side authorization problem, not a glorified bucket listing. The export request should capture the authenticated operator, tenant ID, a point-in-time rule, and the set of file records selected by the application database. Only then should a worker resolve object keys and fetch private objects.

For a small export, the worker can stream files into an archive while checking that every record still belongs to the requested tenant. For a large export, it can create a manifest first, process bounded batches, and publish a result with an expiry. In both cases, the archive itself is a new sensitive object. Its key needs a tenant scope, its download needs a separate signed permission, and its cleanup deadline should be explicit.

Do not infer membership from object names alone. A key prefix is a useful index; it is not an authorization policy. The database query and the storage key must agree, and a mismatch should fail closed. A missing object, a checksum mismatch, or a file deleted after manifest creation should produce a visible partial-export status rather than a successful archive with an unexplained omission.

Here is a deliberately provider-neutral key builder. It does not upload anything; it shows the invariant that is easy to lose when a client filename is allowed to influence the path.

```python
import re
import uuid


def build_object_key(tenant_id: int, property_id: int, filename: str) -> str:
    """Create an application-owned key; the browser never supplies its prefix."""
    safe_name = re.sub(r"[^A-Za-z0-9._-]", "_", filename).strip(".") or "file"
    object_id = uuid.uuid4().hex
    return f"tenant-{tenant_id}/property-{property_id}/file-{object_id}/{safe_name}"


def can_include_in_export(record: dict, requested_tenant_id: int) -> bool:
    return (
        record["tenant_id"] == requested_tenant_id
        and record["state"] == "available"
        and record["object_key"].startswith(
            f"tenant-{requested_tenant_id}/"
        )
    )
```

The last check is defensive duplication, not permission to skip the database predicate. If the key grammar changes, the migration must update both the records and the validation rule. I'm not sure every team needs a point-in-time export snapshot, but teams handling leases or inspection evidence should decide that explicitly; your mileage may vary when managers expect an export to be reproducible a week later.

## Where do simple object-storage designs fail in production?

The first failure is confused authority. A client posts `tenant_id=42` and the server trusts it, or the server signs a path based on a filename that contains `../tenant-43/`. Normalize display names, but derive authorization from the session and database. The second failure is confused completion: the UI marks a file ready before the server has recorded and validated the object. The third is confused deletion: the row disappears while an orphan remains, or the object disappears while a still-visible row points at nothing.

The network adds less dramatic failures. A presign request can be rate-limited with HTTP 429; retry the authorization request with bounded backoff, while making sure a retry cannot issue a different object key for the same logical file. A browser upload can time out after the storage service accepted the bytes. A download can fail halfway through an export. Each case needs an observable state and a retry policy, not a generic “try again” toast.

For every file operation, log the internal file ID, tenant ID, property ID, operation, outcome, and correlation ID. Avoid logging signed URLs and raw filenames when they can contain personal data. Metrics should distinguish authorization denials, expired permissions, missing objects, checksum failures, archive failures, and deletion backlog. A single “upload error” counter is tidy and nearly useless.

The catch is that private object storage is not suitable when the product needs image transformations, permanent public media URLs, immutable retention, or a managed content workflow. In those cases, use a service or storage mode that explicitly supplies the missing capability, and keep tenant authorization in the application anyway. Do not choose a generic store because its byte upload demo is cheap if the team will then rebuild media processing, retention controls, or export orchestration around it.

## Which decision rule survives a cost comparison?

Compare the whole workflow, not only storage bytes or an upload request. A low bill can be offset by the engineering cost of implementing signed access, CORS configuration, orphan cleanup, export retries, audit evidence, and a migration path. Conversely, a higher-level upload or media service may be reasonable when it removes a capability the product actually needs; it is less compelling for private PDFs, ZIP archives, and original property photos that require no transformation.

| Requirement | Generic private object storage | Higher-level upload or media service |
|---|---|---|
| Tenant-scoped authorization | Application-owned keys, records, and signed operations | Still belongs in the application; the service does not know your tenancy model automatically |
| Browser direct upload | Short-lived, object-specific permission | Packaged workflow may reduce UI and callback work |
| Generic files | Natural fit for arbitrary bytes | Check file-type, size, retention, and archive behavior before choosing |
| Public image delivery or transforms | Usually requires separate processing and delivery components | Often the reason to select the higher-level service |
| Tenant export and erasure | Must be designed, tested, and audited by the team | May have helpers, but ownership and deletion evidence remain application concerns |

There is no honest universal “cheapest” answer without volume, retention, egress, request counts, and staff time. Ask instead which boundary you are buying: raw private bytes, upload workflow, or media delivery. For this property-management scenario, the defensible default is private object storage plus an application-owned tenant/file model, provided the team is willing to operate the state machine and export worker that the simple upload screen hides.

## A restrained rollout for tenant-scoped exports

Start with one property and two synthetic tenants. Prove that a user cannot request a key outside the authorized tenant, that a signed write cannot be reused for another key, and that a stale export manifest fails visibly when an object changes. Then exercise 403, 429, duplicate retries, missing objects, deletion races, and an export containing zero files.

Ship the database state machine and audit events before adding a polished progress bar. Run a cleanup job for abandoned pending records and a separate reconciliation job that finds objects with no owning record. Give both jobs bounded tenant scopes and dry-run output. Finally, rehearse erasure and export restoration with test data, because the first real request should not be the team's first encounter with a cross-tenant race.

The migration rule is short: preserve the application file ID and tenant ID, copy objects into the new key grammar, verify ownership and checksums, then switch reads before writes. Keep the old path read-only during the transition, and delete it only after the reconciliation report is reviewed. Cheap and simple are useful properties, but isolation is the property that decides whether the design is acceptable.

## Sources

- https://gdpr-info.eu/art-17-gdpr/
- https://docs.digitalocean.com/products/spaces/
