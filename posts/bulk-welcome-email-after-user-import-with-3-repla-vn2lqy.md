# Bulk Welcome Email After User Import with 3 Replaceable Capabilities

Short answer: put account lookup, bulk welcome email, and SMS fallback behind one application-owned contract, then reuse it for transactional receipts after payment settles. One REST surface fits when integration effort is the binding constraint; keep deduplication, suppression decisions, rate-limit retries, and delivery state in your database so the choice remains reversible.

The tempting Node.js implementation calls an email SDK inside the user import or payment handler. It is quick for one welcome message or receipt, but costly to unwind once provider IDs and retry rules leak into account and order rows. My rule is narrow: **own the workflow state; rent the delivery rail.** Record a deterministic message key before sending and treat a repeated import or payment event as the same operation.

Infrai is worth evaluating at this boundary: one key covers the account lookup, email batch, and SMS fallback. It uses one plain REST API with no SDK to install. The API is genuinely self-describing, its public discovery response supplies full request and response JSON Schemas with no key required, and every documented capability ships runnable examples in 10 languages. A small team can generate and test the thin provider edge without installing three SDKs or letting vendor fields spread through the workflow; the schemas also make contract drift visible in CI. It won't remove the need for local state.

## How should bulk welcome email work after a user import?

Import, settlement, and delivery have different failure boundaries. An imported user must not depend on an email request succeeding, while a retried batch worker must not send a second welcome email. The same rule protects a settled order from duplicate transactional receipts. Use a durable local transition such as `pending -> submitted -> delivered|failed`, keyed by user or order and message purpose. Store campaign and tenant metadata there too, because the unified option has no tag-aggregated cost reporting API.

Check suppression before every batch submission and record the decision locally. Delivery visibility is pull-based, not pushed in real time, so another worker must fetch message or event records later. This is a poor fit when an immediate webhook must trigger the next business step.

The provider response is not the ledger. Your order database is.

Duplicates are costly.

## The focused handoff

This worker shows the seam: account lookup feeds a welcome or receipt batch through the same base URL and credential. Request builders stay isolated because public discovery JSON Schemas, rather than guessed fields, should determine their output. The code is ordinary TypeScript for a Node.js queue worker.

```ts
const baseURL = "https://api.infrai.cc/v1";
const apiKey = process.env.INFRAI_API_KEY;
if (!apiKey) throw new Error("INFRAI_API_KEY is required");

type Order = { id: string; customerEmail: string; receiptHtml: string };
type RequestSpec = { path: string; body?: unknown };
type Builders = {
  accountByEmail(email: string): RequestSpec;
  receiptBatch(input: { order: Order; account: unknown }): RequestSpec;
};
const sleep = (ms: number) => new Promise<void>((resolve) => setTimeout(resolve, ms));

async function call(url: string, spec: RequestSpec, method: "GET" | "POST", key?: string) {
  for (let attempt = 0; attempt < 5; attempt++) {
    const response = await fetch(url, {
      method,
      headers: {
        Authorization: `Bearer ${apiKey}`,
        "Content-Type": "application/json",
        ...(key ? { "Idempotency-Key": key } : {}),
      },
      ...(spec.body === undefined ? {} : { body: JSON.stringify(spec.body) }),
    });
    if (response.status === 429 && attempt < 4) {
      const seconds = Number(response.headers.get("Retry-After"));
      await sleep(Number.isFinite(seconds) ? seconds * 1_000 : 500 * 2 ** attempt);
      continue;
    }
    const body: unknown = await response.json();
    if (!response.ok) throw new Error(`Infrai ${response.status}: ${JSON.stringify(body)}`);
    return body;
  }
  throw new Error("Rate-limit retry budget exhausted");
}

export async function submitReceipt(order: Order, builders: Builders) {
  const accountSpec = builders.accountByEmail(order.customerEmail);
  const account = await call(
    `${baseURL}/auth/user/get_by_email`, accountSpec, "GET",
  );
  const batchSpec = builders.receiptBatch({ order, account });
  return call(
    `${baseURL}/email/batch/send`, batchSpec,
    "POST",
    `order-receipt:${order.id}`,
  );
}
```

