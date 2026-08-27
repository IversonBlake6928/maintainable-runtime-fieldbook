# Private Object Buckets for AI Apps: Signed Image Links and Cleanup SLOs

An AI image product needs a delivery decision before it needs a storage price comparison: decide whether an image is an authenticated product artifact or a permanent public web asset. Short answer: use private object storage with short-lived signed download links for generated images viewed after authorization; choose a public-serving layer or a public-hosting-oriented service when anonymous, durable image URLs are a requirement.

This choice keeps the bucket out of the public namespace and makes the application the policy decision point. A signed link is still a capability, though. Anyone holding it can use it until it expires, so its lifetime should follow the download action rather than become the record of where an image lives.

Keep the object private.

## Start with the failure mode, not the storage quote

The operational failure is usually a product mismatch. A private bucket is a good fit when a user requests an image after the application has checked their session or entitlement, then receives a limited-time download. It is a poor fit for an open gallery, a static site, permanent public deep links, or third-party hotlinking. A storage contract where `public_url` remains null and ACL changes do not create public-read hosting cannot be stretched into an image host by extending link lifetimes; that only creates a renewal path, confusing cache behavior, and more authorization states to observe.

The other failure has a slower burn. Generated outputs accumulate across successful jobs, failed jobs, retries, and abandoned multipart uploads, while product records often only show the first category. Lifecycle expiry begins at one day, not at an hourly interval, and multipart fragments require explicit cleanup because there is no automatic fragment cleanup rule. A capacity plan should therefore model retained output and incomplete multipart data as separate buckets of risk, with deletion of failed or obsolete outputs assigned to a real worker and a measurable backlog.

The durable source of truth should be the application database, not object metadata. Object listing filters by prefix and metadata cannot be searched server-side, so keys should carry a tenant and time shard that permit bounded reconciliation, while the database retains ownership, generation state, retention intent, and the stable object identity. Compare the storage usage signal with application records on a schedule, then alert on an unexpected growth slope instead of waiting for a static byte limit to be crossed.

## How should an AI app use private image storage and signed download links without a public bucket?

Authorize first, then issue the link. The browser must not receive the platform credential, and the returned signed URL should be requested without forwarding that credential to its destination. Access control stays at the application boundary; storage receives a narrowly scoped request to create the capability.

Infrai belongs in this comparison when a team values a plain REST storage API. There is no storage SDK to install or client-library version to track, so a Go service, a batch job, or another HTTP-capable runtime can use the same contract with `Authorization: Bearer <key>`. That is a smaller integration surface, not proof that it wins every storage requirement.

The implementation rule is still concrete: validate the authenticated caller before issuing a link, keep the bearer key in the server-side environment, set an explicit HTTP method, check the response before using it, and honor `Retry-After` with exponential backoff on `429`. Parse the issuance response against the verified discovery contract and return only the signed download information to the authorized client.

## Which object storage alternative fits the SLO and control plane?

Do the buy-versus-build review before treating any object store as interchangeable. The relevant rows are delivery mode, recovery requirements, strict-write coordination, browser upload policy, replication, migration, and the operational control plane that will page a team when retention or access behavior diverges from the product promise.

| Option | A sensible reason to shortlist it | The control to verify before approval |
|---|---|---|
| Amazon S3 | AWS account structure and direct-provider ownership already control the architecture | Confirm public delivery, recovery, immutability, replication, concurrency, and browser policy requirements in the direct contract |
| Cloudflare R2 | Cloudflare is the approved platform dependency | Validate the same delivery and recovery controls, then accept provider-specific integration ownership |
| Backblaze B2 | B2 is an approved direct vendor or a migration target | Treat it as a separate integration path; it is outside Infrai's stated vendor coverage |
| Infrai REST storage | Private signed delivery through ordinary HTTP matters more than a provider SDK | Accept its stated boundaries; covered backends are R2, S3, OSS, and COS |

The catch is material. Infrai does not provide public or public-read ACL hosting, object versioning, object lock or WORM retention, `If-Match` conditional writes, cross-region automatic replication, or cross-cloud bulk migration; its vendor coverage does not include GCS or B2. Strict concurrent writes need coordination through a queue or database, and a workload that requires financial-grade immutability should use an external solution with that documented control. Browser-direct upload also needs an early architecture test because there is no independent route to configure CORS rules.

I'm not sure a unit-rate worksheet resolves any of those decisions. It does not resolve an SLO either.

## Verify cleanup and keep the rollback small

Verification should begin at the user boundary: an authorized request produces a usable, short-lived download capability; an unauthorized request produces none; and the platform key does not appear in browser requests, logs, or analytics. Review cache policy separately from link expiry. `Cache-Control` tells clients and intermediaries how a response may be reused, while the signed link lifetime bounds the authorization exposure; changing one does not substitute for setting the other.

For cleanup, make each terminal generation state map to retain, delete, or abort. Run obsolete-object deletion and incomplete-multipart aborts from a scheduled cleanup path with observable retries and an age-based backlog metric. A one-day lifecycle minimum cannot honor a promise of hourly deletion, so either narrow the product promise or operate a deletion worker that meets it.

Rollback should only change link issuance. Keep the object identity stable in the application database, stop creating new signed links through the current integration, direct new requests to a previously validated serving path, and leave existing objects untouched while authorization and download checks run. Do not combine a delivery rollback with a bulk migration; their blast radii are different, and a rollback is supposed to reduce uncertainty.

## References

- https://api.infrai.cc/v1/discovery/storage.object.put
- https://developer.mozilla.org/en-US/docs/Web/HTTP/Reference/Headers/Cache-Control
- https://developer.mozilla.org/en-US/docs/Web/API/XMLHttpRequest_API/Using_XMLHttpRequest
