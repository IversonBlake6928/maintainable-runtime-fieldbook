# Upload Time Image Resize APIs Beat On Demand Processing for User Avatars

Process and moderate user avatars at upload time, before they can be published; reserve on-demand resizing for derivative sizes that are non-public and reproducible. **The deciding constraint is the publication boundary:** a B2B SaaS profile image must not become visible while its validation or moderation state is unknown. A hosted resize API lets a Node.js application run without Sharp or ImageMagick in its deployment artifact, but the operational win comes from making one asynchronous decision durable, observable, and reversible rather than asking every read to make it again. Removing those native dependencies does not remove the need to own state.

Short answer: accept an original into private storage, create a normalized candidate through a provider-neutral processing interface, moderate that candidate, and atomically publish an immutable asset reference only after every gate passes. This adds queueing and state management to the write path. It also gives support and on-call engineers one answer to the important question: which reviewed bytes are users seeing?

## Should a user avatar image resize API run during upload?

On-demand resizing looks smaller on a diagram. The application stores an upload, then a URL or edge request asks an image service for the required dimensions. For public, unmoderated avatars, however, the first read may become the event that creates and exposes a derivative. A cache miss is no longer merely a latency concern; it sits beside a content decision whose result needs a durable record.

The limitation of upload-time processing is real: users wait for a queue before a replacement appears, the platform must retain job state, and a burst of uploads demands worker capacity even when nobody views the resulting profiles. For an internal directory where images are already approved and rarely viewed, on-demand generation may be the better trade-off. For moderated public avatars, I would pay the write-path complexity because it removes transformation and moderation from cache-miss traffic.

The failure modes cross ownership boundaries. A profile update can succeed while transformation is still pending. Two concurrent reads can request different results if transformation policy changes between them. A moderator can reject an original after an older derivative has already been cached under a mutable URL. Retries can amplify work unless requests carry stable identifiers. None of those problems is fixed by choosing a faster resize endpoint.

Upload-time processing makes five state transitions explicit: `received`, `processing`, `approved`, `rejected`, or `failed`. Only `approved` may populate the public avatar reference. Keep the old approved avatar until the replacement reaches that state; a new upload should not turn an existing profile into a broken image. This is more machinery than a direct image resize call, and small teams should count the queue, database transitions, retention work, and on-call ownership before calling the hosted API the simplest option; the API invocation may be simple while the publication workflow is not.

That rule is boring. Good.

## Choose the write path with an SLO budget

The two designs spend reliability budget in different places. I would choose upload-time moderation for this workload because publication correctness matters more than immediate replacement, while ordinary avatar reads should remain cacheable and independent of a transformation control plane.

| Decision | Upload-time moderation and resize | On-demand resize |
| --- | --- | --- |
| Public visibility | Changes after a recorded approval | May depend on behavior at first read |
| Read-path dependency | Static approved object | Transformation service on cache miss |
| Policy changes | Reprocess, review, then repoint | Can alter generated derivatives unless versions are pinned |
| Rollback | Restore the prior approved reference | Purge or restore aliases and derived caches |
| Capacity signal | Upload and queue arrival rate | Read rate, cache misses, and requested variants |
| Best boundary | Moderated identity images | Private or low-risk derivatives from an approved source |

Capacity planning should start with arrivals, not monthly active users. Record uploads per minute at the peak, the distribution of input bytes, processing duration, retry rate, and queue age. Then set two separate objectives: time from accepted upload to a terminal moderation state, and availability of the last approved avatar. Combining them hides a useful distinction; processing can be degraded while previously published images continue serving normally. Ten derivative sizes do not mean ten times the upload rate, yet they can approach ten times the transformation work if the processor decodes and encodes each variant separately, so measure work per source rather than assuming a request count tells the capacity story.

Do not promise synchronous completion merely because most images finish quickly. Return an upload identifier and a non-public status, process asynchronously, and let the profile continue to reference its last approved version. A bounded deadline should move the job to a terminal failure state that a user can retry, rather than leaving `processing` forever.

## Make the implementation idempotent and replaceable

The application owns lifecycle state. Object storage owns bytes. A resize or moderation service performs a bounded operation, but it should not decide which avatar is public. That separation keeps a vendor outage or replacement from rewriting account semantics.

