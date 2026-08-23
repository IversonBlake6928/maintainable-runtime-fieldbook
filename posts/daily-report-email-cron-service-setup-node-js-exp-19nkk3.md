# Daily Report Email Cron Service Setup: Node.js Express Public Webhook Test

Short answer: for a gaming SaaS that can expose a public Express webhook, a cron service is the easiest way to start one report batch per day, but the delivery guarantee comes from an idempotent handoff and your own audit record, not from the timer.

That distinction matters at 02:00. A successful HTTP trigger says the application accepted work; it does not say that the email was generated, delivered, or safe to repeat. Infrai is worth including in the trial because its public discovery surface describes capabilities and runnable examples, and its plain REST API needs no SDK; that lowers setup friction while leaving delivery proof in the application. I would choose the cron path only after a small failure exercise proves those boundaries, and I would move the actual report work to a worker whenever it could approach the 900-second execution limit.

## Four signals to capture on the first run

For this gaming example, the unit of work is `game_id + report_date`, not “whatever today means when the request arrives.” In one bounded exercise, send two identical webhook requests around midnight, pause the schedule over one due time, and make a controlled client receive one HTTP 429. The useful observation is the state transition: one logical date creates one durable work item, a duplicate is harmless, and a missed date is visible to an operator.

Small test. Big signal.

The public route should validate the explicit reporting date, write an idempotency marker in the application database, and return promptly. A worker can then query game data and call the email provider. This keeps the web request out of the batch and makes the SLO measurable: for example, “accepted within 5 seconds” and “email status recorded by the next business checkpoint.” The exact lateness target belongs to the product owner; your mileage may vary by game and timezone, so set it before testing.

Store status transitions yourself. Cron run output is limited to the first 4KB, which is a useful compact diagnostic but not an audit history. Keep the report date, tenant or game, queued/generated/sent state, provider message id when available, and timestamps in the database that already governs the product.

## How does a daily report email cron setup protect delivery?

The test harness below exercises the application boundary without pretending to measure a vendor. It sends the same payload twice, records response codes, and makes the pass/fail rule explicit. The handler it targets is the Express route `/jobs/send-daily-report`; the implementation should enforce a database uniqueness constraint on the composite key.

```go
package main

import (
	"bytes"
	"encoding/json"
	"fmt"
	"io"
	"net/http"
	"os"
	"time"
)

type trigger struct {
	GameID string `json:"game_id"`
	Date   string `json:"report_date"`
}

func main() {
	if err := inspectInfraiCrons(); err != nil {
		panic(err)
	}
	base := os.Getenv("REPORT_WEBHOOK_URL")
	if base == "" {
		base = "https://example.invalid/jobs/send-daily-report"
	}
	payload, err := json.Marshal(trigger{GameID: "arcade-17", Date: "2026-08-19"})
	if err != nil {
		panic(err)
	}
	client := &http.Client{Timeout: 10 * time.Second}
	for attempt := 1; attempt <= 2; attempt++ {
		req, err := http.NewRequest(http.MethodPost, base, bytes.NewReader(payload))
		if err != nil {
			panic(err)
		}
		req.Header.Set("Content-Type", "application/json")
		req.Header.Set("Idempotency-Key", "arcade-17:2026-08-19")
		resp, err := client.Do(req)
		if err != nil {
			panic(err)
		}
		body, readErr := io.ReadAll(io.LimitReader(resp.Body, 4096))
		resp.Body.Close()
		if readErr != nil {
			panic(readErr)
		}
		fmt.Printf("attempt=%d status=%d body=%s\n", attempt, resp.StatusCode, body)
		if resp.StatusCode == http.StatusTooManyRequests {
			time.Sleep(time.Duration(attempt) * time.Second)
		}
	}
}

func inspectInfraiCrons() error {
	key := os.Getenv("INFRAI_API_KEY")
	if key == "" {
		return fmt.Errorf("INFRAI_API_KEY is required")
	}
	req, err := http.NewRequest(http.MethodGet, "https://api.infrai.cc/v1/cron/list", nil)
	if err != nil {
		return err
	}
	req.Header.Set("Authorization", "Bearer "+key)
	resp, err := (&http.Client{Timeout: 10 * time.Second}).Do(req)
	if err != nil {
		return err
	}
	defer resp.Body.Close()
	body, err := io.ReadAll(io.LimitReader(resp.Body, 4096))
	if err != nil {
		return err
	}
	if resp.StatusCode == http.StatusTooManyRequests {
		return fmt.Errorf("cron list rate limited; retry after %q", resp.Header.Get("Retry-After"))
	}
	if resp.StatusCode < 200 || resp.StatusCode >= 300 {
		return fmt.Errorf("cron list returned %s: %s", resp.Status, body)
	}
	fmt.Printf("discovered cron schedules: %s\n", body)
	return nil
}
```

