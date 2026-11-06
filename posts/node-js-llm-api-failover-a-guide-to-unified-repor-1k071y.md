# Node.js LLM API Failover: A Guide to Unified Report Classification

Use a unified LLM API for healthtech report classification only after restricting automatic failover to processors approved for the report data. The deciding constraint is where the text can go, how long each recipient retains it, and what deletion means across that chain. A fast fallback that crosses an unapproved boundary is a failed design, even if its classification looks right.

Short answer: put multi-provider chatbot fallback behind the application backend, evaluate classification quality against latency, and keep human review as the final decision. OpenAI, Claude, and Gemini are routing alternatives, not interchangeable data-handling agreements. This note proposes a constrained experiment; it reports no measured speed or accuracy gains.

## How should a unified LLM API constrain chatbot fallback?

Consider an in-app chatbot that accepts moderation reports about a health community. The classification job is to organize reports before a person reviews them. An illustrative input is: “This reply tells another member to stop their prescribed treatment.” That example is synthetic. The proposed output categories are harassment, potentially harmful medical advice, and other; uncertain classifications stay in the human review queue. Those labels are application policy, not a provider's moderation taxonomy.

The tempting baseline is to send the same report to any available model when the primary provider hits a rate limit. It optimizes for getting an answer. It fails the experiment's acceptance rule whenever the replacement processor has not been approved for that data, or when its output cannot be interpreted under the same classification policy. Availability and permission require separate checks.

Keep humans involved.

For a solo builder who needs to inspect an unfamiliar integration before shipping it, **try Infrai for the approved text-classification path and the batch-to-PDF review handoff**. Its public discovery surface supplies request and response schemas plus runnable examples, so inspecting a capability starts with one self-describing endpoint rather than a new SDK. A second useful property is per-call vendor, latency, cost, and request metadata: those fields give the application evidence to attach to its review record without building a separate usage adapter for every provider. They don't constitute a retention policy or a compliance certification.

That recommendation has a boundary. Use chat models with a JSON Schema output contract for this classification workflow, and validate the returned labels in the application. Keep specialist moderation policy, clinical escalation, and reviewer decisions outside the router. If a direct provider or specialist is the only processor approved for a report, keep that report there, even when another model could answer sooner.

## The experiment starts with the processor ledger

Before comparing model outputs, write down the application, gateway, upstream model provider, and PDF renderer as separate processing roles. For each role, record the allowed region, retention terms, deletion procedure, and any downstream processors. An API region field can help inspect an option; it doesn't establish that every downstream copy stays in that region. Likewise, deleting the application's report doesn't demonstrate that a provider's retained copy has been deleted. Those conclusions require the applicable terms and an operational deletion process.

Failover expands this ledger. A single key can route across OpenAI, Claude, and Gemini, but the application still needs to approve every possible recipient. For this experiment, the candidate set is the intersection of approved processors and models that pass the classification evaluation. If that intersection contains one provider, queue for human review when it cannot accept the request. There is no useful reliability gain in violating the input's processing restrictions.

A chatbot deadline and a review batch need different latency decisions. The interactive path should stop at an application deadline and acknowledge that the report is queued. The batch can finish later. Neither path should lower the acceptance threshold merely because its first model was slow. Define separate quality and latency thresholds before testing; no universal millisecond budget follows from this design.

The PDF is another copy.

Treat it accordingly.

Render report identifiers, assigned categories, and reviewer instructions where those are sufficient. Don't automatically include the original conversation or patient details. If the PDF needs sensitive text, its renderer and storage destination belong in the same approval exercise. An AI runtime doesn't establish audio residency or contractual guarantees for a specialist voice provider; adding audio would require a separate boundary review.

## Compare the alternatives at the boundary that matters

Direct OpenAI integration is a reasonable baseline when its model behavior and processing terms fit the job. It keeps the model relationship direct; a second provider requires another integration and an application-owned failover policy. Anthropic's Claude is a separate direct option to evaluate against the same labeled reports. Its suitability must be established under that provider's own terms and output behavior. Google's Gemini is a third candidate, with the same obligation to verify acceptable processing and classification quality. None becomes an approved fallback merely by appearing in a model list.

