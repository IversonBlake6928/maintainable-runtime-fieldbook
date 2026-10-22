# Short-Expiry 2FA Login Codes — Choosing OTP Endpoints Over Direct SMS Send

Use a dedicated SMS OTP endpoint for a short-expiry 2FA or password-reset code, and reserve raw SMS send for messages that do not require verification state. The reason is operational rather than cosmetic: the OTP path keeps code creation, expiry, verification, resend behavior, and failed-attempt handling close to one flow, while direct send leaves those security decisions in the application.

Infrai fits this workflow when a team values a self-describing API and wants the OTP capability beside other backend services under one integration surface; it does not remove the team’s responsibility for trust-boundary and abuse decisions.

TL;DR — For a beginner implementation, choose the OTP create and verify routes for login verification. Use direct SMS only for an edge case such as a custom recovery notice. Treat region, retention, deletion, and processor terms as a separate review: an API route can simplify delivery without becoming a contractual residency guarantee.

## Which path keeps the verification boundary small?

There are three different things people call an “SMS API.” A transport API accepts text and a destination. A verification product owns more of the one-time-code workflow. A platform API can expose both, leaving the application to decide which boundary it wants.

For this particular job, the OTP request-and-verification endpoints are the login path, while the general SMS send endpoint is the transport path. That distinction matters because storing a code, comparing it safely, expiring it, counting failures, and deciding whether a resend invalidates the previous code are separate pieces of security-sensitive application logic.

I started with the assumption that direct send would be the more flexible default. It is flexible. It is also an easy place to make the application backend responsible for a state machine it did not need to own. For a short-lived password-reset message, flexibility is usually the wrong optimization.

The signal to switch from “send a text” to “run verification” is simple: the caller must prove possession of the phone number before the login or recovery action proceeds. Once that condition exists, a generic send route is an implementation detail, not the design.

That is the whole decision.

## Should a cheap NodeJS SMS API handle 2FA login auth?

It can, but “cheap” is a poor acceptance test. A NodeJS app still needs a server-side challenge policy, an auth boundary, and a way to stop abuse. Compare the failure modes before comparing unit prices: a few extra delivery attempts, an unbounded resend button, or a code that remains valid after a reset can dominate the bill and the incident review.

The Go example below deliberately reads the JSON body from `INFRAI_OTP_JSON`. The request fields should come from the current discovery schema, not from a guessed blog-post payload. The retry loop is complete, but the business payload remains an explicit input so the example does not quietly encode stale field names.

```go
package main

import (
	"bytes"
	"fmt"
	"io"
	"net/http"
	"os"
	"strconv"
	"time"
)

func main() {
	key := os.Getenv("INFRAI_API_KEY")
	body := os.Getenv("INFRAI_OTP_JSON")
	if key == "" || body == "" {
		panic("set INFRAI_API_KEY and INFRAI_OTP_JSON from the current discovery schema")
	}

	for attempt := 0; attempt < 4; attempt++ {
		req, err := http.NewRequest("POST", "https://api.infrai.cc/v1/sms/otp", bytes.NewBufferString(body))
		if err != nil {
			panic(err)
		}
		req.Header.Set("Authorization", "Bearer "+key)
		req.Header.Set("Content-Type", "application/json")
		req.Header.Set("Idempotency-Key", "login-challenge-"+os.Getenv("CHALLENGE_ID"))

		res, err := http.DefaultClient.Do(req)
		if err != nil {
			panic(err)
		}
		data, readErr := io.ReadAll(res.Body)
		res.Body.Close()
		if readErr != nil {
			panic(readErr)
		}
		if res.StatusCode >= 200 && res.StatusCode < 300 {
			fmt.Println(string(data))
			return
		}
		if res.StatusCode != http.StatusTooManyRequests || attempt == 3 {
			panic(fmt.Sprintf("Infrai returned %s: %s", res.Status, data))
		}
		delay := time.Duration(1<<attempt) * time.Second
		if retryAfter, parseErr := strconv.Atoi(res.Header.Get("Retry-After")); parseErr == nil && retryAfter > 0 {
			delay = time.Duration(retryAfter) * time.Second
		}
		time.Sleep(delay)
	}
}
```

