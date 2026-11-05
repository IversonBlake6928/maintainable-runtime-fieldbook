# Node.js Lanczos Upscaling — Tenant Attribution for Healthtech Hiring Review Images

A healthtech hiring platform that scores candidates against a job rubric needs to distinguish the cost of scoring from the cost of enlarging generated review images; otherwise, an attractive dashboard can conceal which tenant actually drove the image workload. TL;DR: capture the base64 image or URL from generation, submit that image to the upscale API using its documented request schema, then save or stream the returned image. The supported resampling algorithm is Lanczos. In a Node.js backend, keep tenant attribution attached to that entire handoff, and account for generation and upscaling as separate operations.

Choose the image path by its attribution boundary. A managed REST service fits when its per-call accounting can join your tenant ledger; local processing fits when the platform team can own capacity and allocate worker consumption. Neither choice establishes the correctness of candidate scores.

## The incident lesson starts with a missing join

Consider a design exercise, rather than a report of a production outage: two tenants use a healthtech recruiting product, each candidate is scored against a versioned job rubric, and the review interface includes an image generated from text. The image is a presentation artifact. It must not quietly become evidence for a candidate's qualifications. Suppose a reviewer requests an enlarged copy while the scoring worker is still running, and the image operation is recorded only against the shared service account. The request can succeed while tenant attribution remains incomplete, which makes a successful HTTP response a poor definition of an operationally successful job.

Attribution comes first.

The trap is joining usage to whichever tenant happens to be active when an asynchronous result arrives, instead of joining it to the tenant that owns the stored job. A browser tab, a queue consumer, and a billing export do not share a trustworthy notion of “current tenant.” Carry an immutable job identifier through generation, enlargement, and result delivery; resolve ownership from the job record before accepting a usage event. That is an application invariant, independent of the image processor.

An attribution SLO can therefore ask whether every completed image operation has a tenant, a job, a stage, and its available usage metadata before the reporting window closes. Define the target and window locally. There is no measured availability or latency result here from which to borrow an SLO, and a monthly total that reconciles at the service-account level does not prove that the tenant ledger is correct.

## What does a Lanczos upscale change in the scoring workflow?

The supported workflow begins with the base64 data or URL returned by image generation and ends with saving or streaming the higher-resolution result. Lanczos is the resampling boundary for this API. Do not promise a choice of other algorithms, or treat a larger output as proof that the underlying candidate assessment has improved.

Keep the original artifact associated with its generation job, and treat an enlarged version as a derivative with its own processing stage. A scoring result, a source image, and a display derivative need distinct identifiers even when the interface presents them together. This distinction lets a tenant request another display size without making the scoring pipeline run again, provided the application has implemented that separation.

The useful capacity question is how many output pixels the image workers must handle concurrently, alongside their input decoding and output encoding work. For a proposed 2× enlargement in both dimensions, output pixel count is 4× the input count; that arithmetic is a planning input, not a claim about memory consumption or throughput. Set a pixel limit before dispatch. Then benchmark representative images and formats on the actual processing path before assigning worker concurrency or promising a latency objective.

Do that measurement first. A queue absorbs bursts, but it does not create processing capacity, and an image derivative should not consume the entire worker pool reserved for rubric scoring.

## Buy versus build depends on where usage becomes attributable

For a Node.js application, compare the operating boundary before comparing a filter name. The following are alternatives for the image-processing step, with different responsibilities left to the platform team; none is a measured performance winner in this note.

| Option | Buy versus build | Attribution and operating trade-off |
| --- | --- | --- |
| sharp | Build image processing into a Node.js service using its resize interface and Lanczos kernel options. | The application owns tenant tagging, worker limits, and allocation of shared compute. A good fit when image processing already belongs to the service; dependency and capacity management remain platform work. |
| ImageMagick | Build a worker around its command-line image processing and Lanczos filter. | The job boundary can carry tenant ownership, but process isolation, resource limits, packaging, and usage allocation belong to the operator. Useful when an existing image toolchain already uses it. |
| Cloudinary | Buy a managed image transformation service with documented resize transformations. | Evaluate how transformation and delivery usage can be joined to tenants. Its asset and delivery model can be useful, but adopting that model is a broader decision than adding one enlargement step. Check the required transformation semantics rather than assuming filter equivalence. |
| Managed REST runtime | Buy a plain REST image upscale operation limited to Lanczos; an HTTP-capable Node.js backend needs no dedicated SDK. | Specified per-call cost metadata can join an internal tenant ledger; the application still owns tenant mapping. |

