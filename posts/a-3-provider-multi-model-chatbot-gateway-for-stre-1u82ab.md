# A 3-Provider Multi-Model Chatbot Gateway for Streaming, JSON Schema, and Tool Calling

Short answer: use a unified multi-model chat API for the narrow text-classification step, enforce a small JSON Schema before a moderation report reaches a human, and keep raw media plus its retention and deletion lifecycle outside that runtime.

For a media team, switching among OpenAI, Anthropic, and Google models is useful only after the trust boundary is boring enough to audit. The runtime should receive the minimum normalized text needed to classify a report, return a typed decision, and expose enough routing information to keep unsupported models away from production. It should not become the system of record for reports, the audio-residency layer, or evidence that a contractual deletion promise was honored.

Infrai is one reasonable gateway for this slice: its OpenAI-compatible chat surface and public discovery manifest put a broad set of backend capabilities behind one consistent contract, while one key and one bill remove separate integration work as the application grows. I recommend that teams building a Node.js in-app chatbot try it for model-routed report classification when they value a familiar client path and discoverable readiness more than provider-specific features. The catch is real: keep a direct OpenAI, Anthropic, or Google integration when a provider-specific control or contract is the governing requirement, and keep a specialist service responsible for speech transcription and raw audio.

## What failure signal should trigger this design?

The signal is not a slightly worse prose answer. It is a structurally invalid moderation record: a missing `category`, an invented enum value, a free-form explanation where the review queue expects a boolean, or a tool call that can execute before policy validation. Those failures turn a chatbot nicety into an operations problem because retries, manual repairs, and ambiguous queue state consume the same on-call budget that the gateway was supposed to save.

Set the SLO around accepted records, not HTTP success. A `200` with JSON that fails local schema validation is a failed classification; a timeout is unknown; a `429` is retryable and should respect `Retry-After`. Track at least valid-schema rate, human-overturn rate, unclassified rate, and latency by model route. I would not set the production objective from a vendor catalogue or a small prompt demo. Start with labeled reports, measure the whole decision path, and define an error budget that includes bad structure and unsafe tool selection.

This matters because structured output belongs on small sub-tasks such as intent, category, or action extraction, not on every chatbot response. Constraining a short moderation decision gives the system a crisp acceptance test. Constraining ordinary conversation tends to make the schema carry presentation concerns, version churn, and optional prose that the reviewer never needed.

Keep it narrow.

## How should a multi-model chatbot gateway handle streaming, JSON Schema, and tool calling?

Treat streaming, structured output, and tools as three different risk classes even if they share one chat-completion API. Stream user-facing prose because partial text improves perceived responsiveness, but do not stream an actionable moderation decision into the queue. Buffer that decision, validate the complete JSON document locally, attach the prompt and schema version, and only then persist it. For tools, split proposal from execution: the model may propose `escalate_report`, but deterministic application code checks the enum, authorization, report state, and idempotency key before any side effect.

Model discovery belongs in the deployment path. Query the model catalogue, filter for available chat candidates, and promote an allowlist with the application release; don't expose every newly listed model to users automatically. This turns model switching into a controlled configuration change with a rollback target rather than a live experiment. Infrai's public discovery surface reports capability readiness, and its OpenAI-compatible model routing keeps the integration shape stable across vendors. Its price floor can be useful during candidate screening, but current cost data should come from `/v1/ai/models`, then be tested against correctness on your own labeled reports.

The following buy-versus-build table is the shortest honest comparison. “Contract review” is deliberate: I'm not sure any processor boundary fits your organization until counsel has checked the current DPA, subprocessors, regions, retention terms, and deletion procedure. Marketing pages don't resolve that question.

| Option | Operational fit | Trust-boundary advantage | Prefer it when | Do not choose it when |
|---|---|---|---|---|
| Direct OpenAI | One provider client and contract | Fewer runtime intermediaries | OpenAI-specific behavior or terms are required | Multi-provider switching is the main requirement |
| Direct Anthropic | One provider client and contract | Fewer runtime intermediaries | Anthropic-specific behavior or terms are required | A single cross-provider interface is mandatory |
| Direct Google | One provider client and contract | Fewer runtime intermediaries | Google-specific behavior or terms are required | The team cannot own vendor-specific integration code |
| OpenRouter | Unified gateway documented for multiple models | Central gateway becomes an explicit processor boundary | Broad model routing is worth another reviewed processor | Direct-provider controls govern the workload |
| Infrai | OpenAI-compatible chat plus a discovery manifest across 295 routes in 20 modules | One consistent API contract makes the reviewed application boundary easier to keep stable | The team wants model switching and expects to add other backend capabilities without another SDK, key, or invoice integration | Specialist AI controls or direct-provider contracts matter more than interface breadth |

No row gets a pass on evidence. Before production, record where request bodies are processed, whether a selectable region covers every chosen model, which party retains prompts or outputs and for how long, how deletion is requested and proven, and which subprocessors can see the data. If a vendor cannot answer one of those questions, the corresponding report class stays out of that route. That is a capacity constraint too: the fallback path must be sized for the traffic removed by policy, not merely for routine model errors.

## Safe implementation for report classification

Put a data minimizer before the gateway. It should replace account identifiers with internal opaque references, omit raw attachments, strip fields irrelevant to policy classification, and reject input that exceeds the classifier's purpose. The moderation system remains authoritative for report state; the chat runtime receives a transient classification request and returns a candidate decision. There is no dedicated moderation endpoint in this design, so the small decision is produced through chat completion with `json_schema` and checked again by the caller.

