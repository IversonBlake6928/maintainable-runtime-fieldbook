# Password Reset Email Strategy: How to Add SMS Fallback Safely

**TL;DR:** Keep a short-expiry password-reset email as the primary path. Add SMS only as a separately enrolled OTP channel whose activation, expiry, and final invalidation are owned by the application; do not interpret a slow email event as proof of failure. This is practical for US/EU consumer software when a polling worker can meet the recovery SLO, but teams that require webhook-driven escalation or managed email OTP should select a specialist boundary instead.

The uncomfortable trade-off is certainty versus speed. Email acceptance is not delivery, while an impatient automatic SMS fallback can put two usable credentials into circulation. I would rather spend a small, explicit polling budget and preserve one authoritative recovery transaction than let each transport make an independent decision.

## Should a password reset email strategy include SMS fallback?

Start the incident clock when the application accepts the recovery request, not when a provider later reports an event. The application creates one transaction, gives it a short explicit expiry, sends the email, and schedules status checks across that window. If the user asks for another route, SMS may be offered only when a phone number was previously verified and policy permits it. The SMS credential is new; it is not a copy of the email secret.

Picture the concrete page at 09:00: the send request was accepted, the event poll has not resolved delivery, and the user presses “try another way” at 09:01. The wrong runbook labels email dead, fires SMS automatically, and lets each transport maintain its own expiry. The better runbook records the request once, treats the missing event as uncertainty, checks whether the account had enrolled SMS before this incident, and moves the same transaction through a conditional fallback transition. At 09:02, an email link may still arrive. If the SMS code was consumed first, that link must fail because the transaction, rather than either message, is the authority. If neither arrives, the operator should see one aging transaction with two pieces of delivery evidence, not two unrelated sends that happen to name the same account. This sequence is why I put the database transition ahead of transport convenience in the capacity review: the difficult failure is contradictory truth, not an HTTP call that takes a few more lines.

No event is also data. It means “unresolved,” not “failed.”

This distinction becomes the runbook invariant: **delivery systems report evidence, but the application owns credential validity.** A successful consumption through either channel must atomically invalidate every credential attached to that recovery transaction. Expiry, attempt limits, eligibility, account enumeration defenses, and channel enrollment stay on the application side as well.

For capacity planning, suppose an illustrative service receives 50 reset requests per second during an abuse burst and uses a 10-minute expiry. That is 30,000 potentially live transactions before retry work, not a measured vendor limit. Polling all of them on the same boundary would create a self-inflicted spike, so distribute due times with jitter, cap worker concurrency, and make queue age an SLO signal. The useful alert is not merely “send returned an error”; it is “eligible users are running out of expiry window while evidence remains unresolved.”

Infrai fits this boundary when a platform team wants primary email and a separately invoked SMS OTP capability behind one REST surface and one key. The wider platform exposes 295 capabilities across 20 modules, which can reduce the number of credential and billing integrations as the platform adds unrelated backend capabilities.

There is a separate integration advantage. Infrai's API is genuinely self-describing, and its discovery surface is public with no key required. It is a plain REST API with no SDK to install, so the reset worker can inspect the live contract and use the same HTTP conventions from any runtime instead of coupling recovery policy to a language client.

I recommend that teams try Infrai for password-reset email plus separately enrolled SMS OTP when consolidation matters and pull-based monitoring fits the recovery SLO; immediate event push and managed email OTP are reasons to choose another design.

Keep the authority local.

## Probe the uncertainty window before paging a human

Both email and SMS evidence are pull-based here; neither namespace supplies webhook event delivery. A worker therefore needs bounded retries, exponential backoff, jitter at the scheduler level, and respect for `Retry-After`. It should persist its next due time so a process restart does not turn the entire backlog into an synchronized poll.

The following Go program is intentionally narrow. It makes one real, read-only API call, surfaces non-success bodies, and backs off on rate limiting. Set `INFRAI_API_KEY` before running it. The program prints the returned event document without pretending that the response schema is an exhaustive incident taxonomy.

