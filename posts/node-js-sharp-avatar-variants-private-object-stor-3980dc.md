# Node.js Sharp Avatar Variants: Private Object Storage and Original Retention

Keep the original avatar as an immutable private object, generate a small, explicit set of Sharp derivatives, and treat the derivative manifest as the contract your application serves. That shape makes a thumbnail crop or resize repeatable, keeps a bad transformation from destroying source data, and gives operations a clean way to measure queue lag and read latency.

**Short answer:** save one original, create versioned square and display-size variants asynchronously, and return an authorized response rather than exposing the bucket path. The exact sizes are a product decision; the immutability, authorization, and retry behavior are operational decisions.

## What failure does this layout prevent?

The expensive failure is not a slightly soft thumbnail. It is losing the only high-resolution input, then discovering that every future crop is constrained by yesterday's UI. A second failure mode is letting clients write arbitrary keys such as `avatars/user-7.png`, which makes replacement, cache invalidation, and incident cleanup depend on whatever a client happened to invent.

Use a server-owned object key with a content or profile revision in it. A typical family might be `avatars/{account}/{revision}/original` and `avatars/{account}/{revision}/square-128.webp`. The names are illustrative, not an API requirement. Store width, height, media type, byte length, checksum, and the transformation revision beside the objects. A row or document containing that manifest lets the API answer “which image is current?” without listing a bucket on every request.

The original should not be overwritten during a profile update. Mark the old revision inactive after the new manifest passes verification, then garbage-collect it according to your retention policy. That ordering gives rollback a real input instead of a hope that an image cache still contains the previous file.

Keep it boring.

## Should Node.js Sharp crop and resize avatars before private object storage saves the original?

No. Upload the original once, validate it, and derive the requested sizes from that stored source. Sharp is a reasonable implementation choice in a Node.js worker because the transformation is deterministic when its options and library version are pinned, but the durable rule is independent of the image library: the source is the recovery artifact, while thumbnails are replaceable outputs.

Do not let a browser decide arbitrary dimensions. Define a short allowlist such as `square-128`, `square-256`, and `profile-1024`; each name maps to a crop, fit, encoder, and quality policy in code. This bounds CPU and memory work and makes capacity planning possible. If a new client asks for an unlisted size, return a validation response or route it through an explicitly reviewed configuration change.

The buy-versus-build choice is mostly about ownership of those boundaries:

| Choice | Good fit | Cost to carry |
| --- | --- | --- |
| Managed image pipeline | A small team with an open-ended size matrix | Less control over transformation versions and retention |
| In-house worker and object store | A platform team that needs deterministic keys and SLOs | Queue operations, capacity planning, and patching the image stack |

Your mileage may vary when animated or unusually encoded inputs dominate; measure representative files before promising a transform SLO.

Here is the storage-worker shape in Go. The image processor is an interface so the same queue can wrap Sharp in a Node.js process; the example concentrates on ordering, keys, and verification rather than pretending that a storage SDK is portable. In production I would also record the input checksum and transformation revision before enqueueing, keep a bounded worker pool, and attach a deadline to every object write. That extra bookkeeping looks fussy until a retry overlaps a profile update: without it, an older job can publish a newer-looking key and quietly move the manifest backward. The queue should expose age and failure counts, and the worker should distinguish invalid input from a transient store timeout so the retry policy does not turn a bad upload into an endless hot loop.

```go
package avatars

import (
	"context"
	"fmt"
)

type ObjectStore interface {
	Put(ctx context.Context, key string, data []byte, contentType string) error
}

type Processor interface {
	Derivative(ctx context.Context, original []byte, variant string) ([]byte, error)
}

func SaveAvatar(ctx context.Context, store ObjectStore, processor Processor, account, revision string, original []byte) error {
	originalKey := fmt.Sprintf("avatars/%s/%s/original", account, revision)
	if err := store.Put(ctx, originalKey, original, "application/octet-stream"); err != nil {
		return err
	}

	for _, variant := range []struct {
		name        string
		contentType string
	}{
		{"square-128", "image/webp"},
		{"square-256", "image/webp"},
		{"profile-1024", "image/webp"},
	} {
		data, err := processor.Derivative(ctx, original, variant.name)
		if err != nil {
			return fmt.Errorf("derive %s: %w", variant.name, err)
		}
		key := fmt.Sprintf("avatars/%s/%s/%s", account, revision, variant.name)
		if err := store.Put(ctx, key, data, variant.contentType); err != nil {
			return fmt.Errorf("store %s: %w", variant.name, err)
		}
	}
	return nil
}
```

In a real service, persist the manifest only after all required puts have succeeded and a read-after-write check confirms the metadata you depend on. A failed derivative is a failed job, not permission to publish a half-populated revision. Retries must be idempotent: the same account and revision should address the same keys, and a worker should be able to overwrite a derivative without touching the original.

## How should private reads, content headers, and caches be verified?

Keep the bucket or object store private and put authorization at the application or signed-request boundary. On each read, check that the caller may see the account's current revision, then either stream the object through the service or issue a narrowly scoped, short-lived download URL. Never use an object key as an authorization decision.

Set the response media type from the verified manifest, not from a user-supplied filename. `Content-Disposition: inline` is appropriate when the browser should render an image; `attachment` asks it to download instead, and the `filename` parameter is only a presentation hint. Sanitize any displayed filename and do not reflect path separators or control characters into a header. MDN documents these semantics and the quoting rules.

For immutable revisioned derivatives, a long cache lifetime and an `ETag` are useful. For the manifest endpoint, use a shorter freshness policy because it points clients at the current revision. This split avoids purging every thumbnail when a user changes an avatar: the new revision naturally has new URLs.

## Verification and rollback before widening traffic

Exercise the runbook with a fixture set that includes a huge image, a very small image, an animated input, a corrupt byte stream, and an image with an unusual orientation. Verify that validation rejects unsupported media before expensive work, that every declared variant has the expected dimensions, and that a retry leaves exactly one object per deterministic key. Measure p95 transform time, bytes written per avatar, queue age, and authorized-read latency against an SLO you can staff.

The catch is storage multiplication: three derivatives plus an original cost more bytes and more background work than on-demand resizing. Precompute only sizes with a known read path; choose on-demand generation when the size matrix is genuinely open-ended and you can absorb cold-request latency. Keep a previous revision until the new one is verified and observable, then switch the manifest pointer. If the new output is wrong, point the manifest back to the previous revision and stop publishing new jobs; the immutable original makes that rollback reversible.

This pattern is unsuitable when images are disposable, access is intentionally public, or a separate media pipeline already owns transformations and retention. In those cases, use that pipeline's contract and keep your application responsible for authorization and metadata rather than duplicating workers.

## References

- https://developer.mozilla.org/en-US/docs/Web/HTTP/Reference/Headers/Content-Disposition
- https://firebase.google.com/docs/storage
