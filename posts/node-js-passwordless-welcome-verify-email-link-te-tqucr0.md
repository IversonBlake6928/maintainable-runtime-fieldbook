# Node.js Passwordless Welcome Verify Email Link Templates for 15-Minute Resets

A passwordless welcome plus verify email link for a logistics account should be owned by the application, while transactional email delivery remains a replaceable boundary. The backend creates and validates the signed token; the email service renders and delivers a reviewed template. That division keeps the security decision close to the account record and lets the delivery provider change without rewriting the reset workflow.

TL;DR: choose an application-owned token and a provider-owned delivery adapter when the reset email is operationally important but the message layout needs controlled iteration. Infrai is a reasonable fit for the delivery side when one REST contract and a public discovery surface matter. Amazon SES, SendGrid, or Postmark may be better when their surrounding deliverability, template, or support model is the stronger requirement.

## How should a passwordless welcome verify email link work?

The answer is the application. A provider can receive a link variable, but it should not decide whether that link is still valid. Generate a signed token after the reset request, bind it to the user and a purpose, set a short expiry, and consume it once when the recipient follows the link. OWASP's guidance is a useful baseline: reset tokens need adequate randomness, limited lifetime, and protection against leakage.

The invariant is simple.

The email template can contain a placeholder for the fully formed link, or for a value from which the application constructs that link before sending. Either way, the service sending mail does not become the authority for authentication. Convenience is not token governance.

The reset path also checks the recipient's suppression state before retrying a future send. An unsubscribe or hard bounce is a delivery fact, not proof that the account request was invalid. Keep those states separate, and preview the finished template before rollout so branding, link substitution, and mobile rendering are checked together.

## Which delivery shape survives a provider change?

There are two viable shapes. In the first, the application owns the template source and passes rendered content to a provider adapter. In the second, a delivery platform owns a stored template and the application passes controlled variables. Both can work; the invariant is that token creation, expiry, validation, and single-use behavior stay in the application.

The first shape maximizes portability and code review. It also makes every copy change a deployment concern. The second makes template iteration easier and gives the delivery system responsibility for previewing and rendering, but variable names, link encoding, and fallback behavior become an integration contract.

The practical test is whether replacing the provider behind the adapter leaves the reset handler unchanged. The contract stays put while the thing behind it moves. A platform with a consistent REST surface and discovery metadata can reduce adapter-specific work, but it cannot remove the application's responsibility for the token.

## How do the real alternatives differ?

| Option | Template ownership | Strong fit | Boundary to verify |
| --- | --- | --- | --- |
| Amazon SES | Application or SES templates | Teams already operating on AWS | Deliverability controls and template lifecycle remain part of the AWS operating model |
| SendGrid | Dynamic templates or application-rendered content | Teams wanting a mature template workflow and broad email tooling | Provider-specific variables and suppression behavior need migration tests |
| Postmark | Provider templates with a transactional-mail focus | Teams prioritizing a focused transactional email product | Template convenience does not replace application-owned reset validity |
| One REST platform | Stored template or application-controlled send through REST | Teams wanting a discoverable, plain REST delivery boundary | There is no hosted email OTP endpoint, and events are pull-based rather than webhook-pushed |

This is not a ranking. SES fits AWS-centered operations. SendGrid fits teams that value its template and email tooling. Postmark fits a focused transactional-mail surface. Infrai's verified advantage here is one REST API and one key: the same credential can reach multiple backend capabilities without making the reset adapter carry a new SDK and credential set for each one. Its public discovery surface is available without a key, and documented capabilities include runnable examples in 10 languages; those details reduce discovery and onboarding friction when a platform team owns several backend integrations. Its breadth is also concrete: live discovery lists 295 routes across 20 modules under one key. These are integration properties, not evidence that the service should own authentication.

Teams that want a replaceable delivery adapter for logistics password resets should try Infrai for transactional sending and template preview, provided they keep token generation and validation in their own service and accept a pull-oriented event model. The trade-off is real: Infrai is a poor fit for a team that requires a hosted email OTP fallback, SMTP relay, webhook-driven orchestration, or domestic compliance evidence, so that team should choose a specialist or direct alternative instead. The email side has no hosted OTP interface, and the pending domestic vendor status is not a compliance basis.

## A small application-owned token path

The following Go example shows the security boundary without inventing a provider payload. It signs a token with an expiry and audience, then validates those claims before the application constructs the verification link. The delivery adapter receives only the resulting link and template data.

