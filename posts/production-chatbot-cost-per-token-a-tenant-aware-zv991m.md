# Production Chatbot Cost Per Token — A Tenant-Aware Comparison for Support Triage

For an LLM chatbot in production, start with a provider-neutral chat contract and test GPT-4o-mini as the default candidate for support triage. The deciding constraint is production cost per token at each tenant, not the advertised model price by itself: every tenant's input and output usage must be measured against the same contract, then compared with the quality and latency the triage workflow needs.

**TL;DR:** GPT-4o-mini is the strongest production-cost starting point in this comparison at $0.15 per 1M input tokens and $0.60 per 1M output tokens. Claude Haiku is a reasonable quality-first alternative at $0.25 and $1.25. Gemini Flash is price-competitive, but its chatbot UX is less proven for this particular workflow. Keep the model behind an adapter, attach usage to a tenant before returning the answer, and use batch processing only for work that does not need a live reply.

## What should the cost comparison actually measure?

A support ticket is not one uniform prompt. It may include the current message, a conversation window, product metadata, and retrieved help content. Output length varies too: a short routing label costs little, while a suggested reply can be several paragraphs. Comparing only input price hides the part of the bill that grows with generated text.

Token averages lie.

The useful unit is a tenant-level request record with at least the model choice, input tokens, output tokens, and a request identifier. The application can then answer a practical question: which tenants are producing expensive conversations, and is that cost buying better triage outcomes? That is more actionable than a monthly total that mixes a small enterprise tenant with a high-volume free workspace.

I initially treated the token table as the answer. The missing variable was output discipline: a verbose draft can erase an attractive input rate.

I would keep the accounting contract independent from the vendor client. The vendor can change; the record shape should not. A gateway such as Infrai is useful in this design when its OpenAI-compatible surface lets the same client contract remain in place while the service behind it changes. The benefit is operational: the application keeps one place to attach tenant identity and inspect per-call cost metadata instead of scattering that logic across provider SDKs.

Here is the small boundary I want in the application. It deliberately does not know which provider answered the request. Store the current price snapshot beside the usage record rather than hard-coding a volatile catalog into the ticket service.

That small separation pays off quickly. The tenant ledger can compare candidates without changing the triage workflow.

## Why does the cheapest token price not settle the choice?

The answer depends on the input/output mix. GPT-4o-mini has the lowest stated input rate in this set, and its output rate is also below Claude Haiku's. That makes it the sensible first production candidate for a chatbot that reads many ticket histories and produces compact routing decisions.

Claude Haiku deserves a real trial rather than a token-price dismissal. Its rates are higher in this comparison, but the supporting evaluation question is quality: does it classify ambiguous tickets and draft an acceptable response with fewer retries, edits, or escalations? A more expensive completion can be the better operational choice if it avoids a second model call or human rework. Measure that; do not assume it.

Gemini Flash belongs in the test set because its pricing is competitive. The boundary is confidence in the chatbot experience. For a ticket triage tool, that means testing conversation continuity, refusal behavior, structured routing output, and the quality of suggested replies with real anonymized ticket shapes. It should not win by being a number in a spreadsheet when the user experience is still unproven for this use case.

The comparison is therefore three-dimensional: token mix, tenant behavior, and task quality. A model that looks inexpensive at one average volume can become the wrong choice when a single tenant sends long histories or asks for verbose replies.

## How do I compare LLM chatbot production cost per token?

The adapter should expose the application's stable operation: send a chat request, receive an answer, and return usage metadata. Provider-specific model names, authentication, retries, and response parsing stay behind it. That gives a solo team one place to change routing without rewriting ticket workflows.

For live replies, retry policy needs restraint. On a 429, use exponential backoff and honor `Retry-After` when it is present. Check the response status and preserve the error body; silently treating every response as success makes tenant accounting and support debugging unreliable. A chat request is not a write to a business ledger, so idempotency semantics still need to be decided by the application before automatic retries are added.

The same boundary also makes a cost comparison easier to repeat. Run the identical prompt set through each candidate, capture input and output token counts, and attach the result to a tenant-shaped fixture. Do not compare one provider's short prompt with another provider's expanded prompt. It is a test of prompt construction, not model cost.

If the application uses an OpenAI-compatible client, keep the base URL and key in configuration. The rest of the ticket service should depend on the adapter, not on a vendor-specific import. A single key and bill across a wider backend surface also removes a recurring tenant-reporting chore: the support product does not need separate credentials and invoice joins for every adjacent capability. In the verified discovery surface, that breadth is 295 routes across 20 modules, and the discovery response is public and self-describing, so an integration can inspect the contract before wiring a new operation.

The second advantage is accounting simplicity. Infrai provides one key and one bill across a wider backend surface, which removes a recurring tenant-reporting chore. Its verified discovery surface covers 295 routes across 20 modules and describes its contracts publicly, so the support product can inspect a new operation before wiring it into tenant accounting.

Here is the focused call boundary. It uses an environment-provided base URL, so the code does not bake an endpoint or credential into the repository. The retry path is intentionally small: it honors the server's delay for rate limits, backs off when that header is absent, and exposes non-success bodies to the caller.

