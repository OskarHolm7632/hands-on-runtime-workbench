# User Avatar Upload: Signed Object Storage URL or Node.js Server Proxy?

For a beginner SaaS, route user avatar uploads through the application first when the files are small and the team needs one obvious validation boundary; use a signed object-storage URL once carrying upload bytes through the Node.js service has become a measured operational constraint. The deciding constraint is not the URL format. It is whether the system can prove which authenticated user owns a stored object, which validation happened, and how an interrupted upload stops becoming permanent data.

This is an architecture decision record for a profile-image path. The byte path can change later; ownership and publication rules should not. Small files are forgiving until retries, abandoned forms, caches, and cleanup turn a single `POST` into several independently failing operations.

## Should a beginner SaaS use signed URLs, object storage, or a server proxy for user avatar uploads?

Start by distinguishing a storage authorization from a completed avatar. A server relay authenticates the browser, accepts bytes, applies policy, writes the object, and records the new profile reference in one request-shaped workflow. A signed URL removes the application service from the byte path: the service creates a narrowly scoped upload intent, while the browser writes directly to object storage and later asks the application to finalize that intent.

Neither option removes the storage write/database-update boundary. A client can disconnect after an object is stored; a retry can arrive after the profile is already updated; a user can leave the form. The direct path makes those gaps more visible, which is useful only when the team has an explicit state model and a cleanup owner.

| Decision factor | Application relay | Signed direct upload |
| --- | --- | --- |
| Byte path | Browser to application to storage | Browser to storage |
| Authorization point | Request handler | Upload-intent creation |
| Application capacity | Carries every upload byte | Carries control messages only |
| Primary failure boundary | Long request and downstream write | Intent, write, and finalize are separate |
| Good initial fit | Small avatars and simple inspection policy | Upload traffic or slow clients that must not occupy API capacity |
| Cost of getting it wrong | Request limits and workers become a file service | Orphaned objects or premature publication without reconciliation |

The catch is operational. A proxy is not suitable when normal API workers, ingress body limits, or client connection duration are already constrained by real upload traffic. Direct upload is not suitable when nobody owns pending records, validates stored candidates, or can reconcile incomplete work. Pick the failure mode the team can observe and repair, not the one that looks shorter in a diagram.

## The invariants matter more than the byte path

An avatar should belong to one authenticated account, and the client should never supply an unrestricted final object key. A profile should refer only to a candidate that has passed the application's validation policy. Finally, an upload intent that never reaches publication must have an expiration path. Those four invariants turn vague cleanup advice into records that can be queried and repaired.

Use a state such as `pending -> ready -> replaced`, with `abandoned` as a cleanup outcome. A signed flow creates the pending record before issuing permission for a single proposed key. The finalization endpoint then checks the requester, reads the candidate's metadata through the storage abstraction, enforces the policy, and publishes the profile reference. Retrying finalization for the same upload identifier should return the already-published result rather than create a second avatar.

The state record needs enough detail to answer a difficult support question without reconstructing browser behavior from scattered logs: who created the intent, which destination key was authorized, which policy version applied, when the intent expires, and whether finalization has already committed. The finalization operation should take a stable upload identifier rather than infer intent from a filename, because filenames are mutable user input and two browser retries can legitimately carry the same name. It should also be idempotent at the database boundary: if a request repeats after publication, return the established avatar reference; if it repeats while a prior finalization is active, serialize that transition rather than publish competing keys. The storage adapter may expose a candidate object before the application accepts it, but that object stays non-public until the profile record changes. This distinction is where direct upload earns its extra ceremony. Without it, a permission grant gets mistaken for a completed change, cleanup cannot tell an abandoned candidate from a slow client, and a profile update can point at an object whose size or type was never checked. Don't conflate storage metadata with authorization either: metadata helps enforce policy, while the pending record proves the relationship between the account and the proposed object.

Do not trust a client-provided media type as proof of file contents. It is an input to a policy decision, not an inspection result. The exact size ceiling and accepted formats are product policy, so a beginner service can choose a conservative limit and revise it after observing legitimate use.

Short failures help. A direct upload that never finalizes should be visible as an aging pending record, not as a mysterious absence from a profile page.