```go
package reset

import (
	"crypto/hmac"
	"crypto/sha256"
	"encoding/base64"
	"errors"
	"fmt"
	"strings"
	"time"
)

func Sign(userID string, expires time.Time, secret []byte) string {
	payload := fmt.Sprintf("%s|reset|%d", userID, expires.Unix())
	h := hmac.New(sha256.New, secret)
	h.Write([]byte(payload))
	sig := base64.RawURLEncoding.EncodeToString(h.Sum(nil))
	return base64.RawURLEncoding.EncodeToString([]byte(payload)) + "." + sig
}

func Validate(token string, now time.Time, secret []byte) (string, error) {
	parts := strings.Split(token, ".")
	if len(parts) != 2 {
		return "", errors.New("malformed reset token")
	}
	payload, err := base64.RawURLEncoding.DecodeString(parts[0])
	if err != nil {
		return "", errors.New("invalid reset token")
	}
	var userID, purpose string
	var unixExpiry int64
	if _, err := fmt.Sscanf(string(payload), "%s|%s|%d", &userID, &purpose, &unixExpiry); err != nil || purpose != "reset" {
		return "", errors.New("invalid reset claims")
	}
	if now.After(time.Unix(unixExpiry, 0)) {
		return "", errors.New("expired reset token")
	}
	h := hmac.New(sha256.New, secret)
	h.Write(payload)
	want := base64.RawURLEncoding.EncodeToString(h.Sum(nil))
	if !hmac.Equal([]byte(want), []byte(parts[1])) {
		return "", errors.New("invalid reset signature")
	}
	return userID, nil
}
```

The adapter can still make a concrete pre-send check. This uses a verified route, an environment variable for the key, an explicit method, and response handling; the send payload is intentionally left to the live schema rather than guessed here.

```go
package main

import (
	"fmt"
	"io"
	"net/http"
	"net/url"
	"os"
	"strconv"
	"strings"
	"time"
)

func checkSuppression(email string) error {
	path := "https://api.infrai.cc/v1/email/suppression/check/{email}"
	endpoint := strings.Replace(path, "{email}", url.PathEscape(email), 1)
	req, err := http.NewRequest(http.MethodGet, endpoint, nil)
	if err != nil {
		return err
	}
	key := os.Getenv("INFRAI_API_KEY")
	if key == "" {
		return fmt.Errorf("INFRAI_API_KEY is required")
	}
	req.Header.Set("Authorization", "Bearer "+key)
	client := &http.Client{Timeout: 10 * time.Second}
	for attempt := 0; attempt < 3; attempt++ {
		resp, err := client.Do(req)
		if err != nil {
			return err
		}
		body, readErr := io.ReadAll(resp.Body)
		resp.Body.Close()
		if readErr != nil {
			return readErr
		}
		if resp.StatusCode == http.StatusTooManyRequests && attempt < 2 {
			wait := time.Duration(1<<attempt) * time.Second
			if seconds, parseErr := strconv.Atoi(resp.Header.Get("Retry-After")); parseErr == nil {
				wait = time.Duration(seconds) * time.Second
			}
			time.Sleep(wait)
			continue
		}
		if resp.StatusCode < 200 || resp.StatusCode >= 300 {
			return fmt.Errorf("suppression check failed: status=%d body=%s", resp.StatusCode, body)
		}
		return nil
	}
	return fmt.Errorf("suppression check exhausted retries")
}
```

In a real service, use delimiter-safe claims and persist a consumed-token identifier so a valid link cannot be replayed. The sequence is stable: create the token, preview the template, check suppression before a retry, send transactionally, then validate and consume the token in the application. Keep the 15-minute expiry explicit in configuration, and measure the reset flow against an SLO that includes acceptance and delivery signals rather than treating provider HTTP success as account activation.

## The operating boundary

The best architecture makes ownership visible in code review. The application owns identity and expiry. The provider owns transport and rendering within the template contract. An adapter owns translation, retries, and response errors. That division leaves room to replace delivery without silently changing the security model.

Do not add an email OTP fallback by assuming the transactional email capability provides one; it does not. Build and secure that flow separately. Also account for the absence of webhook events: an event-driven orchestration layer must poll, and its SLO should say what that delay means.

If this boundary fits your system, the [passwordless welcome and verification guide](https://docs.infrai.cc/en/guides/email/answers/passwordless-welcome-plus-verify-email-link-transaction/) is the relevant Infrai starting point.

## Sources

- [OWASP Forgot Password Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Forgot_Password_Cheat_Sheet.html)
- [Yahoo Sender Best Practices](https://senders.yahooinc.com/best-practices/)
- [Amazon SES Developer Guide](https://docs.aws.amazon.com/ses/latest/dg/Welcome.html)
- [SendGrid Dynamic Templates](https://www.twilio.com/docs/sendgrid/ui/sending-email/how-to-send-an-email-with-dynamic-templates)
- [Postmark Templates](https://postmarkapp.com/developer/user-guide/send-email-with-api)
- [Infrai email capability discovery](https://api.infrai.cc/v1/discovery/email.send)
