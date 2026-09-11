# Small SaaS Image Thumbnail Generation: Object Storage or CDN in US/EU Healthtech

Short answer: for a small US/EU healthtech SaaS storing training artifacts, keep originals in private object storage, materialize a small set of thumbnail sizes with an asynchronous worker, and make tenant identity part of every key and authorization check. Use an image CDN when arbitrary transforms at the edge are a product requirement; resize synchronously during upload only when image size and upload concurrency are tightly bounded. Isolation comes first.

This is a runbook decision, not a diagram contest.\n\nNo shortcut.\n\nI've found that the first useful review question is not which delivery service has the nicest demo; it is whether an engineer can name the tenant boundary at every transition, from upload authorization through queue claim, source read, derivative write, cache response, and deletion. That sequence is long enough to expose the real design: a thumbnail is a derived copy of a tenant's artifact, and a cache key, log line, presigned URL, or retry queue can become an accidental cross-tenant boundary.\n\nA good architecture makes the wrong request fail early. A thumbnail is still a derived copy of a tenant's artifact, and a cache key, log line, presigned URL, or retry queue can become an accidental cross-tenant boundary. Set the first SLO around isolation and artifact correctness, then set an upload-to-ready objective for derivatives. The exact latency target belongs to the service budget; I am not assuming one here.

Keep the read path boring.

## The first isolation test for a training artifact

Start with a tenant-scoped record in the application database. It should contain the tenant ID, artifact ID, original object key, derivative policy version, processing state, and retention class. A key such as originals/{tenant-id}/{artifact-id}/source and a derivative such as thumbnails/{tenant-id}/{artifact-id}/{policy-version}/320 make ownership inspectable. They are not authorization. Every read and write still needs an application-side tenant check, and a worker must carry tenant context from the claimed job through the object request.

The upload transaction should create the artifact record before accepting a completion event. A worker claims the job, reads the private original, validates media type and decoded dimensions, generates the approved derivative, writes the deterministic key, and marks that exact policy version ready only after the write succeeds. A repeated job converges on the same object. It does not create a new filename merely because the queue delivered a duplicate.

For US and EU tenants, keep placement explicit. Select the storage region from tenant policy, keep the region in the record, and test that a URL issued for one tenant cannot be replayed against another tenant's object. Geographic placement reduces one class of residency risk; it does not prove isolation. Authorization and retention rules handle the rest.

## How does image thumbnail generation change an object storage architecture for a small SaaS?

Object storage plus a worker is the most inspectable default when the product has fixed sizes such as 160, 320, and 640 pixels. CPU is spent once per upload; reads then serve already-created objects. Queue depth, oldest pending job, derivative age, and failed-job count are useful operational signals. A failed thumbnail does not have to make the original unavailable.

An image CDN makes sense when the product genuinely needs arbitrary crops, responsive widths, format negotiation, or transformation at request time. Its cache and URL policy become part of the serving SLO, and a cache miss can invoke work on the read path. That trade is reasonable when presentation flexibility matters more than a small fixed derivative catalog. It is a poor default when the team has not decided which transforms are allowed, how private authorization reaches the edge, or how transformed copies expire.

Synchronous server resize has fewer moving parts, but its worst image becomes part of upload latency and API capacity. It can fit a low-volume service with a strict maximum input size and enough CPU headroom. It is not suitable when uploads can be large or concurrent enough that a resize stall consumes the request budget.

| Approach | Good fit | Capacity signal | Main trade-off |
| --- | --- | --- | --- |
| Private storage plus worker | Fixed sizes and asynchronous readiness | Queue age, worker CPU, failed jobs | No arbitrary edge transform |
| Image CDN | Dynamic crops, widths, and formats | Cache misses, transform latency, origin load | URL and cache policy join the product SLO |
| Resize on upload | Small inputs and low concurrency | Upload p95/p99, CPU headroom | Image work delays the response |

The buy-versus-build boundary is operational. A managed image tier buys transformation behavior and maintenance capacity; a worker pipeline buys replayability and a narrow failure domain. Neither removes authorization, retention, and observability work.

## Capacity signals before implementation

The code below is provider-neutral. The important part is the order of checks and state transitions, not a particular storage SDK. The worker receives a claimed job from a durable queue, refuses a tenant mismatch, writes a fixed derivative key, and makes readiness visible only after the object write returns successfully.

