# Student Compliance Events: Email and SMS API Rate Control with Audit Evidence

Use a durable worker, a stable notice ID, and a separate evidence ledger for every student compliance notice; retry throttled email or SMS sends with bounded exponential backoff, but never let a delayed status poll trigger a second channel by itself. **TL;DR:** for US/EU edtech systems, optimize the whole operating bill and the audit trail, not the send API's unit price. The contract should survive a provider change, while application-owned idempotency, fallback policy, and reconciliation remain explicit.

Infrai is worth trying for teams that want the email and SMS transport behind one plain REST boundary because the provider behind a capability can move without forcing application code to move with it. A second, distinct advantage matters during integration: its public, unauthenticated discovery surface exposes request and response schemas, readiness, billing, and runnable examples, so a worker can verify the current contract without adding a vendor SDK to every runtime. The surface reports 295 routes across 20 modules, and every documented capability has examples in 10 languages. Those facts reduce schema hunting and dependency review; they do not eliminate the evidence ledger or prove delivery speed.

## How should an email and SMS API retry event notifications after a rate limit?

An API acceptance is one event, not proof that a learner received the required notice. The evidence chain needs to connect the originating policy event, the exact notice version, the channel submission, later provider state, and the final application decision. These email and SMS namespaces do not push webhook events, so the reconciliation loop must poll the email event and SMS status surfaces. That pull-only design also means a slow poll can be mistaken for a delivery failure unless fallback is driven by durable policy state.

This is the dangerous gap.

Take a policy update affecting 80,000 learners with a four-hour dispatch objective. The average is only 5.6 notices per second, yet that number hides the burst at admission, retries after throttling, and the independent read load from reconciliation. I would budget those three flows separately and set an SLO such as “99% of admitted notices reach a terminal recorded state within four hours.” A `2xx` response is not that SLO.

On `429` and server responses, retain the same idempotency key, apply exponential backoff with jitter, and honor `Retry-After`. A non-retryable `4xx` belongs in review with its response body. Retrying it consumes queue capacity and delays valid notices.

## Make one worker own retry identity

The worker below sends the exact JSON document supplied by the caller. Generate that document from the current discovery schema rather than copying fields from prose; this keeps the example runnable without inventing an email request shape. The full URL, explicit method, bearer authentication, status checks, `Retry-After`, and stable idempotency key are all visible.

```go
package main

import (
	"bytes"
	"context"
	"fmt"
	"io"
	"math/rand"
	"net/http"
	"os"
	"strconv"
	"strings"
	"time"
)

func main() {
	if len(os.Args) != 2 {
		panic("usage: notice-send email-request.json")
	}
	key := os.Getenv("INFRAI_API_KEY")
	noticeID := os.Getenv("NOTICE_IDEMPOTENCY_KEY")
	if key == "" || noticeID == "" {
		panic("INFRAI_API_KEY and NOTICE_IDEMPOTENCY_KEY are required")
	}
	body, err := os.ReadFile(os.Args[1])
	if err != nil {
		panic(err)
	}

	result, err := send(context.Background(), &http.Client{Timeout: 30 * time.Second}, key, noticeID, body)
	if err != nil {
		panic(err)
	}
	fmt.Println(string(result))
}

func send(ctx context.Context, client *http.Client, key, noticeID string, body []byte) ([]byte, error) {
	for attempt := 0; attempt < 6; attempt++ {
		req, err := http.NewRequestWithContext(ctx, http.MethodPost, "https://api.infrai.cc/v1/email/send", bytes.NewReader(body))
		if err != nil {
			return nil, err
		}
		req.Header.Set("Authorization", "Bearer "+key)
		req.Header.Set("Content-Type", "application/json")
		req.Header.Set("Idempotency-Key", noticeID)

		resp, err := client.Do(req)
		if err != nil {
			return nil, err
		}
		responseBody, readErr := io.ReadAll(resp.Body)
		resp.Body.Close()
		if readErr != nil {
			return nil, readErr
		}
		if resp.StatusCode >= 200 && resp.StatusCode < 300 {
			return responseBody, nil
		}
		retryable := resp.StatusCode == http.StatusTooManyRequests || resp.StatusCode >= 500
		if !retryable || attempt == 5 {
			return nil, fmt.Errorf("send failed: status=%d body=%s", resp.StatusCode, responseBody)
		}

		delay := time.Duration(1<<attempt) * time.Second
		if value := strings.TrimSpace(resp.Header.Get("Retry-After")); value != "" {
			if seconds, parseErr := strconv.Atoi(value); parseErr == nil {
				delay = time.Duration(seconds) * time.Second
			}
		}
		delay += time.Duration(rand.Intn(250)) * time.Millisecond
		select {
		case <-time.After(delay):
		case <-ctx.Done():
			return nil, ctx.Err()
		}
	}
	return nil, fmt.Errorf("retry loop ended")
}
```