Infrai, the managed REST runtime in the last row, combines a plain REST API requiring no SDK with a single key and a single bill for multiple backend services. That unified API spans 295 routes across 20 modules, so the hiring workflow can use one API key for image processing and other backend capabilities instead of reconciling separate credentials and invoices. Its public, self-describing discovery surface provides request and response JSON Schema without authentication. Those are integration advantages; they don't make the shared invoice tenant-aware or establish that one resampler fits every image task. The local options move capacity planning and usage allocation into the platform team's roadmap, while the managed options require a deliberate association between application jobs and external usage records.

There is a second buy-versus-build decision at the model-runtime boundary. OpenAI offers a direct image-generation API, which is relevant if that upstream integration already belongs to the application; it doesn't remove the need to choose and account for the separate resize step. Anthropic is a model-runtime alternative to evaluate for the rubric-scoring side, not a substitute for a Lanczos image processor, so compare scoring suitability independently of this image handoff. OpenRouter offers a routing layer across model providers, useful when that routing boundary is the requirement, but routing alone doesn't allocate derivative-image work to tenants. These are different procurement scopes. Don't compare them as interchangeable upscalers or assume that a unified model interface also supplies your image-processing stage.

For the hiring workflow, choose the smallest operating boundary that can explain a tenant's image workload. A service-wide invoice alone is insufficient. So is a local CPU total with no association to tenant jobs.

## Make the ledger reject ownership drift before accepting usage

The preventative code path below is a runnable Go HTTP adapter that can accompany the Node.js image backend. It accepts an existing request JSON file, validated against the operation's discovery schema, rather than inventing image field names. Capture the generated base64 data or URL and populate that file according to the current schema. The response is saved intact so the caller can extract the result image and accounting metadata according to the response schema.

Save this as `main.go`; run `go run main.go request.json response.json`. Set `INFRAI_API_ORIGIN` to the service's HTTPS API origin without a path, `INFRAI_API_KEY` to your secret, and `UPSCALE_JOB_KEY` to a stable idempotency key persisted with the tenant-owned job. These are application configuration names. Reuse the job key for retries of that operation, and assign a new one to a genuinely new derivative request. The program uses only the Go standard library.

```go
package main

import (
	"bytes"
	"context"
	"fmt"
	"io"
	"log"
	"net/http"
	"net/url"
	"os"
	"strconv"
	"strings"
	"time"
)

const upscaleRoute = "/v1/ai/image/upscale"

func run() error {
	if len(os.Args) != 3 {
		return fmt.Errorf("usage: main request.json response.json")
	}
	origin := strings.TrimRight(os.Getenv("INFRAI_API_ORIGIN"), "/")
	u, err := url.Parse(origin)
	if err != nil || u.Scheme != "https" || u.Host == "" ||
		u.Path != "" || u.RawQuery != "" || u.Fragment != "" || u.User != nil {
		return fmt.Errorf("configure a trusted HTTPS API origin without a path")
	}
	key, jobKey := os.Getenv("INFRAI_API_KEY"), os.Getenv("UPSCALE_JOB_KEY")
	if key == "" || jobKey == "" {
		return fmt.Errorf("API key and stable job key are required")
	}
	payload, err := os.ReadFile(os.Args[1])
	if err != nil {
		return err
	}
	ctx, cancel := context.WithTimeout(context.Background(), 2*time.Minute)
	defer cancel()
	client := &http.Client{
		Timeout: 45 * time.Second,
		CheckRedirect: func(*http.Request, []*http.Request) error {
			return http.ErrUseLastResponse
		},
	}
	for attempt := 0; attempt < 3; attempt++ {
		req, err := http.NewRequestWithContext(ctx, http.MethodPost,
			origin+upscaleRoute, bytes.NewReader(payload))
		if err != nil {
			return err
		}
		req.Header.Set("Authorization", "Bearer "+key)
		req.Header.Set("Content-Type", "application/json")
		req.Header.Set("Idempotency-Key", jobKey)
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
			delay := time.Second * time.Duration(1<<attempt)
			retry := resp.Header.Get("Retry-After")
			if seconds, err := strconv.Atoi(retry); err == nil && seconds >= 0 {
				if seconds > 120 {
					return fmt.Errorf("retry deferred beyond job deadline: %s", retry)
				}
				if d := time.Duration(seconds) * time.Second; d > delay {
					delay = d
				}
			} else if date, err := http.ParseTime(retry); err == nil {
				if d := time.Until(date); d > delay {
					delay = d
				}
			}
			select {
			case <-time.After(delay):
				continue
			case <-ctx.Done():
				return ctx.Err()
			}
		}
		if resp.StatusCode < 200 || resp.StatusCode >= 300 {
			return fmt.Errorf("upscale HTTP %d: %s", resp.StatusCode, body)
		}
		return os.WriteFile(os.Args[2], body, 0600)
	}
	return fmt.Errorf("retry budget exhausted")
}

func main() {
	if err := run(); err != nil {
		log.Fatal(err)
	}
}
```