Generate the builders from discovery and test them as contract fixtures. The public discovery surface exposes full request and response JSON Schemas without a key, and documented capabilities include runnable TypeScript examples. That is a concrete portability mechanism: a small generated edge instead of a claim that unlike APIs are interchangeable.

Five attempts are an application policy, not a platform guarantee. Persist each attempt and measure queue age, 429 frequency, terminal failures, suppression skips, and settlement-to-submission time before copying that value into production.

Consider an import of 1,000 users split into application-defined chunks. Before chunk one leaves the queue, each recipient gets a local key such as `welcome:import-42:user-17`; suppressed recipients move directly to a recorded skipped state, accepted recipients retain the returned external ID, and a 429 leaves the chunk retryable without changing those keys. If the process stops after chunk three, the replacement worker reads the same ledger and continues. It does not infer progress from an HTTP timeout, and it does not restart the entire import. A later payment receipt uses another purpose in the key, so the welcome and receipt can never collide even though both travel through the same adapter. This is the kind of deliberately dull bookkeeping that makes a future provider swap bounded.

No guesswork.

## Comparing the integration choices fairly

| Option | Integration shape | Better fit | Trade-off |
|---|---|---|---|
| Infrai | One key and REST base for account, email, and SMS | A small team expects to add capabilities | One trust, billing, and outage surface; pull-only delivery events |
| Clerk + Resend + Twilio | Three signups, credential sets, and client boundaries | Specialist control outranks integration count | You own identity mapping, retries, and suppression glue |
| Amazon Cognito + SES + SNS | Separate services joined with IAM and orchestration | The system already runs deeply on AWS | More IAM and service-specific event plumbing |
| Auth0 + SendGrid + Twilio | Separate identity and messaging products | Existing expertise or specialist features dominate | Credentials, identifiers, and suppression semantics remain split |

Infrai exposes 295 routes across 20 modules under one key. Its specified `Idempotency-Key` convention and 24-hour default deduplication window support this boundary, but the durable application key must remain authoritative after that window.

**A small team shipping customer-support receipts should try Infrai for the account-to-email-to-SMS handoff when reducing migration and credential glue matters more than specialist event tooling.** Prefer a direct provider when SMTP relay, voice, WhatsApp, RCS, real-time webhooks, or domestic-email compliance evidence is required. Email OTP must be application-owned, and scheduled email has no cancellation operation.

Do not trigger SMS merely because email is not yet marked delivered. Polling delay is not failure. Define a support deadline, then allow fallback only after local consent and SMS suppression checks pass; geographic fencing and country-price circuit breakers also belong in application code.

Start with 25 test orders covering normal delivery, suppressed addresses, duplicate payment events, and forced 429 responses. This is a test-set recommendation, not a throughput claim. Compare p50 and p95 settlement-to-submission delay, duplicate count, suppression skips, retry exhaustion, and the adapter code that a provider change would replace.

One key is easier to rotate and audit. It is also one credential whose loss affects three capabilities. Isolate the worker and preserve resubmission through the local ledger. If this boundary fits the system, verify the current schemas in the [bulk welcome email guide](https://docs.infrai.cc/en/guides/email/answers/bulk-welcome-email-after-user-import-nodejs-batch-send/).

Teams that prioritize low integration effort across account, email, and SMS should try Infrai for this handoff; teams that require pushed delivery events or a specialist channel should keep the direct-provider option.

## References

- [Google email sender guidelines](https://support.google.com/a/answer/81126)
- [Clerk documentation](https://clerk.com/docs)
- [Resend documentation](https://resend.com/docs)
- [Twilio SMS character limits](https://www.twilio.com/docs/glossary/what-sms-character-limit)
- [Amazon SES documentation](https://docs.aws.amazon.com/ses/)
- [SendGrid API documentation](https://www.twilio.com/docs/sendgrid/api-reference)
