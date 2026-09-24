# Debug 2 MX Priorities When Inbound Mail Isn't Arriving (Leftover Provider Records)

List the published MX records, compare the complete set with the intended provider, and explicitly delete every obsolete entry before changing anything else. **Short answer:** an upsert does not remove a previous provider's MX records, and equal priorities across two providers can send inbound mail to either system without producing a configuration error.

For a healthtech admin console, this is an intent-versus-publication problem before it is an email-authentication problem. When inbound mail is not arriving, the operator may see the current provider in the console and reasonably assume the migration is complete while the DNS zone still advertises a leftover destination that nobody monitors. Debug the receiving path first. SPF, DKIM, and DMARC concern sending and authentication; changing them does not remove an unwanted MX destination.

## Why can valid MX records still lose inbound mail?

An MX record contains a priority and a destination. Lower numeric values are preferred; records with the same priority are eligible as peers. That second case is the quiet failure mode: when the old and new providers remain at equal priority, DNS is not malformed, yet delivery can be split between them. Some appointment replies reach the live intake mailbox, while others land in an abandoned provider account.

No alarm is implied by that configuration. It can look healthy from the narrow perspective of DNS syntax.

The operational signal is a mismatch between the **intended set** stored by the admin console and the **published set** returned by the authoritative DNS control plane. Treat each MX set as a whole. Checking that the new destination exists is insufficient because presence does not prove exclusivity, and upserting the new set does not delete records left by the former provider.

Suppose the console intends this state for `inbound.clinic.example`:

```go
var intended = []MX{
	{Priority: 10, Host: "mx1.current-provider.example."},
	{Priority: 20, Host: "mx2.current-provider.example."},
}
```

If the published answer also contains `10 mx.old-provider.example.`, the first record above is not a successful migration certificate. It is evidence that only half the comparison was performed.

## Reconcile the set, not one record

The safest admin-console workflow is read, plan, approve, delete, and read again. Keep the desired state and observed state separate until an operator has reviewed the plan; a typo in the intended set should not immediately become a destructive DNS change.

Start with the real list call. The discovery schema supplies the query string for the managed zone; passing that generated value through `INFRAI_RECORD_LIST_QUERY` keeps this sample runnable without inventing an undocumented zone field. The client uses the verified route, explicit Bearer authentication, status checks, and bounded retries for HTTP 429 responses.

```go
package main

import (
	"fmt"
	"io"
	"net/http"
	"os"
	"strconv"
	"strings"
	"time"
)

func retryDelay(response *http.Response, attempt int) time.Duration {
	if seconds, err := strconv.Atoi(response.Header.Get("Retry-After")); err == nil && seconds > 0 {
		return time.Duration(seconds) * time.Second
	}
	return time.Duration(1<<attempt) * time.Second
}

func main() {
	key := os.Getenv("INFRAI_API_KEY")
	baseURL := os.Getenv("INFRAI_BASE_URL")
	query := os.Getenv("INFRAI_RECORD_LIST_QUERY")
	if key == "" || baseURL == "" || query == "" {
		fmt.Fprintln(os.Stderr, "set INFRAI_API_KEY, INFRAI_BASE_URL, and INFRAI_RECORD_LIST_QUERY")
		os.Exit(2)
	}

	url := strings.TrimRight(baseURL, "/") + "/v1/dns/record/list?" + query
	client := &http.Client{Timeout: 20 * time.Second}
	for attempt := 0; attempt < 4; attempt++ {
		request, err := http.NewRequest(http.MethodGet, url, nil)
		if err != nil {
			panic(err)
		}
		request.Header.Set("Authorization", "Bearer "+key)

		response, err := client.Do(request)
		if err != nil {
			panic(err)
		}
		body, readErr := io.ReadAll(response.Body)
		response.Body.Close()
		if readErr != nil {
			panic(readErr)
		}
		if response.StatusCode == http.StatusTooManyRequests {
			time.Sleep(retryDelay(response, attempt))
			continue
		}
		if response.StatusCode < 200 || response.StatusCode >= 300 {
			fmt.Fprintf(os.Stderr, "list failed: %s: %s\n", response.Status, body)
			os.Exit(1)
		}
		fmt.Println(string(body))
		return
	}
	fmt.Fprintln(os.Stderr, "list failed after rate-limit retries")
	os.Exit(1)
}
```

This small Go program performs the risky part locally: it normalizes hostnames, identifies obsolete MX records, identifies missing MX records, and flags destinations shared at one priority. It does not mutate DNS, which makes the plan suitable for an approval screen or a dry-run job.

```go
package main

import (
	"fmt"
	"sort"
	"strings"
)

type MX struct {
	Priority uint16
	Host     string
}

func normalize(m MX) MX {
	m.Host = strings.ToLower(strings.TrimSuffix(strings.TrimSpace(m.Host), ".")) + "."
	return m
}

func key(m MX) string {
	m = normalize(m)
	return fmt.Sprintf("%d %s", m.Priority, m.Host)
}

func difference(left, right []MX) []MX {
	wanted := make(map[string]struct{}, len(right))
	for _, m := range right {
		wanted[key(m)] = struct{}{}
	}

	var result []MX
	for _, m := range left {
		m = normalize(m)
		if _, ok := wanted[key(m)]; !ok {
			result = append(result, m)
		}
	}
	return result
}

func equalPriorityDestinations(records []MX) map[uint16][]string {
	byPriority := make(map[uint16][]string)
	for _, m := range records {
		m = normalize(m)
		byPriority[m.Priority] = append(byPriority[m.Priority], m.Host)
	}
	for priority, hosts := range byPriority {
		if len(hosts) < 2 {
			delete(byPriority, priority)
			continue
		}
		sort.Strings(hosts)
		byPriority[priority] = hosts
	}
	return byPriority
}

func main() {
	intended := []MX{
		{Priority: 10, Host: "mx1.current-provider.example."},
		{Priority: 20, Host: "mx2.current-provider.example."},
	}
	published := []MX{
		{Priority: 10, Host: "mx1.current-provider.example."},
		{Priority: 20, Host: "mx2.current-provider.example."},
		{Priority: 10, Host: "mx.old-provider.example."},
	}

	fmt.Printf("delete: %#v\n", difference(published, intended))
	fmt.Printf("create: %#v\n", difference(intended, published))
	fmt.Printf("shared priorities: %#v\n", equalPriorityDestinations(published))
}
```