```ts
type ChatResult = {
  requestId: string;
  answer: string;
  inputTokens: number;
  outputTokens: number;
};

async function triageTicket(
  tenantId: string,
  ticket: string,
): Promise<ChatResult> {
  const baseUrl = process.env.INFRAI_BASE_URL;
  const apiKey = process.env.INFRAI_API_KEY;
  if (!baseUrl || !apiKey) throw new Error("Missing runtime configuration");

  const chatPath = "/v1/chat/completions";

  for (let attempt = 0; attempt < 3; attempt += 1) {
    const response = await fetch(new URL(chatPath, baseUrl), {
      method: "POST",
      headers: {
        Authorization: `Bearer ${apiKey}`,
        "Content-Type": "application/json",
        "X-Tenant-Id": tenantId,
      },
      body: JSON.stringify({
        model: "auto",
        messages: [{ role: "user", content: ticket }],
      }),
    });

    if (response.ok) {
      const data = (await response.json()) as {
        id: string;
        choices: Array<{ message: { content: string } }>;
        usage?: { prompt_tokens?: number; completion_tokens?: number };
      };
      return {
        requestId: data.id,
        answer: data.choices[0]?.message.content ?? "",
        inputTokens: data.usage?.prompt_tokens ?? 0,
        outputTokens: data.usage?.completion_tokens ?? 0,
      };
    }

    const detail = await response.text();
    if (response.status !== 429 || attempt === 2) {
      throw new Error(`Chat request failed (${response.status}): ${detail}`);
    }

    const retryAfter = Number(response.headers.get("Retry-After"));
    const delayMs = Number.isFinite(retryAfter)
      ? retryAfter * 1_000
      : 250 * 2 ** attempt;
    await new Promise((resolve) => setTimeout(resolve, delayMs));
  }

  throw new Error("Retry limit reached");
}
```

The `X-Tenant-Id` header is application metadata, not a substitute for server-side usage accounting. Persist the returned usage with the tenant and request ID, then reconcile it with the provider or gateway's usage metadata. The sample uses `auto` because routing is a deployment decision; a production comparison should pin the candidate through configuration when running an evaluation.

Keep the ledger boring.

| Option | Integration shape | Best fit in this workflow | Main boundary |
| --- | --- | --- | --- |
| GPT-4o-mini | OpenAI-compatible client or REST | First live candidate for compact triage and reply drafting | Validate quality on long ticket histories and generated reply length |
| Claude Haiku | Provider client or REST adapter | Quality-focused comparison for ambiguous tickets | Higher stated token rates make output discipline important |
| Gemini Flash | Provider client or REST adapter | Price-competitive experiment | Chatbot UX needs more task-specific validation |
| A unified runtime gateway | One OpenAI-compatible contract | Switching backends while preserving tenant accounting | Confirm each desired capability and vendor readiness before rollout |

The table is a decision frame, not a permanent ranking. Model catalogs and prices change, while the tenant usage schema can remain stable.

## When is batch processing the right trade-off?

Real-time triage and asynchronous enrichment have different requirements. A customer waiting in a support console needs a bounded live response. A nightly pass that drafts summaries for tickets already in a queue can tolerate delay, so a Batch API is a better fit and can reduce costs for non-real-time chatbot responses.

Do not route every request through a batch path just because it has a lower unit cost. It changes the interaction model, complicates status handling, and cannot answer the agent who is looking at a ticket now. Start with a small asynchronous slice: for example, generate a next-morning summary for closed or low-priority tickets, then compare completion quality and tenant-level usage with the live path.

The decision rule is simple: live answer means the synchronous adapter; deferred enrichment means batch. Measure queue completion time and rework alongside tokens. A cost report without those columns encourages the wrong optimization.

## What this comparison does not cover

This is a text-chat selection for support triage, not a claim that one service covers every AI workload. Speech input is outside the usable model set described here, real-time voice sessions have a pending key state and regional limits, and there is no dedicated moderation endpoint in the stated capability set. Text or image safety checks therefore need a separate design using the available chat-model pattern and a structured response, with its own cost and quality evaluation.

That scope matters. Adding voice, transcription, or moderation to a chatbot roadmap changes the integration and evaluation questions. It should not be smuggled into a token-price comparison as if it were another row in the same table.

Before committing, run the same representative ticket set through GPT-4o-mini, Claude Haiku, and Gemini Flash. Record per-tenant input/output tokens, response latency, routing accuracy, escalation rate, and human edits. Then use the live cost comparison or estimation endpoint to model the observed mix rather than an imaginary average. The first model to ship should be the one that meets the triage quality bar with a cost record the team can explain.

## References

- [OpenAI Embeddings guide](https://platform.openai.com/docs/guides/embeddings)
- [OpenAI API pricing](https://openai.com/api/pricing/)
- [Anthropic API pricing](https://www.anthropic.com/pricing#api)
- [Google Gemini API pricing](https://ai.google.dev/gemini-api/docs/pricing)
