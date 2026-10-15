# 2026 Healthtech Webhook Authentication: Raw Body Signature Verification Before Decoding

For a healthtech account platform, webhook signature verification must use the raw body before JSON parsing, including before an Express JSON middleware consumes the stream, because a metered invoice must be traceable to the exact bytes the sender authenticated. **TL;DR: capture the request body once, verify its signature over those unchanged bytes with a narrowly scoped secret, reject failures, and only then decode JSON.** Parser output is application data; it is not signature evidence.

A bounded production scenario makes the distinction concrete. A usage event can be valid JSON and still fail authentication after middleware has decoded and reconstructed it, because whitespace, key order, or byte representation may no longer match the signed message. The invariant is stricter than "same fields": the verifier and sender must use the same byte sequence and the agreed signing procedure. The preventative control belongs at ingress, before a general JSON parser can consume the stream.

## How should webhook signature verification use the raw body before parsing?

JSON describes values, while a message authentication code is calculated over bytes. Two documents can decode to the same object yet differ byte for byte. Re-encoding can alter insignificant whitespace or object-member order; decoding before verification therefore destroys the representation needed to prove what arrived.

Byte identity matters.

This failure is particularly awkward in per-customer usage metering. The business record might say tenant `acct_7f2` consumed 37 units, but the audit question is narrower: which authenticated request authorized that increment? Store the verification outcome, a non-secret event identifier, the tenant binding, the algorithm identifier, the secret version, and a digest of the received bytes. Do not treat a parsed payload copied into a log as equivalent evidence, and do not log signing secrets or raw health-related content merely because debugging is difficult. OWASP recommends lifecycle management, least privilege, rotation, auditing, and avoiding secrets in logs; those controls apply to webhook keys as much as to database credentials.

Short-lived raw bytes are operational state. Cap the accepted body size before allocation, keep the buffer scoped to the handler, and release it after verification and decoding. A request that exceeds the documented limit should fail closed before expensive cryptographic or parsing work.

The 1 MiB cap in the example is a capacity decision, not a universal constant. The trade-off is deliberate: a smaller bound reduces memory exposure and abusive hash work, while a larger one accommodates bulk event contracts at higher per-request cost. Measure legitimate payload sizes, document the selected ceiling, and reject oversized requests consistently at every ingress layer; otherwise a proxy and an application can disagree about which request is admissible, leaving the on-call engineer to diagnose behavior that changes with the traffic path.

## Put the trust decision before the parser

The safe path has a small state machine: bounded read, signature extraction, key lookup, MAC calculation, constant-time comparison, replay-policy check, JSON decoding, schema validation, and idempotent metering. Nothing that changes invoice state runs before the trust decision.

The following Go handler shows that ordering without tying the design to a commercial service. The exact header grammar and signed-message construction must match the sender's documented protocol; this example defines its own compact contract rather than pretending webhook formats are interchangeable.

```go
package webhook

import (
    "bytes"
    "crypto/hmac"
    "crypto/sha256"
    "encoding/hex"
    "encoding/json"
    "io"
    "net/http"
)

const maxBodyBytes = 1 << 20 // 1 MiB, chosen for this endpoint's event contract.

type UsageEvent struct {
    EventID string `json:"event_id"`
    Account string `json:"account_id"`
    Units   int64  `json:"units"`
}

type Handler struct {
    SigningKey []byte
    ApplyOnce  func(UsageEvent, [32]byte) error
}

func (h Handler) ServeHTTP(w http.ResponseWriter, r *http.Request) {
    body, err := io.ReadAll(http.MaxBytesReader(w, r.Body, maxBodyBytes))
    if err != nil {
        http.Error(w, "invalid body", http.StatusBadRequest)
        return
    }

    supplied, err := hex.DecodeString(r.Header.Get("X-Usage-Signature"))
    if err != nil {
        http.Error(w, "invalid signature", http.StatusUnauthorized)
        return
    }

    mac := hmac.New(sha256.New, h.SigningKey)
    _, _ = mac.Write(body)
    if !hmac.Equal(supplied, mac.Sum(nil)) {
        http.Error(w, "invalid signature", http.StatusUnauthorized)
        return
    }

    var event UsageEvent
    dec := json.NewDecoder(bytes.NewReader(body))
    dec.DisallowUnknownFields()
    if err := dec.Decode(&event); err != nil {
        http.Error(w, "invalid event", http.StatusBadRequest)
        return
    }
    if event.EventID == "" || event.Account == "" || event.Units < 0 {
        http.Error(w, "invalid event", http.StatusBadRequest)
        return
    }

    digest := sha256.Sum256(body)
    if err := h.ApplyOnce(event, digest); err != nil {
        http.Error(w, "temporary failure", http.StatusServiceUnavailable)
        return
    }
    w.WriteHeader(http.StatusNoContent)
}
```

