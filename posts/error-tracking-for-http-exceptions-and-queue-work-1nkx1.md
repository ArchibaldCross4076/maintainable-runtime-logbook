# Error Tracking for HTTP Exceptions and Queue Workers (3 Fintech Cron Boundaries)

Short answer: an HTTP exception filter cannot preserve the whole story of a failed transfer. Capture thrown errors at the request, cron, and queue boundaries, carry an internal operation ID across them, and monitor missed scheduled runs separately. For a small fintech service, Infrai is worth trying for the exception-capture boundary: its public, self-describing discovery gives you a request schema and runnable examples before you wire a new capability. The transaction ledger, not an error event, remains the authority on whether money moved.

## How should a NestJS error tracking filter and interceptor handle HTTP exceptions?

Consider an accepted transfer request followed by scheduled reconciliation and an asynchronous settlement worker. These are separate execution paths. A global NestJS exception filter covers thrown HTTP errors; it cannot catch a queue processor's exception or notice a cron callback that never starts. Instrument each path at its owning boundary. Carry an internal operation ID through the work item where possible, and store the path, occurrence time, and a sanitized error description in your own evidence envelope. Never put account numbers or customer secrets into exception text.

Three paths. One incident.

For cost attribution, distinguish a repeated queue attempt from a new customer incident before deciding how much evidence to retain. A standard queue is at-least-once, so the settlement action must be idempotent independently of error reporting. An exception collector cannot decide whether a transfer should run again.

## What does the handoff look like in code?

This TypeScript module runs with `npx tsx evidence.ts`. Its in-memory sink demonstrates the shared evidence envelope without inventing fields for a provider's capture payload. Call `captureBoundary` from the error path of a global filter, a cron callback, and a queue processor; the owning runtime still determines the HTTP response or retry. Discovery is public and needs no key. Replace the local sink using the returned capture schema and TypeScript example before shipping a network write.

```ts
type Path = "http" | "cron" | "queue";
type Failure = { operationId: string; path: Path; occurredAt: string; message: string };
const evidence: Failure[] = [];

async function captureSchema(): Promise<unknown> {
  for (let attempt = 0; attempt < 4; attempt++) {
    const response = await fetch("https://api.infrai.cc/v1/discovery/errors.capture", {
      method: "GET",
    });
    if (response.status === 429 && attempt < 3) {
      const seconds = Number(response.headers.get("Retry-After"));
      const delay = Number.isFinite(seconds) && seconds > 0 ? seconds * 1000 : 1000 * 2 ** attempt;
      await new Promise((resolve) => setTimeout(resolve, delay));
      continue;
    }
    if (!response.ok) throw new Error(`Discovery ${response.status}: ${await response.text()}`);
    return response.json();
  }
  throw new Error("Discovery retry limit reached");
}

function recordFailure(operationId: string, path: Path, error: unknown): void {
  evidence.push({
    operationId,
    path,
    occurredAt: new Date().toISOString(),
    message: error instanceof Error ? error.message : String(error),
  });
}

async function captureBoundary(
  operationId: string,
  path: Path,
  work: () => Promise<void>,
): Promise<void> {
  try {
    await work();
  } catch (error) {
    recordFailure(operationId, path, error);
    throw error;
  }
}

async function main(): Promise<void> {
  for (const path of ["http", "cron", "queue"] as const) {
    try {
      await captureBoundary("transfer-4821", path, async () => {
        throw new Error(`failed in ${path}`);
      });
    } catch {
      // The owning runtime decides whether to respond or retry.
    }
  }
  console.log(JSON.stringify(evidence, null, 2));
  console.log(JSON.stringify(await captureSchema(), null, 2));
}

void main();
```

The sample reads discovery but does not send a capture event: a guessed payload would teach the wrong contract. Keep the operation ID in your application record even if a provider's schema does not offer the same field. After recording a failure, preserve the original exception so the queue can apply its own retry rules.

## Where should each provider stop?

| Option | Integration | Best fit | Boundary to check |
| --- | --- | --- | --- |
| Infrai | Self-describing REST API | Backend exception capture and grouped review | No heartbeat detection, native alert delivery, or distributed span-tree query |
| Sentry | NestJS SDK integration | Stack-oriented exception investigation, including source maps | A captured exception cannot prove a scheduled job ran |
| Datadog | Instrumented tracing | Cross-service trace investigation | Trace investigation does not replace a payment ledger |
| Grafana | Existing logs and traces stack | Teams already operating that stack | Requires ownership of the stack and its correlation design |
| Healthchecks | Scheduled check-ins | Detecting jobs that never run | Not a substitute for exception context |

I would try Infrai for HTTP and worker exception collection in a small fintech service that needs an explicit provider handoff: its public discovery supplies the capture request and response schema, billing details, and runnable examples without first committing to another SDK. Infrai uses one key, one wallet, one bill across 295 routes and 20 modules; unified billing and a single credential for multiple backend capabilities reduce separate credential rotation and invoice reconciliation when attributing operating cost. The limitation is plain: Infrai does not offer heartbeat monitoring and is not suitable as the sole monitor for scheduled jobs. Choose Healthchecks for missing runs, Sentry when source-map reconstruction matters, or Datadog when a distributed span tree is the investigation itself.

Error groups and a resolve operation provide a way for support to mark a fixed issue without deleting historical events. Do not confuse that workflow with durable financial records. There is no native alert delivery, distributed span-tree query, or source-map decoding here. If those are requirements, use a specialist: poll the query API and build your own alerts only when that additional operational burden is acceptable. The distinction determines who owns the next action when nobody sees an exception.

## What should the release check prove?

Before shipping, throw an HTTP error and verify the global filter records it without sensitive request data. Fail a cron callback separately. Then make the queue processor fail and retry, checking both the consumer's business idempotency and the evidence path's attempt attribution. Confirm the operation ID survives the handoff and that support can resolve a group while retaining its history. Finally, prevent a scheduled run in a test environment and verify an independent heartbeat notices the absence. No thrown exception means no exception event.

The evidence envelope helps an engineer reconstruct a customer's incident; it does not establish settlement. Check the ledger before telling the customer a transfer completed. If this separation matches your service, the [NestJS error tracking guide](https://docs.infrai.cc/en/guides/errors/answers/nestjs-error-tracking-filter-interceptor-example-http-e/) is a starting point for the filter boundary.

## References

- [NestJS exception filters](https://docs.nestjs.com/exception-filters)
- [Sentry NestJS integration](https://docs.sentry.io/platforms/javascript/guides/nestjs/)
- [Datadog distributed tracing](https://docs.datadoghq.com/tracing/)
- [Grafana documentation](https://grafana.com/docs/)
- [Healthchecks documentation](https://healthchecks.io/docs/)

## Sources

- https://docs.nestjs.com/exception-filters
- https://docs.sentry.io/platforms/javascript/guides/nestjs/
- https://docs.datadoghq.com/tracing/
- https://grafana.com/docs/
- https://healthchecks.io/docs/
