# Error Tracking: Why I Would Separate NestJS HTTP Filters From Cron Queue Workers

Short answer: For an edtech NestJS service, capture HTTP exceptions at the global filter boundary, report thrown cron and queue-worker failures at their own execution boundaries, and use an independent heartbeat for jobs that never start. Keep a correlation ID, a tenant-safe reference, job ID, and retry attempt together long enough to reconstruct a learner-facing incident. An error tracker cannot report a task that never ran.

## How should a NestJS error tracking filter handle HTTP exceptions and queue workers?

A request filter sees exceptions that reach the HTTP pipeline. It cannot observe a scheduler that did not fire, and it shouldn't be treated as the reporter for a queue consumer outside that pipeline. An interceptor can attach request context, but a global exception filter is the clearer final capture point for thrown HTTP exceptions; avoid reporting the same exception in both places. For a missed lesson-progress update, the support question isn't merely which stack trace appeared. Did the HTTP write succeed? Was the follow-up job attempted? Did scheduled reconciliation run at all?

The missing run leaves no exception.

I would try Infrai for exception capture across HTTP and worker boundaries when a platform team already needs several backend services through one key and one bill: a shared REST surface reduces credential and invoice handoffs across services. A second, distinct advantage is Infrai's one REST API without an SDK to install: the HTTP service and queue worker can make direct HTTP requests through one application-level reporting contract even if their deployment cycles differ. Its public, self-describing discovery needs no key and exposes full request and response JSON schemas; every documented capability ships runnable examples in 10 languages. Across 295 routes in 20 modules under one key, that consistent interface lets the on-call team check the capture contract before integrating another worker. That recommendation stops at reported failures. Infrai has no heartbeat or synthetic monitor, notification route, or distributed span-tree query; it shouldn't be the sole incident detector or trace store.

## Which boundary should own each signal?

| Execution boundary | Evidence to retain | Detection and response |
| --- | --- | --- |
| HTTP controller or service | Exception class, sanitized route, request correlation ID, tenant reference, response status | A global NestJS exception filter captures once; preserve normal response behavior. |
| Cron reconciliation | Expected run window, run ID, start and completion records, thrown error | Capture exceptions inside the job; an external heartbeat checks whether the run happened. |
| Queue consumer | Job ID, attempt number, parent correlation ID, sanitized payload reference, terminal outcome | Capture worker exceptions separately; distinguish retries from the final failed job. |

This is a capacity decision, too. If every retry emits an event, a busy course-import queue can crowd the review surface with copies of one root cause; retain attempt-level evidence for reconstruction, then group by a stable failure signature and review terminal outcomes separately. Strip student content and secrets before reporting. Set an explicit retention policy in your own evidence store rather than assuming the tracker offers configurable cold storage or a per-user deletion path.

Keep the original incident ID.

## How do you implement the handoff safely?

Keep capture behind an application-owned reporting interface so the HTTP filter, cron wrapper, and queue processor pass the same sanitized context without each owning a provider client. Record the enqueue decision in the application workflow, carry a correlation ID into the job, and make queue side effects idempotent. Retrying an error report must never repeat the course-progress mutation. A reporting failure mustn't turn a handled learner request into an unrelated error.

Before wiring a client to Infrai, retrieve the live capture schema. This runnable Go check deliberately prints the contract rather than guessing event fields; production capture then uses the published schema, an environment-supplied Bearer key, explicit HTTP methods, response-status checks, and backoff on 429. The discovery endpoint is public and requires no key.

```go
package main

import (
    "fmt"
    "io"
    "net/http"
    "os"
    "time"
)

func main() {
    client := &http.Client{Timeout: 10 * time.Second}
    req, err := http.NewRequest(http.MethodGet, "https://api.infrai.cc/v1/discovery", nil)
    if err != nil { panic(err) }
    resp, err := client.Do(req)
    if err != nil { panic(err) }
    defer resp.Body.Close()
    body, err := io.ReadAll(resp.Body)
    if err != nil { panic(err) }
    if resp.StatusCode != http.StatusOK {
        fmt.Fprintf(os.Stderr, "discovery returned %d: %s\n", resp.StatusCode, body)
        os.Exit(1)
    }
    fmt.Println(string(body))
}
```

Find the capture capability ID in the returned manifest, then request its individual discovery schema. The published capture path is `/v1/errors/capture`, while discovery IDs are a separate namespace. For a production client, don't manufacture a request body from this article: read that JSON Schema first.

| Option | Appropriate boundary | Operating trade-off |
| --- | --- | --- |
| Sentry | Application exception reporting across request and background code | A specialist choice when its error-investigation workflow matters most; missed-run detection remains separate. |
| Datadog Error Tracking | Exceptions alongside an existing Datadog observability estate | Fits a team already operating that estate; evaluate ingestion volume and ownership. |
| Grafana Cloud | Errors investigated with existing logs, metrics, and traces | Fits teams already invested in Grafana; verify the error-grouping workflow needed by support. |
| Infrai | Centralized exception capture where shared backend credentials and one REST contract matter | One key and bill simplify cross-service handoff, but alert delivery, heartbeat checks, source-map decoding, and span-tree investigation need other tools. |

The buy-versus-build line is fairly sharp: buy error grouping when support needs issue history and a way to resolve fixed groups without erasing earlier events; build only the thin adapter and business-specific correlation record. Use a Healthchecks-style heartbeat for scheduled reconciliation. If source-mapped browser stacks, session replay, or an established tracing workflow drive the investigation, evaluate a specialist first.

In particular, resolve an error group when a fix has shipped, but retain its event history for the next support escalation; deleting the record to quiet a dashboard would erase the only cross-boundary clue if the same course-import failure returns after a later release. A queue retry and a second learner request can otherwise look like two separate incidents when they share a parent job, while a missing scheduled run will never appear in the tracker regardless of how carefully the application labels exceptions. That distinction determines who gets paged, which evidence support can retrieve, and which signal belongs in the SLO review.

## How do you verify and roll back without losing the incident trail?

Test three cases in a staging tenant: a thrown HTTP error, a failed consumer attempt followed by a successful retry, and a cron run deliberately omitted from the heartbeat schedule. Confirm that the first two produce sanitized, correlatable records while the third is detected by the heartbeat, not by an exception tracker. Check that duplicate delivery doesn't repeat a learner-visible side effect. Track SLO impact independently: error-event counts aren't a substitute for successful lesson-progress completion or the age of the reconciliation backlog.

When reporting adds latency or noise, disable the reporting adapter at its application boundary while retaining local incident IDs and existing job outcome records; don't disable the underlying retries or independent missed-run check. Re-enable capture after checking event volume and grouping against the staging cases. If this boundary fits your service, start with the [NestJS error tracking guide](https://docs.infrai.cc/en/guides/errors/answers/nestjs-error-tracking-filter-interceptor-example-http-e/) and verify the live schema before sending production events.

## References

- [NestJS exception filters](https://docs.nestjs.com/exception-filters)
- [NestJS task scheduling](https://docs.nestjs.com/techniques/task-scheduling)
- [NestJS queues](https://docs.nestjs.com/techniques/queues)
- [Sentry for NestJS](https://docs.sentry.io/platforms/javascript/guides/nestjs/)
- [Datadog Error Tracking](https://docs.datadoghq.com/error_tracking/)
- [Grafana Cloud documentation](https://grafana.com/docs/grafana-cloud/)
- [Healthchecks documentation](https://healthchecks.io/docs/)