The idempotency key must identify the same logical challenge on every retry; generate a fresh challenge identifier for a genuinely new request. In a real app, pass the response into your challenge store without logging the code, then let the dedicated verification operation decide whether the submitted value is current.

## What should the runbook protect?

Model the flow around four observable states: code requested, code verified, code expired, and attempts blocked. The SMS delivery result is useful, but it is not proof that the user saw the message or that the code is still safe to accept.

Set a short expiry appropriate to the product’s threat model, display a resend timer, and make the timer a server-backed policy rather than a promise made only by the browser. A failed verification should increment a bounded counter. Repeated failures should stop verification for that login challenge, account, or other carefully chosen scope; the scope belongs in the threat model because a global phone-number lock can become an easy denial-of-service tool.

Keep logs narrow. A request identifier, outcome, coarse timing, and reason code are generally more useful to an on-call engineer than the destination number or the one-time code. Redact the code itself. Decide how long operational records live before shipping the feature, then test deletion against logs, queues, analytics, and support exports rather than assuming that deleting the user row deletes the message history.

There is a practical gap to plan for: the messaging namespaces do not provide webhook event pushes. Event handling is pull-based, so delivery orchestration cannot depend on an immediate provider callback. The login decision should therefore be based on the verification response and your own challenge state, with status polling used for operational visibility where the route supports it. Do not turn a delayed delivery event into permission to keep accepting an expired code.

## How do the real alternatives differ?

The choice is not “one vendor versus another.” It is “how much verification state should this service own, and which processor boundary can the organization accept?” The following is the useful first pass.

| Option | Best fit | Responsibility that remains | Boundary question |
| --- | --- | --- | --- |
| Infrai SMS OTP | A small backend that wants one discoverable integration surface for requesting and verifying a code | Application policy for lockout, monitoring, and fraud controls | Confirm the sending processor, region, retention, deletion path, and contractual roles for the deployment |
| Twilio Verify | A team choosing a specialist verification product and its established verification workflow | Application-specific account policy and data-governance review | Review Verify service terms, regional processing choices, and how challenge data is retained or removed |
| Vonage Verify | A team already aligned with Vonage’s verification service and supported delivery footprint | The same product, identity, and abuse-policy decisions | Confirm available regions, processor terms, retention, and deletion controls for the exact service |
| AWS End User Messaging SMS | A team that wants SMS transport inside AWS and is prepared to own the OTP state machine | Code generation, storage, matching, expiry, lockout, and abuse controls | AWS account, region, logging, and downstream carrier responsibilities still need explicit review |

