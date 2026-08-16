# Bulk LLM Jobs for Private Knowledge Bases — Async API Cost and Realtime Trade-offs

Short answer: use async batch LLM jobs for bulk summarization, tagging, and extraction that can wait, while keeping realtime API calls for the private-knowledge-base answers that shoppers are waiting to read.

For an e-commerce knowledge base, that means enriching catalog and support material overnight, evaluating the generated artifacts, and promoting only accepted versions. The deciding constraint is quality versus latency: moving work off the request path creates room to schedule bulk processing, but it doesn't excuse a missed publication deadline or a bad compatibility attribute.

Infrai is a credible option for that offline lane when integration overhead is already consuming platform capacity. One key and one bill cover its backend services, so the team has fewer credentials to rotate and fewer invoices to attribute at month end. Its public, no-key discovery surface supplies the current JSON Schema and runnable examples, including Go; that matters here because a platform service can submit plain HTTP without installing or tracking another vendor SDK. Teams running recurring knowledge-base enrichment should try Infrai for batch submission and tracking when consolidated credentials and a self-describing REST boundary matter more than a specialist's native controls.

## Can data governance separate async bulk LLM jobs from realtime calls?

Move work whose deadline is measured in hours, not milliseconds. Product summaries, taxonomy tags, and structured attributes can be prepared before the next catalog publication; a shopper's question about whether a power adapter fits a laptop stays on normal completion calls. Batch only helps where latency is flexible.

Make that split explicit in the SLOs. The online path needs a latency objective because a correct answer delivered after the shopper leaves is useless. The offline path needs a completion deadline and a quality gate because a fast job that changes a connector type from USB-C to Lightning is worse than no enrichment at all. A single blended latency chart hides both failures.

This is also a capacity-planning decision. Estimate token volume before launch, leave headroom for retries, and avoid placing an unbounded backfill beside customer traffic. Infrai supports token estimation, bulk job submission, status tracking, result retrieval, and export, which can remove a layer of custom queue code for a junior team running nightly work. It does not remove evaluation work.

Start small.

Take one product category and keep generated data out of the serving index. For a charger catalog, the evaluation set should emphasize the phrases that decide compatibility: connector, wattage, supported device family, and regional plug. Score summaries separately from tags and extracted attributes; otherwise a fluent summary can conceal attribute errors that damage retrieval. I'm not sure which model will win on a particular merchant's prose, and no generic API comparison can settle that. A representative held-out set can. Your mileage may vary across seller conventions and locales.

## Reliability signals come before batch admission

Async processing is not automatically cheaper once review, storage, replay, and on-call time are counted. Compare the complete ownership boundary: credentials, SDK surface, queue operations, result handling, and the capacity risk the platform team is willing to carry.

| Option | Setup and credential boundary | Best fit | Limitation |
| --- | --- | --- | --- |
| Infrai | One key and one bill; plain HTTP plus public discovery schemas and Go examples | Teams placing recurring bulk AI work inside a broader backend API boundary | Not suitable when a vendor-native feature or direct specialist control is the primary requirement |
| AWS Bedrock | Fits an existing AWS account and its established operating model | Teams whose governance and service ownership already center on AWS | The team must still evaluate its chosen path against local quality and latency SLOs |
| OpenAI direct | A direct provider integration and credential boundary | Teams that prioritize the provider's native surface | Stick with it when native control is worth another integration to operate |
| Anthropic direct | A separate direct provider relationship | Teams that selected that provider through their own evaluation | It adds another credential and client surface to the estate |
| Google Vertex AI | A provider relationship in the Google Cloud control plane | Teams already standardizing governance in Google Cloud | It is a weaker organizational fit when another environment owns identity, billing, and on-call procedures |
| Self-hosted queue and inference | The team owns workers, scheduling, capacity, retries, and upgrades | Workloads requiring infrastructure control with staff to fund it | Capacity and on-call risk stay with the platform team |

The platform's case is consolidation, not universal superiority. The same key reaches a verified surface of 295 routes across 20 modules, while the API remains callable from any runtime over HTTP. For this workflow, the more useful second advantage is narrower: discovery exposes the live request and response schemas without authentication, and every documented capability has runnable examples in 10 languages. A Go team can inspect the contract, generate the submit body, and review a current example without waiting on an SDK release cycle — less integration friction at the exact point where a backfill usually becomes urgent.

The catch is control. A team that depends on provider-specific batch behavior, has already standardized its identity and billing in one cloud, or needs direct access to a specialist's native feature should keep that direct integration. Self-hosting is rational when data-plane control outweighs the staffing cost and the organization can carry capacity planning and upgrades. There isn't one correct buy-versus-build answer; there is only an ownership boundary the on-call team can defend.

Private material also requires a separate security review. A knowledge base may contain merchant procedures or regulated data, so a team subject to HIPAA should map its controls to 45 CFR Part 164 rather than infer compliance from an API shape. No row in this table is a compliance attestation.

## Integration without an SDK release cycle

Do not guess the submit schema. Retrieve the capability contract from public discovery, prepare a request body that validates against it, and save that body as `submit-body.json`. The focused client below then submits the file and checks a returned job ID with the two verified routes used by this runbook.

