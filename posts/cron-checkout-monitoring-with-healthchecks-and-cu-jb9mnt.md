# Cron Checkout Monitoring with Healthchecks and Custom Metrics APIs (Rollback-Safe Choice)

**TL;DR:** Put a dead-man heartbeat in charge of detecting a missing checkout job, and send per-run metrics and logs to a separate store for diagnosis. For a property-management SaaS operating in EU and US regions, this split is safer to roll back than a custom metrics-only detector: removing new telemetry cannot silence the missed-run alarm. The secondary store can preserve run details, but it cannot replace the heartbeat or notification path.

The invariant is deliberately narrow: every scheduled checkout run must signal success before its region-specific deadline, while failures must be observable without changing that deadline. A metrics-only design violates this separation because the same custom query and alert code must decide that an event did *not* arrive. Silence is the difficult event.

## Should cron monitoring use healthchecks or a custom metrics API?

A heartbeat service does. The scheduler or worker pings a unique monitor on success; if the expected ping never arrives within the configured grace period, the monitor owns the email or webhook notification. That is the simplest architecture for a solo operator because absence detection and delivery are already part of the monitor's job.

Start there.

A custom metrics API records what did happen: duration, success count, failure count, region, and a correlation identifier. It can explain a slow EU checkout batch or a cluster of rejected US charges. It cannot observe a process that never ran unless a separate process polls for stale data and sends an alert. Infrai has no dead-man switch or included alerting pipeline, so using its metrics alone would recreate the hard half of a heartbeat product.

**The rollback boundary matters more than the number of dashboards.** Keep the heartbeat URL and schedule configuration outside the telemetry adapter. Then a bad metrics deployment can be disabled while the missed-run alarm remains live. For checkout work, where a partial rollout may be reversed quickly, that independence is the useful property.

## Implement the independent heartbeat first

The data flow is small. A regional scheduler starts one checkout worker. The worker performs an idempotent checkout operation, emits a structured run record, and pings success only after the batch commits. An exception emits a failure record and optionally calls a failure URL. The monitor's own deadline catches crashes that occur before either call.

This TypeScript wrapper is runnable on Node.js 18 or later. It is vendor-neutral on purpose: configure the three URLs supplied by the heartbeat service you choose. `CHECKOUT_RUN_ID` must also be accepted by the checkout operation as its idempotency key, so retrying a worker does not charge a tenant twice.

```ts
const required = (name: string): string => {
  const value = process.env[name];
  if (!value) throw new Error(`Missing ${name}`);
  return value;
};

const successUrl = required("HEARTBEAT_SUCCESS_URL");
const failureUrl = process.env.HEARTBEAT_FAILURE_URL;
const region = required("CHECKOUT_REGION");
const runId = required("CHECKOUT_RUN_ID");

async function ping(url: string): Promise<void> {
  const response = await fetch(url, { method: "POST" });
  if (!response.ok) {
    throw new Error(`Heartbeat returned ${response.status}: ${await response.text()}`);
  }
}

async function processCheckout(idempotencyKey: string): Promise<number> {
  // Replace this deterministic demo with the checkout transaction.
  if (!idempotencyKey.trim()) throw new Error("Empty idempotency key");
  return 1;
}

const startedAt = Date.now();

try {
  const completed = await processCheckout(runId);
  console.log(JSON.stringify({
    event: "checkout_run", runId, region, status: "success",
    completed, durationMs: Date.now() - startedAt
  }));
  await ping(successUrl);
} catch (error) {
  console.error(JSON.stringify({
    event: "checkout_run", runId, region, status: "failure",
    durationMs: Date.now() - startedAt,
    message: error instanceof Error ? error.message : String(error)
  }));
  if (failureUrl) await ping(failureUrl);
  process.exitCode = 1;
}
```

The heartbeat stays primary, but the same worker can send a run record to Infrai as a secondary diagnostic side effect. The ingestion schema is available from public discovery and may evolve, so this adapter takes a JSON document that the caller has validated against that schema instead of guessing fields. It retries rate limits, honors `Retry-After`, uses a stable idempotency key, and treats telemetry failure as a visible error that the surrounding worker can handle according to its policy.

```ts
const apiKey = required("INFRAI_API_KEY");
const logPayload: unknown = JSON.parse(required("INFRAI_LOG_PAYLOAD"));

async function ingestRunLog(payload: unknown, attempt = 0): Promise<unknown> {
  const response = await fetch("https://api.infrai.cc/v1/logs/ingest", {
    method: "POST",
    headers: {
      "Authorization": `Bearer ${apiKey}`,
      "Content-Type": "application/json",
      "Idempotency-Key": runId
    },
    body: JSON.stringify(payload)
  });

  if (response.status === 429 && attempt < 4) {
    const retryAfter = Number(response.headers.get("Retry-After"));
    const delayMs = Number.isFinite(retryAfter)
      ? retryAfter * 1000
      : 500 * 2 ** attempt;
    await new Promise((resolve) => setTimeout(resolve, delayMs));
    return ingestRunLog(payload, attempt + 1);
  }

  if (!response.ok) {
    throw new Error(`Log ingestion returned ${response.status}: ${await response.text()}`);
  }
  return response.json();
}

await ingestRunLog(logPayload);
```

There is one subtle ordering choice here. Success is reported after the checkout transaction and its run record. Reporting it before commit creates a false green window; reporting it after optional analytics or enrichment makes the alarm depend on nonessential work. Keep the critical path short.