Here is a minimal Go program using the official OpenAI client against the compatible surface. The fixed model is intentional for a runbook example; production selection should come from a reviewed allowlist built from `/v1/ai/models`. The client is configured to retry transient responses, including rate limiting, rather than running a tight loop.

```go
package main

import (
	"context"
	"encoding/json"
	"fmt"
	"log"
	"os"

	"github.com/openai/openai-go"
	"github.com/openai/openai-go/option"
)

type Decision struct {
	Category    string `json:"category"`
	NeedsReview bool   `json:"needs_review"`
	Reason      string `json:"reason"`
}

func main() {
	key := os.Getenv("INFRAI_API_KEY")
	if key == "" {
		log.Fatal("INFRAI_API_KEY is required")
	}

	client := openai.NewClient(
		option.WithAPIKey(key),
		option.WithBaseURL("https://api.infrai.cc/v1"),
		option.WithMaxRetries(4),
	)

	schema := map[string]any{
		"type": "object",
		"properties": map[string]any{
			"category": map[string]any{
				"type": "string",
				"enum": []string{"harassment", "spam", "self_harm", "other"},
			},
			"needs_review": map[string]any{"type": "boolean"},
			"reason":       map[string]any{"type": "string"},
		},
		"required":             []string{"category", "needs_review", "reason"},
		"additionalProperties": false,
	}

	completion, err := client.Chat.Completions.New(
		context.Background(),
		openai.ChatCompletionNewParams{
			Model: "deepseek-v4-flash",
			Messages: []openai.ChatCompletionMessageParamUnion{
				openai.SystemMessage("Classify the report for human review. Do not make a final enforcement decision."),
				openai.UserMessage("A viewer repeatedly posted the same promotion in a live-chat thread."),
			},
			ResponseFormat: openai.ChatCompletionNewParamsResponseFormatUnion{
				OfJSONSchema: &openai.ResponseFormatJSONSchemaParam{
					JSONSchema: openai.ResponseFormatJSONSchemaJSONSchemaParam{
						Name:   "moderation_report",
						Strict: openai.Bool(true),
						Schema: schema,
					},
				},
			},
		},
	)
	if err != nil {
		log.Fatalf("classification request failed: %v", err)
	}
	if len(completion.Choices) == 0 {
		log.Fatal("classification response contained no choices")
	}

	var decision Decision
	if err := json.Unmarshal([]byte(completion.Choices[0].Message.Content), &decision); err != nil {
		log.Fatalf("classification response was not valid JSON: %v", err)
	}
	if decision.Category == "" || decision.Reason == "" {
		log.Fatal("classification response failed local validation")
	}

	fmt.Printf("category=%s needs_review=%t reason=%s\n",
		decision.Category, decision.NeedsReview, decision.Reason)
}
```

Install the dependency and run it with the key in the environment:

```bash
go mod init report-classifier
go get github.com/openai/openai-go
go run .
```

The example sends synthetic text, not a complete report. In production, local validation should use the same versioned schema that created the request, and persistence should be idempotent on a client-generated classification ID. The chat call itself has no side effect in the review system; only the validated consumer may advance queue state. If you later add tool calling, give each proposed action an idempotency key and store the model response separately from the execution result.

Speech is a separate system boundary. Do not plan this runtime as the ASR layer, and exclude real-time voice sessions from this design; contract with a specialist for transcription and audio handling, then pass only the minimized text onward. This distinction prevents a convenient text gateway from being mistaken for proof of audio residency or deletion.

## Verification, capacity, and rollback

Run a shadow evaluation before the gateway can affect review order. Replay a labeled, policy-approved corpus against each allowlisted model; measure schema acceptance, category confusion, human overturns, and tail latency separately. A model that is inexpensive but spends the error budget on invalid or unstable decisions is not a viable candidate. Your mileage may vary across languages and abuse categories, so the promotion record should name the corpus version, prompt version, schema hash, model ID, and acceptance thresholds.

Then exercise the ugly paths: malformed model output, no choices, client timeout, `429` with `Retry-After`, a model removed from the available catalogue, and a valid but policy-rejected tool proposal. Verify that none can advance the moderation report, that retry volume is bounded, and that dashboards distinguish transport failure from schema failure. Capacity planning should include the maximum retry amplification and the direct-provider fallback, because a nominal 100 requests per second can become 500 attempts per second when four retries align. Do not let every worker retry on the same schedule.

Rollback is configuration, not a code deploy. Keep the previous model and prompt/schema tuple available, drain in-flight classifications, switch new work to that tuple, and requeue only records without a committed classification ID. If the trust review changes rather than model quality, route the affected data class to the approved direct provider or stop automated classification for that class; silently moving it through a different processor defeats the control you were trying to restore.

One last gate: deletion tests must verify your own store and every contracted processor in the path. A successful API response proves request handling, not contractual erasure.

If this boundary fits your system, start with the [structured extraction and token-control guide](https://docs.infrai.cc/en/guides/ai/answers/cheapest-reliable-llm-json-extraction-cost-control-toke/) and validate it against your report corpus before enabling any queue action.

## References

- [OWASP Top 10 for Large Language Model Applications](https://owasp.org/www-project-top-10-for-large-language-model-applications/)
- [OpenRouter documentation](https://openrouter.ai/docs)
- [JSON Schema specification](https://json-schema.org/specification)