```go
package thumbnail

import (
    "context"
    "fmt"
    "io"
)

type Job struct {
    TenantID string
    ArtifactID string
    SourceKey string
    PolicyVersion string
    Width int
}

type ObjectStore interface {
    Get(context.Context, string) (io.ReadCloser, error)
    Put(context.Context, string, io.Reader, string) error
}

type ArtifactState interface {
    TenantOwns(context.Context, string, string) (bool, error)
    MarkReady(context.Context, string, string, string, string) error
    MarkFailed(context.Context, string, string, error) error
}

func Process(ctx context.Context, store ObjectStore, states ArtifactState, job Job) error {
    owned, err := states.TenantOwns(ctx, job.TenantID, job.ArtifactID)
    if err != nil { return err }
    if !owned { return fmt.Errorf("artifact is outside the tenant boundary") }
    source, err := store.Get(ctx, job.SourceKey)
    if err != nil { return states.MarkFailed(ctx, job.TenantID, job.ArtifactID, err) }
    defer source.Close()
    thumbnail, contentType, err := DecodeResize(source, job.Width)
    if err != nil { return states.MarkFailed(ctx, job.TenantID, job.ArtifactID, err) }
    key := fmt.Sprintf("thumbnails/%s/%s/%s/%d", job.TenantID, job.ArtifactID, job.PolicyVersion, job.Width)
    if err := store.Put(ctx, key, thumbnail, contentType); err != nil {
        return states.MarkFailed(ctx, job.TenantID, job.ArtifactID, err)
    }
    return states.MarkReady(ctx, job.TenantID, job.ArtifactID, key, job.PolicyVersion)
}
```

DecodeResize stands for the image library and validation policy selected by the service. Its contract should reject unsafe dimensions, enforce a maximum decoded size, and return a bounded stream or buffer suitable for the storage client. Keep that policy versioned. A future crop rule should create a new derivative namespace rather than silently changing an object that an audit or training job expects to reproduce.

One subtle point matters in reviews: a deterministic key makes retries idempotent, but it does not serialize two different policy versions or prove that a tenant is entitled to the source. Those guarantees belong in the database state machine and request authorization.

## The rollback drill and retention ledger

Verify the ordinary path from each supported region: create an artifact for tenant A, upload a source, observe a claimed job, fetch the ready derivative, and confirm its dimensions and content type. Repeat the read using tenant B credentials and expect an authorization denial. Test direct object access separately from the application endpoint; passing one test does not prove the other boundary.

Then stop the worker consumer. Existing originals must remain readable, while the artifact stays pending and the oldest-pending metric advances. Restart the previous worker version and replay the pending job. The fixed key should give the same intended derivative, and a second delivery should not add another database record. Record the policy version in logs, but never put source bytes or bearer credentials there.

Retention is part of capacity planning. Apply lifecycle rules to originals and derivatives according to the tenant retention class, and make deletion observable across the database and both object namespaces. Do not rely on a prefix alone as the searchable index; support staff need an authorization-aware artifact record that explains what exists, where it lives, and when it expires. Test abandoned multipart uploads and incomplete jobs, since they can consume storage without becoming visible thumbnails.

The catch is that this fixed-derivative design is not suitable for a public gallery needing unlimited URL transforms, a product requiring edge negotiation for every request, or a team that cannot operate a queue and replay path. Choose a dedicated image delivery layer for that workload. Conversely, do not introduce an image CDN merely to avoid defining tenant-scoped keys and retention; it relocates those responsibilities and adds cache invalidation to them.

Rollback should be boring: stop new claims, deploy the last worker, replay records whose readiness transition did not complete, and compare the derivative policy version before marking anything ready. Preserve the original until the new derivative has passed verification. That gives the platform team a recoverable path without making the upload API responsible for every image-processing failure.

## References

- https://docs.aws.amazon.com/AmazonS3/latest/userguide/object-lifecycle-mgmt.html
- https://cloud.google.com/storage/docs
- https://www.rfc-editor.org/rfc/rfc9110

## Further reading

- https://docs.aws.amazon.com/AmazonS3/latest/userguide/object-lifecycle-mgmt.html
- https://cloud.google.com/storage/docs
- https://www.rfc-editor.org/rfc/rfc9110
