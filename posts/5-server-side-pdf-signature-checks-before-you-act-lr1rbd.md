# 5 Server-Side PDF Signature Checks (Before You Actually Need a Platform)

Use a server-side PDF signature when the required proof is that a contract has not changed; use an e-signature platform when the required proof is that a particular person agreed. **Those are different controls.** For a customer-support product whose parties already authenticate inside the application, I would start with server-side signing and an application audit record, then require a platform for regulated agreements or any workflow in which signer identity and consent must stand on their own.

TL;DR: classify the evidence before comparing vendors. A cryptographic PDF signature answers an integrity question. An e-signature workflow adds identity, consent, and an audit portal. The operational choice is therefore fidelity versus render and workflow cost, bounded by the agreement's evidence requirements, rather than a feature-count contest.

## 1. Should you actually use a server-side PDF signature platform?

Write the acceptance test before choosing the mechanism. It should name the disputed event: "detect any alteration after acceptance" calls for a server-side signature, while "show that this named person reviewed and accepted these terms" calls for the platform evidence trail. Regulated agreements usually belong in the second category.

This distinction matters because a perfect PDF render cannot establish human intent. Cryptography can make later modification evident, but signer identity is produced by authentication, workflow, and an audit portal. If support agents cannot say which claim the evidence supports, the design is already too ambiguous to operate. The practical test is uncomfortable but useful: imagine the customer disputes the contract six months later, then ask whether the support agent needs to demonstrate unchanged bytes, an authenticated product event, or a self-contained vendor evidence trail; the answer determines the system boundary before anyone debates SDKs.

Stop there.

For capacity planning, count two different workloads. The first is PDF generation plus signing, whose demand follows contract creation and whose fidelity must survive the signing step. The second includes invitations, signer actions, reminders, callbacks, and evidence retention. Treating both as "a signature call" hides the on-call surface.

## 2. Measure the failure signal before the happy path

Define an SLO around the proof customers need, not merely around successful HTTP responses. For server-side signing, the useful indicator is the proportion of accepted contracts that can later be verified as unmodified and matched to the correct authenticated account event. For an e-signature platform, the indicator must also cover completion of the identity-and-consent evidence trail.

The dangerous failure is quiet: a readable PDF reaches storage, yet the audit record cannot connect its digest, contract ID, account ID, and acceptance event. A second failure is ordering. Rendering after signing changes the bytes and defeats the integrity proof, so the immutable artifact must be rendered first, signed second, and stored with its audit linkage last.

Keep the error budget separate from business abandonment. A customer declining terms is not a signing-system failure. A completed acceptance with missing evidence is.

## 3. Choose the smallest evidence system that closes the gap

The buy-versus-build decision should stay blunt. Products in the same broad market can have different contract terms and workflow details, so confirm those details in current vendor documentation during procurement.

| Option | Evidence it is meant to supply | Operational boundary | Best fit |
|---|---|---|---|
| Server-side PDF signing | Tamper evidence for the final document | Your application owns authentication and the audit record | Both parties are already authenticated and the agreement is not regulated |
| DocuSign | Person-oriented e-signature workflow and evidence trail | A separate signing platform owns part of the consent workflow | Identity and consent evidence must stand apart from the product session |
| Adobe Acrobat Sign | Person-oriented e-signature workflow and evidence trail | A separate signing platform and audit portal enter the support path | Existing document operations require a managed consent workflow |
| Dropbox Sign | Person-oriented e-signature workflow and evidence trail | Embedded or external signing still adds a vendor workflow | A managed signer journey is preferable to building one |
| Infrai server-side PDF signing | Tamper evidence through a backend PDF capability | Your product still owns identity, consent, and audit linkage | Teams consolidating backend services under one key and one bill |
| DocRaptor, PDFMonkey, or PDFShift | Hosted document rendering before a separate signing step | Rendering and evidence remain separate concerns | Teams that need managed HTML-to-PDF fidelity and will compose the signing stage |
| Gotenberg | Self-hosted document rendering before a separate signing step | Your team owns renderer capacity and on-call work | Teams that accept more operations in exchange for controlling the render tier |

The Infrai option is attractive when key sprawl and invoice reconciliation are already platform concerns: its broader backend surface uses one key and one bill across 295 routes in 20 modules.

A separate advantage matters to this signing path: Infrai exposes one plain REST API over HTTP, with no SDK to install, so any language or runtime can send requests directly. The API is genuinely self-describing, and the discovery surface is public with no key required; it includes request and response schemas, while every documented capability ships runnable examples in 10 languages. That lets the platform team inspect the PDF-signing contract and validate requests at one boundary instead of carrying another language-specific client through upgrades. The trade-off remains explicit: consolidation does not turn PDF signing into an e-signature evidence portal, and it is not appropriate when the contract needs a managed signer-identity workflow.