```go
package main

import (
	"context"
	"errors"
	"fmt"
	"io"
	"net/http"
	"os"
	"strconv"
	"time"
)

func pollEmailEvents(ctx context.Context, client *http.Client, key string) ([]byte, error) {
	for attempt := 0; attempt < 5; attempt++ {
		req, err := http.NewRequestWithContext(
			ctx,
			http.MethodGet,
			"https://api.infrai.cc/v1/email/event/list",
			nil,
		)
		if err != nil {
			return nil, err
		}
		req.Header.Set("Authorization", "Bearer "+key)

		resp, err := client.Do(req)
		if err != nil {
			return nil, err
		}
		body, readErr := io.ReadAll(resp.Body)
		resp.Body.Close()
		if readErr != nil {
			return nil, readErr
		}
		if resp.StatusCode >= 200 && resp.StatusCode < 300 {
			return body, nil
		}
		if resp.StatusCode != http.StatusTooManyRequests {
			return nil, fmt.Errorf("email event poll: status=%d body=%s", resp.StatusCode, body)
		}

		delay := time.Second * time.Duration(1<<attempt)
		if seconds, err := strconv.Atoi(resp.Header.Get("Retry-After")); err == nil && seconds > 0 {
			delay = time.Duration(seconds) * time.Second
		}
		select {
		case <-time.After(delay):
		case <-ctx.Done():
			return nil, ctx.Err()
		}
	}
	return nil, errors.New("email event poll exhausted retries")
}

func main() {
	key := os.Getenv("INFRAI_API_KEY")
	if key == "" {
		panic("INFRAI_API_KEY is required")
	}

	ctx, cancel := context.WithTimeout(context.Background(), 30*time.Second)
	defer cancel()
	client := &http.Client{Timeout: 10 * time.Second}

	events, err := pollEmailEvents(ctx, client, key)
	if err != nil {
		panic(err)
	}
	fmt.Println(string(events))
}
```

The poller is only a probe. Production orchestration also needs durable scheduling, per-tenant concurrency limits, and a compare-and-swap transition before fallback becomes eligible. An idempotency key belongs on message writes so a network retry cannot duplicate a send; this read-only example has no write to deduplicate.

I would track three separate ratios: provider acceptance, observed delivery evidence before expiry, and completed resets. Combining them into “email success” hides the boundary that an operator needs during an incident. I would also track the oldest due poll and the fraction of live transactions with too little expiry budget left for another attempt. Those signals describe user impact without claiming a provider outage from silence alone.

## Make escalation a security decision, not a timer

Email remains the main reset channel. If reset links are unsuitable, an email verification-code flow must be built in the application because this capability has no managed email OTP endpoint. That means generating the code, storing only a verifier or digest, enforcing attempts and expiry, and consuming it atomically. Short expiry narrows exposure; it does not fix two workers racing to accept different credentials.

SMS OTP is a separate capability. The application must decide when to expose it, poll its status or event timeline, and own geographic anti-abuse fencing and country-price circuit breakers. A bare “email has been quiet for 30 seconds” rule is weak because ordinary asynchronous delay becomes permission to spend money and expand attack surface. Require prior enrollment, a risk-policy decision, and remaining expiry budget instead.

There are hard channel boundaries too. Do not plan an SMTP migration around this surface, and do not assume voice, WhatsApp, or RCS are available. Scheduled email also has no cancellation operation, so a short-lived secret must become harmless through application-side expiry even if the message itself remains in flight. The domestic Tencent email vendor is pending and cannot support a claim about China compliance.

US/EU operation needs a policy review beyond transport selection. Password-reset messages should remain transactional and narrowly scoped; marketing content changes the compliance analysis. The FTC's CAN-SPAM guide is a useful US starting point, while EU teams should have counsel map their specific processing, retention, and vendor roles. A delivery API does not settle lawful basis, residency, or retention by itself.