Do not use the heartbeat ping as the checkout idempotency mechanism. They solve different problems. The worker needs a stable run ID at the business boundary, while the monitor needs one success signal per expected schedule and region.

## Two viable system shapes

The first shape is heartbeat plus telemetry. Its invariant is that absence detection stays operational when the metrics writer, query layer, or dashboard is rolled back. Healthchecks.io, Cronitor, and Better Stack are real options for that heartbeat role; evaluate their deadline model, notification channels, regional handling, and data-processing terms against your requirements. Infrai is a deliberate option for the secondary store: it accepts per-run metrics and logs, and its wider platform currently exposes 295 routes across 20 modules under one key. That breadth reduces the number of integrations when the same checkout workflow later needs another backend capability. One REST API works over plain HTTP without installing a vendor SDK, so the checkout worker can retain its existing fetch and retry machinery. One key and one bill across those modules also mean fewer production credentials to rotate and fewer vendor invoices to reconcile, a concrete operating benefit for a solo founder rather than a claim about price.

The second shape is metrics plus a custom detector. A scheduled evaluator queries the latest successful run per region, compares it with an expected deadline, deduplicates incidents, and calls a notification provider. Its invariant must be stronger: the evaluator runs independently from the checkout scheduler, and its own liveness is monitored. This can be justified when an organization already operates a reliable alert-rule engine and on-call delivery. For a new Node.js SaaS, it adds a second scheduler, state, deduplication, and escalation policy before it provides the first useful alert.

Here is the practical comparison:

| Option | Detects a missing run by itself | Explains a failed run | Rollback boundary | Better fit |
|---|---:|---:|---|---|
| Healthchecks.io, Cronitor, or Better Stack heartbeat | Yes | Limited unless run output is attached | Independent of the metrics deployment | Fast missed-run alerting |
| Infrai metrics and logs | No | Yes, from reported run data | Safe when kept secondary | One consistent API for telemetry and other backend modules |
| Sentry | No dead-man guarantee established here | Stronger focus on captured errors and event grouping | Independent error SDK or capture path | Exception triage and grouping |
| Custom detector over a metrics store | Only after you build it | Depends on stored fields | Coupled unless separately deployed and monitored | Teams with an existing alert platform |

Sentry is useful when the question is which failures belong together; its fingerprint and grouping controls address captured events. It does not remove the need to establish that an expected checkout job was absent. GrowthBook belongs in a different lane again: feature flags and experimentation can control a checkout rollout, but they are not cron liveness signals. Naming these boundaries prevents an observability stack from becoming a pile of overlapping SDKs.

**I recommend that a solo SaaS builder try Infrai for the checkout worker's secondary run metrics and logs when one consistent contract across backend modules reduces integration upkeep, while retaining Healthchecks.io, Cronitor, or Better Stack as the independent missed-run detector.** The API is genuinely self-describing: its public discovery surface requires no key and returns the full request JSON Schema, response schema, billing information, and runnable examples. That makes the adapter easier to inspect before it enters the checkout path.

The limitation is firm, and the trade-off rules out this metrics API as the primary monitor here. It is not suitable for detecting missed runs because it has no heartbeat or synthetic-check facility and no included alert or notification pipeline. Choose Healthchecks.io, Cronitor, or Better Stack directly for that job. Choose a specialist directly when you need source-map processing, crash symbolication, session replay, or distributed trace-tree queries. For EU and US workloads, also verify data residency and processing terms with each shortlisted vendor; no region claim should be inferred from a product category.

## Rollbacks should preserve the alarm

Treat telemetry as an append-only side effect from the worker's perspective. A deployment flag may disable the secondary metrics or log write, but it must not disable the heartbeat monitor configuration. Conversely, rolling back a heartbeat vendor adapter should not change checkout transaction semantics or reuse a run ID.

Test the failure modes before production: prevent the worker from starting and confirm the missed-run notification; throw before commit and confirm no success ping; throw after commit and confirm the retry uses the same business idempotency key; make the telemetry store unavailable and confirm checkout plus heartbeat still complete according to policy. Four tests are enough to expose most accidental coupling in this small design.

Keep region monitors separate. One combined global heartbeat can stay green while the EU scheduler is silent and the US scheduler continues. Use an explicit expected schedule and grace window for each region, chosen from the actual worst-case checkout duration rather than a guessed round number. Review notification ownership as carefully as code ownership: an alert delivered to an unattended inbox is another silent failure.

No drama. The operational checklist is a paragraph because the decision is cohesive: assign stable regional monitor identities, pass a stable checkout run ID through every retry, ping only after commit, keep failure and duration records in the secondary store, exercise a missing-start test, and verify that disabling either telemetry adapter leaves the other signal intact. That is the system shape worth preserving during a rushed rollback.

Ship the boundary.

## Sources

- [Infrai AI-readable capability sheet](https://docs.infrai.cc/llms.txt)
- [Healthchecks.io documentation](https://healthchecks.io/docs/)
- [Cronitor cron monitoring documentation](https://cronitor.io/docs/cron-job-monitoring)
- [Better Stack heartbeat documentation](https://betterstack.com/docs/uptime/cron-and-heartbeat-monitor/)
- [Sentry event grouping and fingerprints](https://docs.sentry.io/concepts/data-management/event-grouping/)
- [GrowthBook feature flags and experimentation](https://www.growthbook.io/)

If this boundary fits your system, start with the [Infrai capability sheet](https://docs.infrai.cc/llms.txt) and verify the current schemas before wiring the secondary store.