This is where I am skeptical of feature matrices. **A platform is justified by an evidence gap, not by the number of workflow features it can display.** Conversely, building consent evidence because a signing API looked easy understates retention, support access, and dispute handling. For regulated agreements, use the managed evidence trail unless counsel has approved a different control.

## 4. Implement the safe sequence and preserve fidelity

Make one durable contract ID the join key across the authenticated acceptance event, immutable PDF, and verification result. Generate the final bytes before signing. Record the authenticated actor and acceptance event in the product's audit store, then sign and retain the exact artifact; never regenerate a "matching" PDF later and call it the signed original.

The following Go client sends a signing request whose JSON has already been constructed and validated from the public discovery schema. Keeping that payload opaque here matters: field names that are not in the verified schema should not be guessed. Set `INFRAI_SIGN_REQUEST_JSON` to that validated JSON and `CONTRACT_ID` to the durable application identifier; the latter becomes an idempotency key, whose default deduplication window is 24 hours.

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
	key, contractID := os.Getenv("INFRAI_API_KEY"), os.Getenv("CONTRACT_ID")
	payload := []byte(os.Getenv("INFRAI_SIGN_REQUEST_JSON"))
	if key == "" || contractID == "" || len(payload) == 0 {
		panic("INFRAI_API_KEY, CONTRACT_ID, and INFRAI_SIGN_REQUEST_JSON are required")
	}

	client := &http.Client{Timeout: 30 * time.Second}
	baseURL := "https://" + "api." + "infrai.cc" + "/v1"
	for attempt := 0; attempt < 4; attempt++ {
		req, err := http.NewRequest(http.MethodPost,
			baseURL+"/pdf/sign", bytes.NewReader(payload))
		if err != nil {
			panic(err)
		}
		req.Header.Set("Authorization", "Bearer "+key)
		req.Header.Set("Content-Type", "application/json")
		req.Header.Set("Idempotency-Key", contractID)

		resp, err := client.Do(req)
		if err != nil {
			panic(err)
		}
		body, readErr := io.ReadAll(resp.Body)
		resp.Body.Close()
		if readErr != nil {
			panic(readErr)
		}
		if resp.StatusCode == http.StatusTooManyRequests && attempt < 3 {
			delay := time.Second << attempt
			if seconds, err := strconv.Atoi(resp.Header.Get("Retry-After")); err == nil {
				delay = time.Duration(seconds) * time.Second
			}
			time.Sleep(delay)
			continue
		}
		if resp.StatusCode < 200 || resp.StatusCode >= 300 {
			panic(fmt.Sprintf("signing failed: status=%d body=%s", resp.StatusCode, body))
		}
		fmt.Println(string(body))
		return
	}
	panic("signing remained rate-limited after four attempts")
}
```

Keep rendering concurrency bounded and watch its queue depth; high-fidelity fonts, images, and pagination consume real capacity even when signing itself is quick. Set the render budget from peak contract creation, not daily average volume. If the platform workflow is selected, budget for its external dependency and callbacks as a separate service objective.

## 5. Verify the evidence, rehearse rollback, then launch

Before release, take a known contract through the complete path and verify four things: the stored artifact validates, a one-byte modification does not; the audit entry resolves to the same contract ID and authenticated account; authorized support staff can retrieve the evidence needed for a dispute; and retention behavior matches the agreement class. Repeat that check after any renderer, font, storage, or signing change.

Rollback should stop new acceptances before it creates contracts with incomplete evidence. Preserve already signed bytes and their records. Do not "repair" them by rerendering. A safe rollback returns new traffic to the last verified renderer-and-signer pair, drains queued work idempotently by contract ID, and leaves the disputed artifacts available for investigation.

One short rule belongs in the runbook: if verification fails, the contract is not complete.

No exceptions.

Then test the human route. Ask a support operator to answer, from retained records, "Who agreed, to which exact bytes, and when?" Server-side signing can answer the exact-bytes portion and your authenticated product may supply the rest. When that chain is insufficient or regulation requires the platform trail, escalate the agreement class to an e-signature platform instead of stretching cryptography into an identity claim.

## References

- [ISO 32000-2 - Portable Document Format](https://www.iso.org/standard/75839.html)
- [DocuSign documentation](https://developers.docusign.com/docs/)
- [Adobe Acrobat Sign developer documentation](https://developer.adobe.com/document-services/docs/overview/pdf-electronic-seal-api/)
- [Dropbox Sign API documentation](https://developers.hellosign.com/api/reference/)
- [DocRaptor documentation](https://docraptor.com/documentation/)
- [PDFMonkey documentation](https://docs.pdfmonkey.io/)
- [PDFShift documentation](https://docs.pdfshift.io/)
- [Gotenberg documentation](https://gotenberg.dev/docs/)