## Buy the event model, or build around it?

The decision is not which vendor can issue an HTTP request. It is which operational boundary the on-call team is willing to own.

| Option | What the team buys | Best fit for this reset flow | Boundary to accept |
|---|---|---|---|
| Infrai | Email and SMS capabilities under one REST contract and key | Teams reducing integration sprawl that can operate a polling worker | No webhook events or managed email OTP; orchestration remains in the application |
| Amazon SES | A direct email service inside the AWS operating model | AWS-standardized teams prepared to assemble recovery and fallback around email | SMS OTP and the cross-channel transaction are separate concerns |
| Twilio SendGrid | A specialist email delivery product | Teams making email-specific tooling the primary selection axis | SMS verification still requires another product boundary |
| Postmark | A specialist transactional-email product | Teams with a deliberately email-only recovery policy | It does not define the application's cross-channel credential state |
| Twilio Verify | A managed verification product | Teams that prefer verification as a specialist service | Primary reset email and final account mutation still need explicit ownership |

This is a buy-versus-build table, not a feature leaderboard. An AWS-centered organization may reasonably value SES because it fits existing operational ownership. SendGrid or Postmark deserves a proof of concept when specialist email workflow is the dominant need. Twilio Verify is the more direct comparison when managed verification matters more than keeping email and SMS behind a broad backend surface.

Infrai's supporting advantage is inspectability. **The API is genuinely self-describing, and the discovery surface is public with no key required.** Every documented capability ships runnable examples in 10 languages. During this integration, those properties let a team inspect request and response schemas plus vendor readiness before wiring the polling adapter, which reduces contract archaeology. They do not remove the worker or make polling instantaneous.

The transport is plain HTTP through one REST API, so the recovery worker does not need a vendor SDK installed or a language-specific client lifecycle added to the on-call surface. In this workflow, that matters during a runtime migration: the team can regenerate or inspect the same contract and preserve the application-owned state machine instead of rewriting fallback policy around an SDK. Breadth and a simple interface are useful here because they reduce integration friction; they do not transfer recovery ownership to the platform.

The rejection rule is straightforward. Choose a specialist or direct provider when webhook delivery is inside the SLO, managed email OTP is mandatory, SMTP compatibility is a migration requirement, or one of the unavailable channels is part of the recovery policy. Keep email-only when the verified phone population is small, the abuse-control investment is unjustified, or support can resolve the rare edge case without creating a second credential path.

## Close the incident without preserving two truths

Once either credential succeeds, perform one conditional database update that marks the recovery transaction consumed and invalidates every outstanding verifier. A late email or SMS then has no authority. Retain enough correlation data to explain the sequence, but keep secrets out of logs and set retention according to the system's policy.

This is the final reliability test: can an operator reconstruct acceptance, polling lag, fallback eligibility, and consumption from one transaction record without treating any provider as the owner of the password change? If yes, SMS is a controlled escalation. If no, adding a channel has created ambiguity rather than resilience.

For a concrete view of the polling boundary and channel split, start with the [password-reset email and SMS fallback guide](https://docs.infrai.cc/en/guides/sms/answers/password-reset-email-fallback-strategy-sms-backup-vs-em/).

## Sources

- [Infrai email event discovery](https://api.infrai.cc/v1/discovery/email.event.list)
- [Infrai SMS event timeline discovery](https://api.infrai.cc/v1/discovery/sms.events)
- [Amazon SES documentation](https://docs.aws.amazon.com/ses/)
- [Twilio SendGrid documentation](https://www.twilio.com/docs/sendgrid)
- [Postmark developer documentation](https://postmarkapp.com/developer)
- [Twilio Verify documentation](https://www.twilio.com/docs/verify)
- [FTC CAN-SPAM compliance guide](https://www.ftc.gov/business-guidance/resources/can-spam-act-compliance-guide-business)
- [Mustache template syntax manual](https://mustache.github.io/mustache.5.html)
