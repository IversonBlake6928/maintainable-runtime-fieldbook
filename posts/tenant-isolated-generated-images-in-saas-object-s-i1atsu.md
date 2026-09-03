# Tenant-Isolated Generated Images in SaaS: Object Storage, Database Blobs, or Local Disk

Use private object storage for the image bytes, a relational database for ownership and lifecycle metadata, and local disk only for bounded processing. For a customer-support SaaS that accepts private uploads and serves expiring download links, tenant isolation is the deciding constraint: an image must be addressable by the right tenant, unavailable to the wrong one, and recoverable when an application instance disappears.

Short answer: object storage plus database metadata is the least surprising production boundary for AI-generated images; database blobs are reasonable for a genuinely small, transaction-heavy workload, and local disk belongs in the temporary-work category.

The recommendation is deliberately operational. A preview that works on one laptop proves almost nothing about a service with several workers, retries, retention rules, and a recovery SLO. The storage choice has to survive those conditions before it deserves a place in the roadmap.

## How should SaaS teams store generated images with object storage or database blobs?

Start with the record, not the bucket. The database row should identify the tenant, generation or upload request, media type, object key, lifecycle state, creation time, and any retention deadline. The object store should hold the bytes under a private, tenant-scoped key. The application checks authorization against the row before it creates a short-lived download response.

That split gives each system one job. Queries, ownership, and state transitions remain relational; large binary payloads do not inflate every database replica and backup. Local disk can hold a decoded image while a worker validates or uploads it, but a file that must outlive a process cannot be considered durable merely because the process has not restarted yet.

Keep it private.

The link-issuing path is where a plausible storage design often loses tenant isolation. A request arrives with a media identifier, the handler loads the row, and an implementation that treats the identifier as sufficient can accidentally skip the tenant predicate before generating a URL. The safer sequence is explicit: authenticate the caller, load the record by both tenant ID and media ID, confirm the record is ready and within its retention window, then ask the storage adapter for a short-lived download. If any check fails, return the same externally observable authorization response used for an unknown resource; do not reveal whether another tenant's object exists. Record the decision without logging the signed URL or the image bytes. This is a small amount of code, but it is the part that determines whether an object store is a private media system or merely a second place where data can leak. The storage key helps with organization; it is not an authorization token, and a prefix is not a security boundary by itself.

Database blobs have a legitimate use case. If the payload count is tightly bounded, the team needs one transaction for metadata and bytes, and restore time remains inside the SLO, keeping the blob with the row can remove an integration boundary. “Small” needs a number, though: estimate images per day, retained days, mean and maximum byte size, replica count, backup copies, and restore throughput. I would not approve a blob design from a vague statement that the workload is small.

| Decision | Buy | Build or operate | The trade-off |
|---|---|---|---|
| Object storage | A managed durability and access boundary | Key naming, authorization, lifecycle workers, and reconciliation | Less binary-data plumbing in the database, more distributed state to observe |
| Database blob | Existing database capacity and backup workflow | Size limits, migration behavior, and restore tests | A simpler write transaction, but heavier replication and recovery |
| Local disk | Fast temporary workspace already attached to a worker | Durability, sharing, and recovery | Useful for processing; unsafe as the system of record |

This is a buy-vs-build decision in disguise. Buying durable bytes does not buy tenant authorization or a deletion policy; building those policies around a local filesystem does not make the filesystem durable.

## The failure mode is usually identity, not image format

The dangerous object key is `latest.png`. It is easy to guess, easy to overwrite, and easy to detach from the tenant record when a retry races with the first request. Use an immutable key derived from a server-owned tenant identifier and a stable request identifier, such as `tenant-42/generation-781/image.png`. Do not derive authorization from a filename supplied by a browser or a model response.

The write state should make partial progress visible. Create the row as pending, reserve the key, upload the bytes, then mark the row ready. A worker that stops between those operations leaves evidence for reconciliation instead of presenting an object as customer-visible before the application has recorded it. A retry reuses the same request identity; it should not create an anonymous second image simply because the first worker lost its connection after the upload.

Here is the shape I want at the adapter boundary. The storage implementation is deliberately boring: it accepts an already-authorized key and returns only an upload result. Policy stays above it.

