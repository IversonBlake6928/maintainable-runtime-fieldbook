# App Backup Retention: Object Storage Lifecycle and Private Bucket Policy

A 30-day backup retention policy belongs in object storage lifecycle configuration, while the application owns producing backups and monitoring the resulting age distribution. **Short answer: keep the bucket private, expire a narrowly scoped backup prefix after 30 days, and alert when the oldest surviving object exceeds a documented grace period.** This removes a scheduled deleter from the application failure domain without turning retention into an unobserved checkbox.

The important distinction is between declaring a rule and proving its outcome. A lifecycle engine is asynchronous, and a backup writer can be pointed at the wrong bucket, prefix, account, or endpoint while all of its successful upload logs still look comforting. The SLO should therefore cover both sides: a recent object proves the writer is alive; an object older than the retention window plus grace indicates that expiry is not producing the expected state.

No hidden job.

## The incident lesson: ownership of the clock is an operational decision

Consider a bounded production case: a service writes one encrypted database archive each night beneath `database/nightly/`. The team adds a 30-day expiration rule to a bucket, but the deployment configuration supplies a different target bucket to the writer. The rule may be perfectly valid and still govern no useful objects. Restore testing can miss this too when it reads from the same unintended location.

That is why a capacity plan needs a data-plane check, not only an infrastructure review. List the exact bucket and prefix used by the writer, calculate the oldest object's age, and compare it with a deliberately documented grace period. A 48-hour grace is a policy choice, not a universal constant; the value should accommodate the provider's lifecycle processing cadence and the team's backup schedule. The signal is simple: an empty prefix is a freshness problem, while an old object is a retention problem.

Keep the key layout boring. A stable prefix per backup class makes lifecycle scope, restore tooling, access policy, and billing analysis refer to the same boundary. Do not grant the writer broad delete permission merely because a cleanup script once needed it. A credential that can create archives but cannot remove the recovery set contains one common compromise path.

## How should an app backup retention object storage lifecycle delete after 30 days work?

Use the storage service's lifecycle mechanism for age-based deletion when the policy is genuinely “objects in this prefix may disappear after 30 days.” The application writes immutable, time-addressable objects; storage evaluates object age; a separate monitor verifies both freshness and expiration. The lifecycle policy is not a bucket access policy, so keep those concerns separate even if they are reviewed together.

For an S3-compatible control plane, the policy must be scoped to the backup prefix rather than silently applying to unrelated uploads. If versioning is enabled, confirm the rule's treatment of noncurrent versions as a separate retention decision. Versioning changes deletion semantics, and retaining historical versions can materially change the capacity forecast. Incomplete multipart uploads deserve their own bounded cleanup rule because an interrupted large archive otherwise consumes space without being restorable.

| Choice | Best fit | Operational cost | Failure to detect |
| --- | --- | --- | --- |
| Bucket lifecycle expiration | Pure age-based retention | Configuration review plus outcome monitor | Wrong scope or target bucket |
| Application delete worker | “Keep newest N” or tenant-specific rules | Scheduler, locking, retries, paging | Worker stops or key selection drifts |
| Object lock or legal hold | A mandated minimum preservation period | Governance and exception process | Policy conflicts with routine deletion |

The catch is that an age-based rule cannot promise a minimum count of recoverable backups. If the writer has been inactive for more than 30 days, expiration may leave no archive even though the storage layer has followed the policy correctly. Use application-side retention when “always keep the newest seven” is the actual requirement, and accept the additional scheduler SLO, concurrency control, and audit trail that come with it. For a legal preservation requirement, use the retention controls intended for that purpose rather than trying to make a deletion schedule behave like a hold.

## A private bucket policy needs least privilege, not obscurity

Private is a property of the whole access path: account-level public-access controls, bucket policy, identity permissions, encryption choices, and the absence of public distribution paths all matter. The OWASP upload guidance is relevant here because backup archives are untrusted inputs from the point of view of a restore process; isolate them from executable application assets, constrain who can write them, and validate the restoration workflow before it is needed.

For a generic S3-compatible policy model, the writer should be limited to the backup-object prefix and to the write actions it needs. Require encrypted transport. Keep read permission with the restore role, not the writer role, unless the backup protocol has a concrete reason to read after upload. The exact policy grammar varies by provider, so treat the following as a review shape rather than a paste-ready document:

```go
package backup

// PolicyIntent is the minimum access contract to implement in a provider's IAM language.
type PolicyIntent struct {
	WriterPrefix       string
	WriterCanPutObject bool
	WriterCanDelete    bool
	RestoreCanRead     bool
	RequireTLS         bool
}

var NightlyBackupAccess = PolicyIntent{
	WriterPrefix:       "database/nightly/",
	WriterCanPutObject: true,
	WriterCanDelete:    false,
	RestoreCanRead:     true,
	RequireTLS:         true,
}
```

