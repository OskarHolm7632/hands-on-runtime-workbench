# SaaS Image Storage: Object Storage, Database Blobs, or Local Disk?

Short answer: for a production SaaS that keeps generated images, place immutable image bytes in object storage and keep authorization, lifecycle state, and the object key in a database; use database blobs only for deliberately bounded transactional attachments, and treat local disk as disposable workspace or cache.

The operational constraint is not where a write call feels easiest. It is whether an image shown as ready remains readable after a retry, a deploy, a worker move, and a restore. Generated media makes this boundary hard to ignore: the files can be large, their delivery pattern is uneven, and an asset record can become visible while its bytes are still somewhere else. A storage decision record should make that gap explicit before it becomes a support incident.

## What should a production SaaS use to store generated images: object storage, database blobs, or local disk?

Choose object storage plus a small metadata row when images are durable user assets. The row owns tenant authorization, generation-request identity, media type, byte length, checksum, timestamps, and lifecycle state; the object store owns the immutable bytes. Keep an opaque object identifier in the row rather than a delivery URL, because a delivery domain or signing mechanism is a deployment choice, whereas the identifier remains application data.

Do not make a URL the primary identifier.

The useful invariants are plain. A ready asset has bytes retrievable through the authorized read path. Repeating one generation request does not publish two assets. A tenant cannot obtain another tenant's bytes by guessing a key. Deletion is a state transition with a reconcilable cleanup path, not two unrelated calls that happen to run near one another.

Write those invariants down.

| Persistence choice | Appropriate use | Failure boundary | Cost of the choice | Important limit |
|---|---|---|---|---|
| Object storage with metadata | Durable, growing image catalogs and direct delivery | Object acknowledgement and database commit are separate | Reconciliation and two recovery domains | It cannot make a cross-system write one database transaction |
| Database blob | Small, bounded attachments with a real transactional requirement | The database write, replicas, and backups | Larger backups and replication, plus query-path pressure | Capacity needs an explicit size and throughput budget |
| Local disk | Development, evictable cache, or reproducible temporary output | Host lifetime, scheduling, volumes, and deployment | Placement, capacity, and restoration operations | It is not a shared SaaS system of record |

This is a default, not a slogan. The catch is the extra state boundary: object storage trades database pressure for a reconciliation problem. A service that must atomically commit a very small attachment with a parent row, and can prove a hard size ceiling, may reasonably retain the blob in its database. Local disk is valid for an expendable decoding workspace. It is not suitable for authoritative assets across horizontally scheduled application instances.

## The write path must model partial completion

An object write and a relational commit commonly have independent acknowledgement points. Trying to describe that as atomic hides the work that determines recovery behavior. Allocate the asset and its opaque key, write a pending metadata row with an idempotency key, upload the bytes, verify the bound metadata, then conditionally move the row to ready. The application publishes only ready assets.

A process can stop after the upload and before the ready transition. That leaves an unreferenced object and a pending row, both detectable by a scheduled reconciler. A stop before upload leaves a pending row without bytes. A repeated request must consult the idempotency key before allocating another visible asset. These cases are not exotic; queues redeliver and clients retry after losing a response.

Here is the critical path in Python. The adapter boundary is intentionally generic, because the domain decision should not depend on a particular storage SDK.

```python
from dataclasses import dataclass
from hashlib import sha256
from typing import Protocol
from uuid import UUID, uuid4


class ObjectStore(Protocol):
    def put(self, key: str, body: bytes, media_type: str) -> None: ...


class AssetRepository(Protocol):
    def create_pending(
        self, asset_id: UUID, tenant_id: UUID, request_id: str, object_key: str
    ) -> bool: ...

    def mark_ready(
        self, asset_id: UUID, byte_length: int, checksum: str, media_type: str
    ) -> bool: ...


@dataclass(frozen=True)
class ReadyAsset:
    asset_id: UUID
    object_key: str
    byte_length: int
    checksum: str


def persist_image(
    tenant_id: UUID,
    request_id: str,
    body: bytes,
    media_type: str,
    objects: ObjectStore,
    assets: AssetRepository,
) -> ReadyAsset:
    asset_id = uuid4()
    object_key = f"tenants/{tenant_id}/assets/{asset_id}"

    if not assets.create_pending(asset_id, tenant_id, request_id, object_key):
        raise ValueError("request already owns an asset")

    digest = sha256(body).hexdigest()
    objects.put(object_key, body, media_type)

    if not assets.mark_ready(asset_id, len(body), digest, media_type):
        raise RuntimeError("ready transition was not accepted")

    return ReadyAsset(asset_id, object_key, len(body), digest)
```

The sketch deliberately does not pretend that a compensating delete is atomic rollback. A reconciler should inspect old pending rows, compare them with object inventory or object metadata, and apply a documented retention window before cleaning up an orphan candidate. For large outputs, stream the bytes and compute the checksum incrementally; holding an unbounded upload in application memory turns a storage choice into a memory admission problem.

## Delivery, access control, and HTTP behavior

Store keys are not permissions. Keep the byte namespace private by default, load metadata first, authorize the tenant and asset state, then issue a time-bounded delivery capability or route the request through a controlled byte-serving layer. A user-provided filename belongs in presentation metadata, never in the object key.

When a browser response needs a download name, use `Content-Disposition` to express inline or attachment handling and its filename parameters. The header does not authorize access, validate media, or make a filename safe by itself. Validate decoded image data and accepted size separately, scope credentials to the smallest required namespace, and ensure the read path does not reveal existence across tenants.

Keep the read path thin. Authorization lookup, capability issuance, byte delivery, and first-byte latency are different measurements; combining them into one average hides the slow path that users experience after a deployment or a cold cache. Put asset identifiers in structured logs and traces rather than metric labels, because object keys create unbounded metric cardinality.

## Recovery drills decide whether the design is durable

A database backup and an object inventory are not automatically a recoverable catalog. The recovery procedure needs to identify which metadata snapshot pairs with which object namespace, how ready rows whose objects are absent are quarantined, how unreferenced objects age before removal, and who approves an irreversible purge. The test should begin with a known catalog: create a pending asset, make it ready, create a second pending asset, and schedule a deletion. Restore the database and object namespace according to the proposed runbook, then assert the expected state for every record instead of accepting that each backup tool completed. A ready row without its bytes must be quarantined rather than served. An object without a row needs an aging policy, because immediate deletion can destroy a write that was acknowledged before metadata committed. The deletion state must prevent a late retry from reviving an asset the retention process has already selected. These details feel procedural until a restore combines snapshots taken at different times; then they are the actual definition of durability. Test termination at each write boundary: after pending creation, after object acknowledgement, and after the ready commit. Then exercise retry, duplicate delivery, deletion, and restore with the same state-machine assertions.

Measure pending age, ready-transition count, checksum mismatches, orphan candidates, deletion backlog, bytes written, bytes served, and first-byte latency by object-size band. Disk fills quickly. Database backup growth can be just as surprising, because every replica and restore path inherits the byte volume. Those are capacity planning facts, not footnotes.

The rejected default is putting all generated images in a database blob column merely because the first implementation has one connection and one migration. It remains a valid exception for bounded attachments that genuinely need the database transaction, but write down the ceiling and the migration trigger before traffic establishes a much larger, accidental contract. The same discipline rules out local disk as the authoritative store for SaaS assets while preserving its proper role as transient workspace.

## References

- https://developer.mozilla.org/en-US/docs/Web/HTTP/Reference/Headers/Content-Disposition
- https://supabase.com/docs/guides/storage