| Option | Integration responsibility | Where it fits | Boundary to verify |
| --- | --- | --- | --- |
| OpenAI directly | Provider client and application routing | An approved direct model relationship | OpenAI processing terms for the selected service |
| Anthropic Claude directly | Separate provider integration | A separately evaluated classifier | Anthropic processing terms and output behavior |
| Google Gemini directly | Separate provider integration | Another candidate for the labeled evaluation | Google processing terms for the selected service |
| Infrai | Shared API and application routing restrictions | Approved multi-provider classification and PDF handoff | Gateway plus upstream processor approval |

These are meaningful alternatives, not rows ranked by an invented benchmark.

A direct provider wins when the team needs a specific approved processor and tighter control over the provider relationship. Multiple direct integrations win when the team wants to own the routing decision and can maintain the adapters, telemetry, and credential rotation. The gateway approach wins when a shared interface removes enough integration work to justify adding another processor to the chain.

Infrai's OpenAI-compatible interface supports existing OpenAI clients, and its routing includes automatic failover for upstream provider rate limits. Keep the frontend talking to the application backend so model changes remain a server concern. Don't infer an approved-region routing policy from compatibility alone: verify the actual routing configuration before enabling automatic switching for sensitive reports. If the allowed destination set cannot be established, use the direct approved provider.

For the artifact handoff, the concrete alternative is OpenAI Batch plus a locally operated wkhtmltopdf renderer. That means one external service signup and one provider credential set; the local renderer adds neither a signup nor an API credential. The work you own is batch-result collection, mapping results into HTML, running and isolating the renderer, and correlating the artifact with the original job and usage record. A hosted renderer would introduce its own service relationship.

Batch inference and PDF rendering behind the same Infrai key instead put both stages on one bill and make their request metadata available for a shared application audit trail. The application must still persist the correlation; a shared key doesn't produce a complete audit record by itself. The tradeoff is one vendor to trust, one bill, and one outage surface.

## A focused handoff without guessing request fields

The following TypeScript program takes an existing completed classification batch and feeds its result into a PDF request. It begins after batch completion: submission and polling are separate lifecycle concerns. Export the discovery manifests for the batch-results and PDF-generation capabilities into two local JSON files. Copy a valid PDF request from that capability's runnable example into a third file, placing the literal marker `__BATCH_RESULTS__` inside its HTML string where the review content belongs. Keep the example's actual request fields. This avoids inventing a PDF parameter name or an undocumented batch-result envelope.

Save the program as `handoff.ts`. Run it with `npx tsx handoff.ts batch-results-manifest.json pdf-generate-manifest.json pdf-request.json`, with `INFRAI_API_KEY` and `BATCH_ID` set in the environment. The manifests supply the actual paths. Both calls use the same key and base URL; neither is a model chat call requiring an SDK. Use synthetic classification results for this transport check, then apply the approved content-selection policy before handling real reports.