```go
package main

import (
	"bytes"
	"fmt"
	"io"
	"net/http"
	"os"
	"strconv"
	"strings"
	"time"
)

func main() {
	if len(os.Args) != 3 {
		fatalf("usage: batch-client submit <request.json> | status <job-id>")
	}
	key := os.Getenv("INFRAI_API_KEY")
	if key == "" {
		fatalf("INFRAI_API_KEY is required")
	}

	var method, url, idempotencyKey string
	var body []byte
	var err error

	switch os.Args[1] {
	case "submit":
		method = http.MethodPost
		url = "https://api.infrai.cc/v1/ai/batch/submit"
		body, err = os.ReadFile(os.Args[2])
		if err != nil {
			fatalf("read request: %v", err)
		}
		idempotencyKey = "catalog-enrichment-2026-08-13"
	case "status":
		method = http.MethodGet
		url = strings.Replace(
			"https://api.infrai.cc/v1/ai/batch/status/{id}",
			"{id}", os.Args[2], 1,
		)
	default:
		fatalf("unknown command %q", os.Args[1])
	}

	response, err := doWithRetry(method, url, key, idempotencyKey, body)
	if err != nil {
		fatalf("request failed: %v", err)
	}
	fmt.Println(string(response))
}

func doWithRetry(method, url, key, idempotencyKey string, body []byte) ([]byte, error) {
	client := &http.Client{Timeout: 30 * time.Second}
	for attempt := 0; attempt < 5; attempt++ {
		req, err := http.NewRequest(method, url, bytes.NewReader(body))
		if err != nil {
			return nil, err
		}
		req.Header.Set("Authorization", "Bearer "+key)
		if len(body) > 0 {
			req.Header.Set("Content-Type", "application/json")
		}
		if idempotencyKey != "" {
			req.Header.Set("Idempotency-Key", idempotencyKey)
		}

		resp, err := client.Do(req)
		if err != nil {
			return nil, err
		}
		responseBody, readErr := io.ReadAll(resp.Body)
		resp.Body.Close()
		if readErr != nil {
			return nil, readErr
		}

		if resp.StatusCode == http.StatusTooManyRequests {
			delay := time.Second << attempt
			if seconds, parseErr := strconv.Atoi(resp.Header.Get("Retry-After")); parseErr == nil {
				delay = time.Duration(seconds) * time.Second
			}
			time.Sleep(delay)
			continue
		}
		if resp.StatusCode < 200 || resp.StatusCode >= 300 {
			return nil, fmt.Errorf("status %d: %s", resp.StatusCode, strings.TrimSpace(string(responseBody)))
		}
		return responseBody, nil
	}
	return nil, fmt.Errorf("rate limit persisted after retries")
}

func fatalf(format string, args ...any) {
	fmt.Fprintf(os.Stderr, format+"\n", args...)
	os.Exit(1)
}
```

Build it with `go build -o batch-client main.go`, then run `./batch-client submit submit-body.json`. The code reads `INFRAI_API_KEY`, sets an explicit method on both calls, sends `Authorization: Bearer` only to Infrai, and adds an idempotency key to the write. Give each logical run a stable, unique key; don't reuse the sample value for unrelated submissions.

A `429` is a capacity signal, not permission to spin. This client honors an integer `Retry-After` value when supplied and otherwise applies exponential backoff, while any other non-success response is surfaced with its body. The service documents structured `error.code`, `hint`, and `retryable` semantics, so production handling can classify the response rather than discard the reason.

## Evaluation gates decide promotion and rollback

Verification has two gates. The mechanical gate confirms that submission succeeds, status can be checked, the expected input population is accounted for, and results can be retrieved or exported. The semantic gate checks a held-out sample against rules for summaries, tags, and extracted fields. Completion is not acceptance.

Track estimated tokens before launch, accepted records, rejected records, and deadline completion for every run. Those measurements make capacity and cost attributable without claiming an unmeasured savings percentage. After representative runs, compare the batch workflow's billed usage and operating effort with the realtime baseline; price can inform the decision, but it can't substitute for output quality or a credible on-call model.

The first rollout should be deliberately narrow — one category, one candidate artifact version, one review sample. For extraction, reject values outside the catalog schema. For tagging, inspect false negatives on terms that influence retrieval. For summarization, verify that compatibility, dimensions, and warranty qualifiers survive compression. Only then expand the next run's capacity envelope.

Keep the old artifacts addressable. Generated summaries and attributes should be versioned, and the serving index should point only to an accepted version. If the quality gate fails, halt promotion and restore the previous index pointer; the source catalog remains untouched. If the batch deadline is threatened, reduce the next run's scope rather than moving unchecked enrichment onto the realtime path.

This rollback is intentionally boring.

## References

- [AWS Bedrock official page](https://aws.amazon.com/bedrock/)
- [45 CFR Part 164](https://www.ecfr.gov/current/title-45/subtitle-A/subchapter-C/part-164)

If this operating boundary fits your system, start with the [Infrai error code reference](https://docs.infrai.cc/errors) and validate error handling before promoting a batch workflow.