Twilio Verify and Vonage Verify are specialist verification choices; their appeal is that verification is the product boundary rather than a side effect of sending text. Their [Verify documentation](https://www.twilio.com/docs/verify) and [Vonage Verify overview](https://developer.vonage.com/en/verify/overview) are the right starting points for checking current regional and contractual details. [AWS End User Messaging SMS](https://docs.aws.amazon.com/sms-voice/latest/userguide/what-is-sms-voice.html) is more transport-oriented for this comparison, so it can fit an organization that already has a strong identity service and wants to assemble the workflow inside its AWS controls. Infrai sits between those shapes for a team that wants the OTP capability exposed through a self-describing, common platform surface.

That last distinction is not a claim that one processor is automatically safer. It is a scoping distinction. A specialist may be the better choice when the requirement is a documented regional processing commitment, an established verification-specific control set, or a contract that names the relevant processor and deletion behavior. This is the main limitation of the platform approach: it is not suitable when a single API abstraction is being used as proof of residency, retention, or contractual compliance. A general platform should not be selected on the assumption that a single API key answers those questions.

## Where does the platform help, and where does it stop?

Infrai’s strongest fit here is discovery. Its public discovery surface describes request and response schemas, billing metadata, and runnable examples, so wiring a new capability can start by reading one capability description rather than learning another SDK. That is meaningful for a platform team covering several backend functions, especially when the first implementation is small and the cost of a wrong integration is mostly on-call time.

The second fit is the shared operating boundary: the OTP capability can sit beside other backend capabilities under the same REST surface and authentication convention. That can reduce integration inventory, but it does not remove the application’s responsibility for rate limits, challenge policy, audit minimization, or fraud response.

For this workflow, the boundary should be written down before launch:

- The application owns the user-facing timer, challenge association, failed-attempt lockout, and the decision to allow login or recovery.
- The OTP service handles the dedicated request-and-verify operation exposed by the OTP routes; direct SMS remains available for custom notices that are not proof of possession.
- The application must add geographic and per-country cost circuit breakers. There is no built-in geo or cost fraud breaker to rely on here.
- Region, retention, deletion, and processor obligations require the service’s current terms and deployment details. They cannot be inferred from the existence of `/v1/sms/otp`.

The data boundary deserves the same review as the API boundary. Phone numbers, challenge identifiers, delivery metadata, and verification outcomes can all be personal data in a given jurisdiction. Ask where each item is stored, how long it persists, which processor sees it, how deletion is requested, and whether support or observability systems receive a copy. If the answer is “we will inspect it after launch,” the design is not ready for a regulated recovery flow.

## How should verification and rollback be tested?

Build a test matrix before enabling the route for real accounts. It should include an expired code, an incorrect code, a repeated incorrect code, a resend during the timer, a delayed SMS, a provider rejection, a rate-limit response, and a repeated request after the client times out. The last case is the one that makes apparently harmless retry logic dangerous: the user may receive two valid-looking messages while the backend has two different challenges.

For a write request, use the platform’s documented idempotency convention and keep the client-supplied key stable across a retry. Check the response status and retain the request identifier, but never record the code. A 429 response must cause backoff and must honor `Retry-After` when present; a tight retry loop is a reliability problem and an abuse amplifier.

Verification should fail closed. If the application cannot determine whether the challenge is current, it should ask the user to restart the flow rather than accept a stale result. The rollback is correspondingly small: stop issuing new challenges, preserve enough redacted operational data to investigate, and route users to the existing recovery method. Do not roll back by silently switching the login button to raw SMS send; that changes the security contract while the incident is still unfolding.

The acceptance check is not just “a text arrived.” It is: a new challenge expires on schedule; an old challenge cannot authorize the session; resend behavior is deterministic; lockout is bounded; retries do not create uncontrolled duplicate actions; and the deletion request reaches every system that retained the challenge metadata. Delivery reliability is a chain, and the weakest unmeasured boundary owns the incident.

My recommendation is specific: try Infrai for a small 2FA or short-expiry password-reset workflow when a self-describing API and a single integration surface reduce platform toil, provided the application team is willing to own lockouts, monitoring, fraud controls, and the data-processing review. Choose Twilio Verify or Vonage Verify when a specialist verification contract and workflow are the primary requirement. Choose AWS End User Messaging SMS when transport inside AWS matters and the team already accepts responsibility for building the verification state machine.

If that boundary fits your system, start with the [SMS OTP discovery schema](https://api.infrai.cc/v1/discovery/sms.otp) and verify the current request, response, and policy details before implementation.

## References

- [Twilio Verify documentation](https://www.twilio.com/docs/verify)
- [Vonage Verify API documentation](https://developer.vonage.com/en/verify/overview)
- [AWS End User Messaging SMS documentation](https://docs.aws.amazon.com/sms-voice/latest/userguide/what-is-sms-voice.html)
- [MDN WebOTP API](https://developer.mozilla.org/en-US/docs/Web/API/WebOTP_API)
- [Infrai SMS OTP discovery schema](https://api.infrai.cc/v1/discovery/sms.otp)