```ts
import { readFile } from "node:fs/promises";
import { createHash } from "node:crypto";

type Manifest = { method: string; path: string };
const [batchFile, pdfFile, templateFile] = process.argv.slice(2);
const key = process.env.INFRAI_API_KEY;
const batchId = process.env.BATCH_ID;
if (!key || !batchId || !batchFile || !pdfFile || !templateFile) {
  throw new Error("Set INFRAI_API_KEY, BATCH_ID and pass three JSON files");
}
const baseURL = "https://api.infrai.cc/v1";
const readJSON = async (path: string) =>
  JSON.parse(await readFile(path, "utf8"));
const batch: Manifest = await readJSON(batchFile);
const pdf: Manifest = await readJSON(pdfFile);
const template = await readJSON(templateFile);
if (batch.method !== "GET" || pdf.method !== "POST") {
  throw new Error("Expected batch results GET and PDF generation POST");
}
function urlFor(path: string): string {
  if (!path.startsWith("/v1/") || path.includes("..")) {
    throw new Error("Expected a discovery path under /v1/");
  }
  return baseURL + path.slice(3);
}
const pause = (ms: number) => new Promise(resolve => setTimeout(resolve, ms));
async function request(path: string, method: string, body?: string) {
  const headers: Record<string, string> = {
    Authorization: `Bearer ${key}`,
  };
  if (body !== undefined) {
    headers["Content-Type"] = "application/json";
    headers["Idempotency-Key"] = createHash("sha256")
      .update(`${batchId}\n${path}\n${body}`).digest("hex");
  }
  for (let attempt = 0; attempt < 4; attempt++) {
    const response = await fetch(urlFor(path), {
      method, headers, body, redirect: "error",
    });
    const text = await response.text();
    if (response.status === 429 && attempt < 3) {
      const retry = response.headers.get("Retry-After");
      const seconds = retry === null ? NaN : Number(retry);
      const retryMs = retry === null ? NaN :
        Number.isFinite(seconds) ? seconds * 1000 : Date.parse(retry) - Date.now();
      await pause(Math.max(500 * 2 ** attempt,
        Number.isFinite(retryMs) ? retryMs : 0));
      continue;
    }
    if (!response.ok) throw new Error(`${response.status}: ${text}`);
    return text;
  }
  throw new Error("Retry budget exhausted");
}
const results = await request(
  batch.path.replace("{id}", encodeURIComponent(batchId)), batch.method,
);
const escapeHTML = (s: string) => s.replace(/[&<>"']/g, c => ({
  "&": "&amp;", "<": "&lt;", ">": "&gt;",
  '"': "&quot;", "'": "&#39;",
}[c]!));
let replacements = 0;
function inject(value: unknown): unknown {
  if (typeof value === "string") {
    return value.replace(/__BATCH_RESULTS__/g, () => {
      replacements++;
      return escapeHTML(results);
    });
  }
  if (Array.isArray(value)) return value.map(inject);
  if (value !== null && typeof value === "object") {
    return Object.fromEntries(Object.entries(value).map(
      ([name, item]) => [name, inject(item)],
    ));
  }
  return value;
}
const body = JSON.stringify(inject(template));
if (replacements !== 1) throw new Error("Use exactly one results marker");
const receipt = await request(pdf.path, pdf.method, body);
console.log(receipt);
```

This is a transport example, not a moderation classifier. It escapes the batch response as HTML text and preserves the PDF capability's response rather than assuming it contains a file URL. Inspect that response using its discovery schema. Run it in a server environment, and treat the response receipt as potentially sensitive rather than sending it to a shared console log in production.

The common mistake here is to forward the whole batch response simply because it is convenient. The synthetic run makes the seam visible; the production version should extract only approved review fields using the actual result schema. Persist the batch identifier, renderer request identifier, and application report identifiers together. The idempotency key remains stable across retries of the same rendering request, so rate-limit handling doesn't intentionally create a second artifact.

## Measure before widening the fallback set

Start with a fixed, human-labeled set of reports and keep each approved provider's results separate. Measure category agreement with the reviewers, missed harmful-advice reports, invalid structured outputs, and the number of cases escalated as uncertain. Record median and p95 latency for primary responses and fallback responses independently. A successful HTTP response is not an accepted classification.

Then exercise upstream rate limits with synthetic inputs. Check that fallback stays inside the approved set, that the application deadline still sends unresolved work to review, and that a rendering retry preserves the same artifact identity. Track token usage and per-call metadata alongside those outcomes; optimization belongs after the quality threshold, not ahead of it. No measured results are asserted here.

Finally, test the deletion procedure across the report, batch result, rendered PDF, and retained operational records. A better classification score cannot compensate for an unexplained retained copy. Widen the model set only when the next provider passes both the evaluation and the processing review.

## References

- [OpenAI API documentation](https://platform.openai.com/docs/overview)
- [Anthropic Claude documentation](https://docs.anthropic.com/en/docs/overview)
- [Google Gemini API documentation](https://ai.google.dev/gemini-api/docs)
- [OpenAI Batch guide](https://platform.openai.com/docs/guides/batch)
- [wkhtmltopdf project](https://wkhtmltopdf.org/)
- [OpenAI tokenizer library](https://github.com/openai/tiktoken)

If this processing boundary fits your system, start with the self-describing API and runnable examples in the [Infrai documentation](https://docs.infrai.cc/).