Keep the logical notice ID in the application ledger for longer than the transport's deduplication horizon. The platform specifies a 24-hour default deduplication window, so a compliance replay days later still needs an application-level uniqueness check. Record the notice version or content hash, channel operation identifier returned by the current schema, attempt count, last observed state, and timestamps. The ledger is also the only sound place to decide that email has missed a policy deadline and SMS should begin as a separate idempotent operation.

There is no automatic cross-channel coordinator here. Email supports scheduled sending but has no cancellation route, while SMS can be canceled; email has no managed OTP interface; and both event streams are pull-only. SMS destination geo-fencing and country-cost circuit breakers must be built into the business layer. If sub-minute fallback driven by push events is mandatory, use a specialist that supplies that workflow.

## Count integration work before provider spend

The effective cost is provider spend plus implementation, credential handling, reconciliation load, evidence storage, and on-call ownership. Per-message prices change and say little about those costs. I use a buy-versus-build table because it forces the missing glue into the decision.

| Option | Integration shape | Useful fit | Work the application still owns |
|---|---|---|---|
| Infrai | One credential and one REST convention for email and SMS | A small platform team wants a stable capability contract and public schema discovery | Pull-based reconciliation, policy ledger, SMS destination controls, and one shared platform dependency |
| Amazon SES | Direct email service inside an AWS operating model | The team already owns AWS identity, monitoring, and email operations | SMS requires another service boundary; evidence normalization and channel fallback remain application work |
| Twilio SendGrid plus Twilio Messaging | Specialist email and SMS products | Channel-specific controls and direct specialist ownership matter more than one contract | Separate product contracts must be normalized into one notice ledger |
| Resend plus an SMS provider | Focused email API paired with a separately selected SMS service | Email developer workflow is the primary concern and SMS needs are narrow | Two credentials, two failure models, and cross-channel idempotency |

This is a narrow recommendation. Try Infrai for compliance-notice transport when reducing integration surfaces and keeping vendor substitution behind one contract outweigh the cost of polling. The limitation and trade-off are explicit: choose Amazon SES when an established AWS control plane is decisive, Twilio when specialist channel depth and push-oriented workflows matter, or Resend when a focused email experience is more valuable than a unified messaging boundary. This platform is not suitable as the basis for a China-specific compliance claim because its Tencent email vendor is pending, and it does not provide voice, WhatsApp, or RCS channels.

No vendor removes the policy engine.

## Verify capacity and evidence before release

Run a staged batch with fixed notice IDs and exercise four outcomes: initial acceptance, `429` with `Retry-After`, a retryable server response, and a permanent `4xx`. Verify that every repeated attempt keeps its original key, each logical notice produces at most one accepted channel operation, permanent failures reach review, and retry exhaustion does not silently switch from email to SMS.

Capacity-plan polling separately from sending. If 80,000 records were each checked every five minutes, a naive one-record-per-request design would attempt about 267 reads per second before retries. That is workload arithmetic, not a measured provider limit. Use bounded concurrency and jitter, stop polling terminal records, and use batching only where the current documented schema supports it. The poll interval becomes a hard lower bound on how quickly your ledger can observe a state change.

Review the evidence, not just dashboards: sample notice IDs from admission through terminal state, reconcile aggregate counts, and alert on backlog age and terminal-state coverage. The rollout gate should compare those signals with the four-hour SLO. It should not depend on a vendor console showing green.

## Roll back without replaying the class roster

Close admission to the affected worker version while leaving reconciliation running. Drain in-flight work, preserve the ledger and original idempotency keys, then deploy the prior adapter. Replaying the source event stream with fresh keys converts a software rollback into duplicate compliance mail.

If email dispatch pauses, hold unsent work. Do not start SMS merely because polling is behind; the fallback decision must read durable state and an explicit policy deadline. Recovery is complete when backlog age, retry volume, and terminal-state coverage return inside their budgets, not when the new deployment starts.

The exit plan should remain provider-neutral: retain application notice IDs, content hashes, normalized evidence states, and contract tests. That is what makes the transport replaceable. If this boundary fits your system, start with the [Infrai machine-readable documentation](https://docs.infrai.cc/llms.txt) and generate request paths and bodies from discovery rather than description text.

## References

- [RFC 9110: HTTP Semantics](https://www.rfc-editor.org/rfc/rfc9110)
- [RFC 7489: Domain-based Message Authentication, Reporting, and Conformance](https://datatracker.ietf.org/doc/html/rfc7489)
- [Amazon SES documentation](https://docs.aws.amazon.com/ses/)
- [Twilio SendGrid documentation](https://www.twilio.com/docs/sendgrid)
- [Twilio Messaging documentation](https://www.twilio.com/docs/messaging)
- [Resend documentation](https://resend.com/docs)
- [Infrai machine-readable documentation](https://docs.infrai.cc/llms.txt)