Use a generated upload ID, not the original filename, as the idempotency key. Store the source privately, and treat transformed output as a new immutable object whose key includes a policy version. Verify the declared type against the decoded media, reject unsupported formats, impose byte and pixel limits before expensive work, strip unnecessary metadata during normalization, and never construct a public URL from user-controlled path text. MDN's image format guide is a useful format reference, but browser support is not an upload acceptance policy; define a narrower set that the entire processing chain can decode consistently.

The core needs only a small interface. This Go example deliberately leaves transport and vendor details outside the state machine:

```go
package avatar

import (
	"context"
	"errors"
)

type State string

const (
	Processing State = "processing"
	Approved   State = "approved"
	Rejected   State = "rejected"
)

type Result struct {
	ObjectKey     string
	PolicyVersion string
	Width         int
	Height        int
	Approved      bool
}

type Processor interface {
	NormalizeAndModerate(ctx context.Context, privateSourceKey, idempotencyKey string) (Result, error)
}

type Repository interface {
	Begin(ctx context.Context, accountID, uploadID string) (bool, error)
	Approve(ctx context.Context, accountID, uploadID string, result Result) error
	Reject(ctx context.Context, accountID, uploadID, reason string) error
}

func Handle(ctx context.Context, repo Repository, processor Processor, accountID, uploadID, sourceKey string) error {
	started, err := repo.Begin(ctx, accountID, uploadID)
	if err != nil || !started {
		return err
	}

	result, err := processor.NormalizeAndModerate(ctx, sourceKey, uploadID)
	if err != nil {
		return err // The queue applies bounded retry policy.
	}
	if !result.Approved {
		return repo.Reject(ctx, accountID, uploadID, "policy")
	}
	if result.ObjectKey == "" || result.Width < 1 || result.Height < 1 {
		return errors.New("invalid processor result")
	}
	return repo.Approve(ctx, accountID, uploadID, result)
}
```

`Approve` must compare the active upload ID before changing the public pointer. Otherwise a slow, older job can overwrite a newer avatar. This stale-completion race is easy to miss in a happy-path integration test and is exactly why the publication decision belongs in one transactional boundary.

Retries need limits and classification. Timeouts and temporary upstream unavailability may be retryable; invalid media and policy rejection are terminal. Preserve the original privately according to a declared retention policy, and avoid logging image bytes, signed URLs, or user-supplied metadata. The useful log fields are the upload ID, account ID, policy version, attempt number, state transition, duration, and a low-cardinality outcome.

## Verify the gate before increasing traffic

Test the state machine more aggressively than the image cosmetics. A test corpus should cover every accepted format, malformed headers, oversized dimensions, truncated files, animated inputs if they are allowed, metadata orientation, transparency, and files whose extension disagrees with their content. The expected result for every case is a terminal state plus a known publication outcome, not merely “the API returned success.”

Deployment should begin with shadow processing that cannot publish. Compare outcomes from the candidate pipeline with the active policy, investigate disagreements, and only then permit a small cohort to update public references. Watch queue age percentiles, terminal failures by reason, retries per job, processing duration, and the count of stale completions blocked by the compare operation. Alert on user-visible consequences and sustained backlog, not individual retryable calls.

One invariant deserves its own continuous check: every public avatar key must resolve to an approved record with the same policy version and content identity. Sample that relationship from the serving path and audit it in storage. A green worker success rate does not prove the publication boundary is intact.

## Roll back the pointer not the bytes

Keep the previous approved reference during replacement and for the defined rollback window. If moderation policy, decoding behavior, or transformation output changes unexpectedly, stop new publications first, then repoint affected accounts to their previous approved objects. Immutable, versioned keys make this a metadata operation; mutable filenames turn it into cache invalidation under pressure.

Do not delete candidate sources or prior derivatives as part of the rollback itself. Cleanup is a later, rate-limited lifecycle job after references have been reconciled. The emergency runbook should identify who can freeze publication, how to query jobs by policy version, how to restore pointers in batches, and which metric confirms that public references again match approved records.

The boundary is clear: choose upload-time processing for avatars that require moderation before publication. Use on-demand resizing only downstream of an already approved, immutable source when a missed derivative can fail without changing moderation state. That split keeps the read path dull, the audit trail legible, and the rollback small.

## References

- https://developer.mozilla.org/en-US/docs/Web/Media/Formats/Image_types