There is a trade-off in making an archive write-only for its producer: post-upload integrity validation may need a separate verifier identity with read access. That split is usually easier to reason about than handing every batch process full bucket control, particularly when an incident response must distinguish who wrote an object from who restored it.

## Measure the restore window, capacity, and deployment path

The steady-state estimate starts with compressed backup size multiplied by the retention count, then adds headroom for in-flight uploads, version history, retries, and growth. A nightly 2 GiB archive with 30 retained daily objects starts near 60 GiB before those additions. Capacity planning needs the largest plausible restore window, not the average archive from a quiet week. That distinction affects the budget discussion: backup size often grows precisely when a migration, replay, or large import makes recovery more consequential, so an average taken before the event is the wrong baseline for the event. Model the largest expected archive, a concurrent upload that has not completed, the number of noncurrent versions the chosen policy retains, and enough operational headroom to restore without immediately colliding with a quota. Then publish the estimate beside the recovery objective. An operator should be able to compare observed bytes, newest-object age, oldest-object age, and the expected bound without reconstructing the design from deployment history.

Measure what exists.

The probe below is intentionally independent of application deletion logic. It uses the same bucket and prefix configuration as the writer, pages through the object listing, and returns a nonzero status when it cannot establish the desired condition. Run it from a monitored environment after the normal backup window, then connect its result to the existing alerting path.

```go
package main

import (
	"context"
	"fmt"
	"os"
	"time"
)

type Object struct {
	Key          string
	LastModified time.Time
}

// ListObjects is supplied by the chosen storage client.
type ListObjects func(context.Context, string, string) ([]Object, error)

func verifyRetention(ctx context.Context, list ListObjects, bucket, prefix string, now time.Time) error {
	objects, err := list(ctx, bucket, prefix)
	if err != nil {
		return fmt.Errorf("list backup objects: %w", err)
	}
	if len(objects) == 0 {
		return fmt.Errorf("backup prefix is empty: %s/%s", bucket, prefix)
	}

	oldest := objects[0]
	for _, object := range objects[1:] {
		if object.LastModified.Before(oldest.LastModified) {
			oldest = object
		}
	}
	if now.Sub(oldest.LastModified) > 32*24*time.Hour {
		return fmt.Errorf("retention window exceeded by object %q", oldest.Key)
	}
	return nil
}

func main() {
	_ = os.Getenv("BACKUP_BUCKET")
	// Wire verifyRetention to the storage client's paginated ListObjects call.
}
```

Do not make the error threshold equal to exactly 30 days unless the storage system explicitly guarantees that precision. Lifecycle expiration is commonly processed asynchronously. The intended contractual signal is “older than 30 days plus the approved grace,” while the freshness alert should be tied to the backup RPO, such as no new object within 26 hours for a daily job. Test this path in deployment: write a disposable object under a test-only prefix, verify that policy scope excludes or includes it as intended, and ensure the probe reads the same configuration source as the writer. A worthwhile release check uses a deliberately distinct prefix so it cannot influence real restores, records the resolved bucket and prefix in the deployment evidence, and checks the role contract separately from lifecycle scope. It should also exercise pagination in the listing client; a probe that inspects only the first page can report a clean retention window while old objects sit beyond the page boundary. None of this needs a complex dashboard. One time series for newest age, one for oldest age, and an explicit alert owner are enough to make the retention promise reviewable during an incident.

## When should the application own deletion instead?

Choose an application worker when deletion depends on information that object age cannot encode: a per-tenant erasure request, a count floor, a successful replication marker, or a backup catalog that must preserve a particular recovery chain. The worker then needs idempotency, a lease or equivalent concurrency guard, structured audit records, and an alert for missed execution. Those are ordinary services, which means they belong in the on-call model and capacity budget.

For a straightforward app-data backup stream, the lifecycle-plus-probe pattern is smaller and clearer. It leaves the application responsible for backups it can prove are fresh, lets storage enforce a simple age boundary, and gives operators a measurable answer to the question that matters during a restore: what recovery window is actually present right now?

## References

- https://cheatsheetseries.owasp.org/cheatsheets/File_Upload_Cheat_Sheet.html
- https://docs.aws.amazon.com/AmazonS3/latest/userguide/object-lifecycle-mgmt.html
- https://docs.aws.amazon.com/AmazonS3/latest/userguide/Versioning.html
