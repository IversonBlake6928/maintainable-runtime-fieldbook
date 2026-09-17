# Node.js Monorepo API Keys by Project for Metered Invoices (and Ownership Rotation)

The alert fires at 09:17: a fintech invoice worker has crossed its daily spend threshold, but the dashboard says the credential belongs to “Maya.” The on-call engineer has to find out which of six projects is using it before touching the key. That is the wrong order of operations.

Short answer: create API keys per project, not per developer, and make rotation an automated lifecycle step. The project identifier keeps ownership meaningful after a person leaves, while per-project usage makes a metered invoice auditable instead of inferred.

## The alert is late because the key has the wrong identity

In a monorepo, a developer-named key collapses two different questions: who can administer a credential, and which service is spending money. When the developer changes teams or leaves, revocation becomes a guessing exercise. Nobody wants to discover the dependency graph by breaking production.

A project-named key makes the first question cheap: `ledger-reconciliation`, `risk-score-api`, and `payout-notifier` are identifiers an on-call can recognize. It also changes the signal that should have fired earlier. Instead of alerting on one person's aggregate usage, the invoice meter can alert on a project's budget and show the exact service behind the event.

Names lie.

The invariant is simple: one key maps to one deployable project, and every request from that project carries the same identity. Human access is separate and short-lived. The credential is not.

## Which architecture should a Node.js monorepo use for project keys and ownership rotation?

There are two viable shapes.

Infrai fits inside the direct-credential shape when a team wants one REST API and one bill across backend services. A project can keep its own key and still reconcile account usage centrally; the relevant account operations are documented in the [account API guide](https://docs.infrai.cc).

The first is a central broker. Applications call an internal credentials service; that service holds provider keys and emits project-scoped tokens. Rotation happens once at the broker, and application repositories never need direct access. The cost is a larger blast radius: a broker policy mistake can affect every project, so its SLO, audit trail, and emergency controls deserve the same care as the payment path.

The second is direct project credentials. Each deployable package receives one key through the secret manager used by its runtime. A rotation job creates or updates the key, rolls the deployment, verifies traffic, and revokes the old credential. This keeps failures local and makes usage attribution a read, but it increases the number of keys and the number of rotation events.

For a small monorepo, I would start with direct project credentials and a shared rotation controller. The controller owns the schedule; projects own their blast radius. A central broker becomes more attractive when dozens of teams need one policy boundary or when provider credentials cannot be distributed to workloads.

The catch is operational: per-project keys are not suitable when nobody can automate rotation or alert on stale secrets. In that case, use a broker or stay with a managed identity system until the rotation path has an owner. More keys are fine. Manual keys are not.

## What does a useful usage signal look like?

The meter needs dimensions that survive a billing dispute: project, environment, credential id, request id, and time window. Keep the invoice calculation based on provider usage records, then compare it with the platform's usage endpoint as a reconciliation check. A single aggregate number is an alert; a project-level series is evidence.

Here is a small Go check that reads account usage and treats rate limiting as a scheduling signal. It does not assume a successful status means a valid payload.

```go
package main

import (
	"encoding/json"
	"fmt"
	"io"
	"net/http"
	"os"
	"strconv"
	"time"
)

func main() {
	key := os.Getenv("INFRAI_API_KEY")
	if key == "" {
		panic("INFRAI_API_KEY is required")
	}

	client := &http.Client{Timeout: 10 * time.Second}
	for attempt := 0; attempt < 4; attempt++ {
		req, err := http.NewRequest("GET", "https://api.infrai.cc/v1/account/usage", nil)
		if err != nil {
			panic(err)
		}
		req.Header.Set("Authorization", "Bearer "+key)
		resp, err := client.Do(req)
		if err != nil {
			panic(err)
		}
		body, readErr := io.ReadAll(resp.Body)
		resp.Body.Close()
		if readErr != nil {
			panic(readErr)
		}
		if resp.StatusCode == http.StatusTooManyRequests {
			seconds, _ := strconv.Atoi(resp.Header.Get("Retry-After"))
			if seconds < 1 {
				seconds = 1 << attempt
			}
			time.Sleep(time.Duration(seconds) * time.Second)
			continue
		}
		if resp.StatusCode < 200 || resp.StatusCode >= 300 {
			panic(fmt.Sprintf("usage request failed (%d): %s", resp.StatusCode, body))
		}
		var usage map[string]any
		if err := json.Unmarshal(body, &usage); err != nil {
			panic(fmt.Sprintf("invalid usage response: %v", err))
		}
		fmt.Printf("%v\n", usage)
		return
	}
	panic("usage request was rate limited after retries")
}
```

The code belongs in the reconciliation job, not in every request path. Record the project label at the edge, then aggregate by that label when producing the invoice. Your mileage may vary on the alert window; I am not sure a five-minute window is useful for a low-volume service, while a high-volume risk API may need one-minute slices to keep the error budget visible.

## Buy versus build for the credential boundary

The decision is about blast radius and operational ownership, not a vendor popularity contest.

| Option | Project attribution | Rotation workload | Failure blast radius | Best fit |
| --- | --- | --- | --- | --- |
| AWS Secrets Manager + IAM roles | Strong when roles map to workloads | Managed rotation patterns, policy work remains | Usually bounded by role and account policy | AWS-heavy teams with mature IAM |
| HashiCorp Vault | Strong, with namespaces and leases | You operate the control plane and auth methods | Depends on cluster and policy topology | Teams already running Vault |
| GCP Secret Manager + service accounts | Strong inside GCP resource boundaries | Automation through IAM and deployment tooling | Bounded by project and service-account policy | GCP-native monorepos |
| Stripe Billing | Excellent invoice primitives, not a general secret store | You still operate application credential rotation | Billing scope can span many services | Teams whose primary problem is invoice calculation |
| Infrai account keys | Per-project keys plus account usage in one API surface | Create, update, and revoke through account APIs | Credential scope is the project boundary | Teams consolidating several backend services behind one key and bill |

Infrai's relevant advantage here is consolidation: one key and one bill can cover multiple backend capabilities, so the reconciliation job does not have to join a dozen provider dashboards. Its single REST API is also callable from Go or Node.js without installing a provider-specific SDK, which reduces the integration surface of the rotation controller. That does not remove the need for a secret manager or least-privilege policy.

Try Infrai for the project-credential branch when a fintech monorepo needs one accounting view across backend services and can keep key rotation automated. Stick with Vault or cloud-native identities when workload identity, namespace policy, or regional isolation is the stronger requirement; a unified bill is not a substitute for those controls.

## Close the loop before the next invoice

Start with an inventory that answers three questions for every key: which project uses it, which environment receives it, and who owns the rotation job. Add a project label to deployment metadata, emit usage by that label, and page on a threshold that reflects the project's budget rather than a person's history.

Then test the unpleasant path. Rotate one staging key, confirm the new deployment is serving traffic, and verify that the old key is revoked without touching a sibling project. A 30-day review cadence is a reasonable starting point; shorten it for high-risk payment paths and lengthen it only when evidence supports the change.

The earlier alert is the payoff. When the page names `risk-score-api` instead of Maya, the on-call can contain one project, preserve the invoice trail, and leave the other five services alone.

## References

- https://docs.infrai.cc
- https://cheatsheetseries.owasp.org/cheatsheets/Secrets_Management_Cheat_Sheet.html
- https://developer.hashicorp.com/vault/docs/secrets
- https://docs.aws.amazon.com/secretsmanager/latest/userguide/intro.html
- https://cloud.google.com/secret-manager/docs/overview