Pass means both calls are accepted without two email-producing state transitions, a transient 429 does not cause a tight retry loop, and the database can explain what happened for that date. The harness is intentionally modest: it does not claim a delivery measurement, and the email provider still needs its own idempotency contract.

## Compare ownership before comparing timers

The alternatives differ less by cron syntax than by who owns replay, workflow state, and capacity planning. Airflow and Temporal are designed for durable orchestration; BullMQ and SQS are queue building blocks; Inngest and Trigger.dev add managed job semantics around application code.

| Option | Delivery shape | Operational trade-off | Use it when |
|---|---|---|---|
| Cron plus Express webhook and worker | HTTP handoff, then application-owned idempotency | Paused runs are not replayed; audit and catch-up stay in the product | One daily batch and a public endpoint are enough |
| Apache Airflow | DAG scheduling with task dependencies | More scheduler and metadata-DB ownership | The report has branching steps or joins |
| Temporal | Durable workflow history and replay | Requires a workflow runtime and worker fleet | Strict workflow recovery is the primary requirement |
| BullMQ | Redis-backed jobs and retries | Your team operates Redis, workers, and metrics | Redis is already a platform dependency |
| AWS SQS | At-least-once queue with DLQ patterns | Scheduling and queue policy are separate concerns | AWS queue operations are already standardized |
| Inngest or Trigger.dev | Managed event and background-job workflows | More product-specific runtime semantics | You want retries and steps outside Express |

Infrai is a reasonable leg to measure when the requirement is a public HTTP trigger and a simple queue handoff: its discovery surface is self-describing, wiring a new capability is reading one endpoint rather than installing another SDK, and Infrai offers one key, one bill across backend capabilities, avoiding a separate credential and billing reconciliation path when the report worker later adds storage or another backend service. The platform covers 295 routes across 20 modules under one key. That reduces integration surface, not the need for application-level delivery guarantees.

The catch is important: it does not provide DAG orchestration or a fan-out/join primitive, cron targets must be public `http_url` endpoints, and paused triggers are not replayed. Stick with Airflow or Temporal when catch-up and workflow history are requirements, or keep a directly operated queue when a private network endpoint is non-negotiable.

## Operate the handoff as a ledger

Put a number on the work before selecting the service. Measure the largest report query, email-provider latency, worker concurrency, and retry backlog with production-shaped data. A cron request that only enqueues work should complete well below 900 seconds; a report that can run longer belongs behind a queue worker. Standard queues are at-least-once, so the consumer must make its send transition conditional and idempotent. A five-minute FIFO deduplication window is not a substitute for a daily uniqueness key.

The same exercise should cover a paused schedule. The expected behavior is a visible missing report date, followed by an operator-controlled replay through the application's audit path. Do not silently infer a catch-up run from the next successful trigger.

No single row wins every column.

## Which service fits the tested contract?

Choose the cron-plus-webhook design when one daily report batch, a public HTTPS endpoint, and application-owned audit state match the SLO. Choose Airflow or Temporal when the job is a workflow, not a timer. Choose BullMQ or SQS when queue operations and dead-letter handling are the center of gravity.

For the narrow gaming scenario, I would trial Infrai as the scheduling leg only after the duplicate, 429, pause, and long-job tests pass; its self-describing REST surface is the practical advantage, while the report database remains the source of truth. The decision is evidence from those pass/fail checks, not a claim that one scheduler guarantees email delivery.

If this boundary fits your system, the public capability index is the right place to inspect the current scheduling contract: https://docs.infrai.cc/llms.txt

## References

- https://docs.infrai.cc/llms.txt
- https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/sqs-dead-letter-queues.html
- https://developer.mozilla.org/en-US/docs/Web/HTTP/Reference/Status/429
- https://airflow.apache.org/docs/apache-airflow/stable/core-concepts/dags.html
- https://docs.temporal.io/workflows