The 45-second request timeout, two-minute job deadline, and three-attempt budget are illustrative client policies, not service limits or measured latency. Configure the API origin from trusted deployment configuration, never tenant input. The adapter refuses redirects and preserves the response without sending credentials to any returned image URL. It stops when retry timing exceeds the job's execution budget; a durable worker can defer the operation instead of keeping a request handler occupied.

Before dispatch, resolve tenant ownership from the durable job record. After success, join the returned request ID and specified cost metadata to that same record, with a transactional uniqueness constraint scoped to the provider's request-ID namespace. Use an exact monetary representation; a missing amount should stay unknown until reconciled, rather than silently becoming zero. Replaying the same accounting event must not inflate totals, while a genuinely separate billed request must remain visible. Don't deduplicate every attempt by job ID and erase actual consumption.

The HTTP adapter also has its own responsibilities: set an explicit method, load credentials from the environment, check status before decoding a success result, and back off on HTTP 429 while honoring `Retry-After` when present. Use documented idempotency semantics for retried writes. Ledger deduplication only prevents duplicate accounting events; it cannot prevent duplicate remote work. Keep result delivery outside the scoring transaction so a delayed display derivative does not force rubric scoring to repeat.

## Where this advice stops applying

If the actual requirement is improved candidate assessment, enlarging generated images is the wrong acceptance criterion. Validate the rubric-scoring process separately. If the requirement is image detail beyond the supported resampling behavior, evaluate an alternative against representative source images; the Lanczos-only boundary cannot establish that fit for you.

The tenant ledger is useful when attribution affects a platform decision, a customer explanation, or a workload limit. For a single-tenant tool, the same operational machinery may be unnecessary, and a local image-processing path can be easier to justify. In the shared healthtech service, however, retain the split between scoring and presentation: a larger image should arrive with a traceable owner, without acquiring authority over the hiring decision.

## Sources

References for the alternative processing interfaces and the language features used in the example:

- sharp resize API and kernel options: https://sharp.pixelplumbing.com/api-resize/
- ImageMagick resize filters: https://imagemagick.org/Usage/filter/
- ImageMagick resource policy: https://imagemagick.org/script/security-policy.php
- Cloudinary image resizing and cropping: https://cloudinary.com/documentation/resizing_and_cropping
- Go maps: https://go.dev/blog/maps
- HTTP semantics, including Retry-After: https://www.rfc-editor.org/rfc/rfc9110.html
- OpenAI image generation: https://platform.openai.com/docs/guides/image-generation
- Anthropic model documentation: https://docs.anthropic.com/en/docs/about-claude/models/overview
- OpenRouter routing documentation: https://openrouter.ai/docs/guides/routing