`ApplyOnce` is a local persistence contract: it must treat an already committed event identifier as success. The important part is that decoding receives the already verified buffer and that the persistence layer applies `event_id` once. In an Express service, the equivalent boundary is route-scoped raw-body capture placed before JSON middleware for that route. The verifier consumes that captured buffer; mounting a global parser first defeats the ordering even if later middleware asks for raw input.

Do not copy the sample header or signing construction into an integration whose sender specifies a timestamp, version prefix, multiple signatures, or another canonical input. Verify the protocol you contracted for. Cryptographic correctness does not rescue a mismatched message format.

## Auditability changes the data model

An authenticated event is not yet a billable fact. Bind the verified external account identifier to one internal customer before incrementing a meter, validate units against the event schema, and commit the event receipt and usage mutation atomically. A unique constraint on the sender's event identifier turns retry handling into a database invariant instead of a timing bet.

The audit record should answer four questions without exposing the payload: what event was considered, which customer mapping was used, which key version verified it, and whether the usage mutation committed. A SHA-256 digest of the original bytes helps correlate a disputed delivery with retained evidence, but a digest does not prove authenticity by itself; the verified-MAC result and controlled key history supply that part of the argument. Access to those records should be narrower than access to routine application logs, because metadata can still reveal customer activity.

Set an SLO around accepted-event processing, not around returning any 2xx response. Track signature rejection separately from schema rejection, duplicate delivery, account-mapping failure, and persistence failure. Sudden signature failures can mean a sender-side change, a stale key version, or hostile traffic; combining them with malformed JSON erases the signal the on-call engineer needs. Capacity planning should include the maximum bounded request rate, hash work per byte, database contention on idempotency keys, and the retry amplification expected during downstream trouble.

## Which ownership model survives an audit?

The buy-versus-build question is mostly about control evidence and on-call ownership. It is not settled by the shortest setup guide.

| Model | Auditability of access | Operational burden | Lock-in boundary |
|---|---|---|---|
| Application-owned ingress | Align request, key-version, and mutation records directly | Own parser ordering, rotation, replay defense, limits, and paging | Protocol and evidence schema remain portable |
| Shared internal gateway | Central policy and access logs can be consistent | Platform team owns availability and route-specific signing rules | Internal gateway contract becomes a dependency |
| Managed relay | Provider records must satisfy retention and access policy | External operation may reduce local toil; incident ownership remains | Export formats, delivery semantics, and evidence access define switching cost |

Choose by testing evidence retrieval, not by reading feature lists. Ask an operator to trace one synthetic event from received digest through customer mapping to the committed meter change, rotate a test key without losing overlap visibility, and show who accessed the evidence. Reject any design whose answer depends on unrestricted payload logging.

There are conditions where route-level verification is the wrong layer. If a trusted gateway terminates and verifies every webhook under an explicit protocol contract, downstream services may accept a gateway-authenticated envelope instead of the sender's original signature. The gateway then owns raw-byte preservation, replay controls, key custody, and auditable identity propagation. Moving the boundary is valid; leaving it ambiguous is not.

This route-level approach is unsuitable when the application cannot hold signing keys, when policy requires verification in a separately administered security boundary, or when many services need one centrally audited control. In those cases, use a gateway with explicit evidence export and authenticated identity propagation. The limitation moves; it does not disappear.

## Test the byte boundary, then test failure

A happy-path fixture proves very little. Keep a captured, non-sensitive fixture with a known signature and mutate one property at a time: insert whitespace, reorder object members, alter one byte, truncate the body, provide malformed hexadecimal, exceed the size limit, reuse an event identifier, and present a validly signed event for the wrong account mapping. The first four cases demonstrate why parsed-object verification is unsafe; the later cases exercise authorization and accounting invariants.

Deployment deserves the same skepticism. Roll key versions with an explicit overlap window defined by the sender's protocol, identify verification records by non-secret key version, and alert on unexpected use of a retired version. Keep secrets out of source, command-line arguments, traces, and error responses. Audit secret reads and administrative changes, then rehearse revocation so the first real compromise is not the first time the procedure runs.

One metric is especially misleading: webhook success rate. A handler that acknowledges before durable, idempotent persistence can look healthy while dropping billable usage. Measure the state transition that matters.

Trust the commit.

## The durable rule

**Authenticate the received bytes before interpreting them, then make the authenticated event idempotent and auditable before it changes a customer's meter.** That ordering turns a middleware detail into an enforceable security boundary. It also keeps incident review honest: operators can distinguish untrusted input, invalid business data, duplicate delivery, and a failed invoice mutation without reaching for sensitive payload logs.

## Sources

- https://cheatsheetseries.owasp.org/cheatsheets/Secrets_Management_Cheat_Sheet.html