No hidden magic.

For delivery, choose `Content-Disposition` deliberately. `inline` describes content intended for browser display; `attachment` describes a download prompt and can include a filename. An avatar commonly needs inline behavior, but access-control policy and the origin used for untrusted content deserve a separate decision from upload transport.

## A transport-neutral critical path

The following Python sketch keeps identity, retry behavior, and publication in domain code while leaving the storage adapter to implement either a signed handoff or an application relay. The function names are intentionally generic: a different object-store implementation should not rewrite the profile state machine.

```python
from dataclasses import dataclass
from typing import Protocol
from uuid import UUID, uuid4


@dataclass(frozen=True)
class StoredObject:
    key: str
    size: int
    media_type: str


class ObjectStorage(Protocol):
    def authorize_upload(self, key: str, media_type: str) -> str: ...
    def inspect(self, key: str) -> StoredObject: ...


class UploadRecords(Protocol):
    def create_pending(self, upload_id: UUID, user_id: UUID, key: str) -> None: ...
    def owner_and_key(self, upload_id: UUID) -> tuple[UUID, str]: ...
    def publish_avatar(self, upload_id: UUID, user_id: UUID, key: str) -> None: ...


ALLOWED_TYPES = {"image/jpeg", "image/png", "image/webp"}
MAX_AVATAR_BYTES = 2 * 1024 * 1024


def begin_avatar_upload(
    user_id: UUID, media_type: str, storage: ObjectStorage, records: UploadRecords
) -> tuple[UUID, str]:
    if media_type not in ALLOWED_TYPES:
        raise ValueError("unsupported avatar media type")
    upload_id = uuid4()
    key = f"pending/{user_id}/{upload_id}"
    records.create_pending(upload_id, user_id, key)
    return upload_id, storage.authorize_upload(key, media_type)


def finalize_avatar(
    upload_id: UUID, user_id: UUID, storage: ObjectStorage, records: UploadRecords
) -> str:
    owner, key = records.owner_and_key(upload_id)
    if owner != user_id:
        raise PermissionError("upload owner mismatch")
    candidate = storage.inspect(key)
    if candidate.media_type not in ALLOWED_TYPES or not 0 < candidate.size <= MAX_AVATAR_BYTES:
        raise ValueError("stored avatar failed validation")
    records.publish_avatar(upload_id, user_id, candidate.key)
    return candidate.key
```

The example's 2 MiB limit is illustrative, not a universal recommendation. A production implementation should inspect bytes according to its security policy, preserve idempotency for `publish_avatar`, and avoid logging credential-bearing signed query strings. Record the upload identifier, account identifier, proposed key, state transition, and policy result instead. Those fields make the control path observable without placing secrets in logs.

## Rejected default and lifecycle backstop

The rejected default is a signed direct-upload path for the first version of a small-avatar product. Its valid use case begins when uploads demonstrably compete with ordinary requests, when mobile connections make long application requests impractical, or when upload capacity needs independent scaling. In that case, retain the same pending/finalize contract and replace only the byte transport.

Lifecycle rules are the backstop, not the transaction. Keep pending candidates under a separate prefix and expire them after a documented retry window; apply a different retention rule to objects currently referenced by profiles. Object lifecycle management can define actions such as expiration over groups of objects, while a scheduled reconciliation job compares old pending records, profile references, and stored keys before making any conservative cleanup decision.

Versioned avatar keys simplify replacement. Writing a new key lets the profile reference move after validation and gives caches an unambiguous object version. It temporarily duplicates data and requires a retention policy for the previous image. That is an acceptable trade-off for many services, though a product with strict retention requirements should define the deletion window before publishing the feature.

The useful conclusion is narrow: adopt the transport whose failure boundaries fit current operations, then preserve a state machine that allows the other transport later. A signed URL is an authorization mechanism, not a completion record; a proxy is a convenience boundary, not a durability guarantee.

## References

- [MDN: Content-Disposition response header](https://developer.mozilla.org/en-US/docs/Web/HTTP/Reference/Headers/Content-Disposition)
- [Amazon S3: Object lifecycle management](https://docs.aws.amazon.com/AmazonS3/latest/userguide/object-lifecycle-mgmt.html)