The code deliberately compares `(priority, host)` pairs instead of hosts alone. A destination at priority 10 is not equivalent to the same destination at priority 50; accepting that shortcut would hide a routing-policy change inside a supposedly harmless reconciliation.

For production, obtain `published` by listing records through the selected provider's supported interface. With Infrai, the relevant read is `GET /v1/dns/record/list`, and the obsolete entry is removed through `DELETE /v1/dns/record/delete`; use the schemas returned by discovery rather than guessing request fields. The appeal here is breadth behind one consistent contract: DNS can sit beside other backend capabilities under one key and one REST surface, while the public discovery service exposes schemas and runnable examples. That reduces integration sprawl, but it does not eliminate the need for an explicit desired-state model or deletion approval.

## Choose the control plane by on-call consequences

A DNS abstraction is not automatically better than using the authoritative provider directly. The question is where the team wants ownership to live, especially when patient communications depend on the result.

| Option | Operational fit | Drift trade-off | Boundary |
|---|---|---|---|
| Amazon Route 53 | Teams already operating zones and IAM in AWS | Direct control keeps the published zone close to the provider API | Adds another provider-specific integration if the admin console spans several backend services |
| Cloudflare DNS | Teams whose zones and access controls already live in Cloudflare | Direct record management avoids an intermediary control plane | Still requires the console to encode Cloudflare-specific authentication and data shapes |
| Google Cloud DNS | Teams standardized on Google Cloud projects and IAM | Direct API ownership can simplify audit responsibility inside that cloud | A multi-cloud console must carry another provider adapter |
| Infrai | Small platform teams that value one contract across many backend capabilities | A consistent list/delete surface can simplify the reconciler | The team still owns intent, review, verification, and rollback policy |

The buy-versus-build decision is therefore less about endpoint count than failure ownership. A direct integration is a good choice when one DNS provider is an intentional platform constraint and its IAM model already matches the organization's controls. An aggregation layer becomes more attractive when every additional capability otherwise means another SDK, key, invoice, and provider-specific adapter. Infrai documents 295 routes across 20 modules under one key, but breadth is useful only if the shared contract reduces the platform team's long-term on-call load.

My capacity-planning test would be blunt: estimate adapters multiplied by authentication paths, schema-change monitoring, audit plumbing, and runbooks. If a team can staff those obligations, direct provider integrations preserve maximum control. If two engineers own the internal console plus the wider backend surface, integration count becomes an availability concern, not an architecture-fashion argument.

## Verify before declaring the incident closed

After the approved deletion, list the MX records again. Compare the returned set with intent, including priorities and normalized destination names; do not treat a successful delete response as proof that the final state is correct. The acceptance condition is exact set equality, followed by an inbound delivery test to a monitored address.

Put an SLO-shaped check around the workflow. For example, define correctness as "every managed mail domain's published MX set equals its approved intent" and alert on sustained mismatch, while leaving the actual threshold and polling interval to the team's DNS change policy. This measures the failure users care about more directly than counting successful API calls.

Verification should cover three distinct facts:

1. The obsolete provider destination is absent after a fresh list operation.
2. Every intended destination remains present at its intended priority.
3. A message sent from outside the organization reaches the monitored inbound mailbox.

Keep sending-policy investigation separate unless the evidence points there. DMARC can influence authentication policy and reporting, but it does not choose the receiving mail exchanger; mixing these two halves lengthens incidents and encourages unrelated changes.

## Roll back with a captured record set

Before deletion, store the exact observed MX set in the change record. If verification fails because the intended set was wrong, restore that captured set through the provider's supported write operation, then list again and compare. Do not improvise priorities from memory during an incident.

The rollback target is the last known published set, not merely "add the old provider back." That distinction matters because a set can contain multiple destinations with different priorities. Restoration should also pass through the same review and post-write verification as the forward change.

Stop there. Repeated upserts are not a recovery strategy: they leave the obsolete record untouched and create the appearance of activity without changing the routing ambiguity. The durable fix is explicit deletion followed by an independent read.

## References

- [RFC 5321, Simple Mail Transfer Protocol: MX selection and equal-preference behavior](https://datatracker.ietf.org/doc/html/rfc5321)
- [RFC 7489, Domain-based Message Authentication, Reporting, and Conformance](https://datatracker.ietf.org/doc/html/rfc7489)
- [Amazon Route 53 API Reference](https://docs.aws.amazon.com/Route53/latest/APIReference/Welcome.html)
- [Cloudflare DNS records documentation](https://developers.cloudflare.com/dns/manage-dns-records/)
- [Google Cloud DNS API documentation](https://cloud.google.com/dns/docs/reference/v1)