```go
package media

import (
	"context"
	"fmt"
)

type ObjectStore interface {
	Put(ctx context.Context, key string, content []byte, contentType string) error
	Delete(ctx context.Context, key string) error
	SignedDownload(ctx context.Context, key string, expiresInSeconds int) (string, error)
}

func StoreReadyImage(ctx context.Context, store ObjectStore, tenantID, generationID string, content []byte) (string, error) {
	if tenantID == "" || generationID == "" {
		return "", fmt.Errorf("tenant and generation identifiers are required")
	}
	key := fmt.Sprintf("%s/%s/image", tenantID, generationID)
	if err := store.Put(ctx, key, content, "image/png"); err != nil {
		return "", err
	}
	return key, nil
}
```

The surrounding transaction still has work to do: it must verify that the generation belongs to the authenticated tenant, persist the returned key, and advance the row only after the upload succeeds. The example does not pretend those operations are atomic. They are not.

For a private support attachment, the download endpoint should authorize the row and then issue a temporary URL or stream the object through the application. Set an explicit download filename and disposition where appropriate; the HTTP `Content-Disposition` header defines whether a response is handled inline or as an attachment, and it also carries the filename contract that clients see.

## What should the runbook verify before production?

Test the boundary with two tenants, two application replicas, a worker restart, and a repeated request. Tenant A must never receive a link for Tenant B's key, even when the request IDs are similar. A ready record must remain retrievable after the worker that uploaded it is replaced. A pending record must remain invisible until its object and metadata satisfy the readiness rule.

Then test the awkward timing windows. Stop a worker after the object write but before the database update. Stop it before the write. Retry both cases. The reconciliation job should find orphaned objects and incomplete rows using the same request identity, apply the retention policy, and emit an auditable result. Do not silently delete an object just because a client request timed out; the timeout says something about the caller's observation, not necessarily about the server's write.

Capacity planning belongs in this test plan. Track byte growth and object count separately, reserve room for retries and temporary files, and measure restore throughput rather than assuming backup completion equals recovery readiness. Define the image-read latency and availability targets from the product SLO. I'm not sure a universal threshold would be honest here; the missing inputs are tenant count, image dimensions, download volume, and the recovery objective.

Keep one metric for authorization denials, one for expired-link requests, one for upload-to-ready duration, and one for reconciliation age. A spike in 403 responses can indicate an authorization regression; a growing reconciliation queue can indicate a broken state transition even while image previews still look fine.

## When is another storage choice more appropriate?

The catch is that object storage adds a second system whose lifecycle must be understood. If the application requires a single atomic commit for a very small number of images and its database backup and restore budget can carry the bytes, a database blob may be the more honest design. Recheck that decision whenever image dimensions, retention, or tenant count changes.

Local disk is suitable for development fixtures, image decoding, virus scanning, and upload staging. It is not suitable when another replica must read the file, when a deployment must preserve it, or when a customer expects an expiring link to remain valid after the worker is replaced. Stick with local disk only when the file is explicitly disposable and the durable write happens before the job reports success.

Object storage is also a poor fit if the required service has no acceptable private-access, deletion, retention, or recovery semantics. Choose a different managed service or operate a different backend when those are hard requirements; do not hide the mismatch behind a longer application wrapper. The abstraction should make a backend replaceable, not make incompatible guarantees look identical.

## Roll back without losing ownership evidence

Before changing the adapter, add an owner field to each media record and keep the old read path available. Write a small set of test images for each tenant, verify authorization and expiration, and compare the resulting metadata before switching new writes. Avoid casual dual writes: two successful uploads do not form one transaction, and the system still needs one canonical object.

If verification fails, stop new writes to the candidate backend, return writes to the known path, and preserve the request IDs, tenant IDs, keys, and state transitions for records already created. Reconcile before deleting anything. The rollback is complete when the application can explain every ready row, every retained object, and every object scheduled for removal.

Three words: prove ownership first.

## References

- https://developer.mozilla.org/en-US/docs/Web/HTTP/Reference/Headers/Content-Disposition
- https://supabase.com/docs/guides/storage
