# Node.js Express Failure Alerts in 2026: Why I Chose Log-Based Metrics

Short answer: I chose a small, structured evidence stream plus derived failure counters for a Node.js property-management API. Logs retain the detail needed to reconstruct a resident incident; counters make detection cheap and predictable. Every record carries a stable operation name, outcome, property ID, and cost attribution key, while alerts evaluate rates over windows rather than repeatedly searching raw text. This is the useful compromise when the real goal is failure alerting without turning log volume or vendor choice into the architecture.

It also keeps two questions separate: **Is the service failing now?** and **what happened to this customer's request?** A metric answers the first. A bounded evidence trail answers the second.

## Why not alert by searching every error log?

A text search feels ship-first because the application already emits logs. It breaks down as an alert contract, though. Wording changes, stack traces inflate event size, retries create several records for one operation, and a broad `error` match mixes customer-visible failures with recovered attempts. The resulting alert is hard to attribute to a property or workflow, and its evaluation cost follows retained log volume.

Polling an error-status endpoint has a different weakness: the poll observes snapshots. A short failure can begin and recover between polls, while a long failure can page repeatedly unless the polling process owns state. The API being monitored also should not be the only place that stores its own failure history.

I use logs as evidence and metrics as the alarm input. The distinction is practical. A counter is intentionally low-detail; an evidence event is rich but bounded. Shared fields let an operator move from an alert to the relevant records without making arbitrary message text part of the monitoring contract.

One event. One outcome.

## A runnable Express evidence path

The data flow is short. Express assigns a request ID, the route names the business operation, and one completion handler records the final HTTP outcome. That handler increments an example counter and emits one JSON evidence record. In production, the counter would go to a metrics backend and stdout would go to an independently retained log pipeline, but the application contract stays the same.

```ts
import express, { NextFunction, Request, Response } from "express";
import { randomUUID } from "node:crypto";

type Outcome = "success" | "failure";
type Labels = { operation: string; outcome: Outcome };

const app = express();
const totals = new Map<string, number>();

function increment({ operation, outcome }: Labels): void {
  const key = `${operation}:${outcome}`;
  totals.set(key, (totals.get(key) ?? 0) + 1);
}

function evidence(operation: string) {
  return (req: Request, res: Response, next: NextFunction): void => {
    const requestId = req.header("x-request-id") ?? randomUUID();
    const startedAt = Date.now();

    res.on("finish", () => {
      const outcome: Outcome = res.statusCode >= 500 ? "failure" : "success";
      increment({ operation, outcome });

      const record = {
        timestamp: new Date().toISOString(),
        requestId,
        operation,
        outcome,
        statusCode: res.statusCode,
        durationMs: Date.now() - startedAt,
        propertyId: req.header("x-property-id") ?? "unknown",
        costCenter: req.header("x-cost-center") ?? "unattributed"
      };

      process.stdout.write(`${JSON.stringify(record)}\n`);
    });

    next();
  };
}

app.post("/maintenance-requests", evidence("maintenance_request_create"), (_req, res) => {
  res.status(202).json({ accepted: true });
});

app.get("/internal/failure-counts", (_req, res) => {
  res.json(Object.fromEntries(totals));
});

app.listen(3000);
```

This records one final event, not one event per middleware step. The operation label is fixed code, so it has bounded cardinality. Property and cost-center identifiers remain in evidence records rather than metric labels; customer identifiers can create a large and continuously changing metric series set. Do not log resident names, messages, access codes, authorization headers, or request bodies. An incident trail is not permission to duplicate sensitive data.

The sample treats 5xx responses as failures because they represent the server-side failure class for this route. That is only a starting policy. A maintenance submission that returns 202 but never reaches its worker needs a separate terminal event or a timeout counter. A 409 may be an expected duplicate. Model the business outcome explicitly when HTTP status cannot express it.

## Cost attribution changes the schema

For a property platform, a global failure ratio can hide the expensive shape of an incident. One property may generate most failed retries; one workflow may attach unusually large model inputs; one background job may fan out after a request has returned. I want enough dimensions to answer who incurred work and why, without putting every dimension into the alert metric.

| Field | Metric label | Evidence record | Reason |
|---|---:|---:|---|
| `operation` | Yes | Yes | Small, controlled vocabulary |
| `outcome` | Yes | Yes | Drives the failure ratio |
| `propertyId` | No | Yes | Needed for incident scope; grows with customers |
| `costCenter` | No | Yes | Supports allocation and later aggregation |
| `requestId` | No | Yes | Joins the request path, never an aggregate dimension |
| `durationMs` | Observe | Yes | Useful as a distribution and request evidence |

This creates an honest limitation: the first alert says which operation is unhealthy, not which property caused it. The investigation query then groups a bounded time range of evidence by `propertyId` or `costCenter`. That extra step is preferable to paying a permanent cardinality cost for a dimension needed only during investigation.

Keep raw evidence retention tied to the time in which support can reopen an incident, then retain cheaper aggregates for longer cost review. Exact durations are policy decisions; there is no universal number. Write them down anyway. Otherwise retention grows by accident.

## Make the alert describe customer impact

A useful rule evaluates failures and traffic together. A raw count of five failures means something different at ten requests than at ten thousand. Conversely, a ratio on one request is too fragile to page anyone. I require both a minimum request volume and a failure-ratio threshold over a stated window, then use a longer window or repeated evaluations before paging. Exact thresholds come from the service objective and normal traffic, not an article. No traffic deserves its own signal: for a route expected to receive continuous work, zero completions can indicate a broken ingress or worker even though the failure counter is also zero, while a quiet overnight workflow would make the same rule noisy. Calendar-aware expectations belong in the alert policy. Retries need one owner as well. Emit attempt evidence if it helps debugging, but increment the customer-visible failure counter only when the operation exhausts its retry policy or crosses its deadline. Otherwise one failed request can look like three failures, and cost attribution becomes equally misleading. The trade-off is explicit: detection keeps a compact vocabulary, while investigation pays for a narrow evidence query only after a signal fires.

Silence can fail.

Feature flags add a useful dimension during deployment, but I keep high-cardinality flag combinations out of the primary metric. Record relevant flag state in evidence, compare a small number of controlled cohorts during rollout, and remove temporary dimensions after the decision. Long-lived toggles carry operational cost and need active management, as Martin Fowler's feature-toggle guidance explains.

## What should I verify before paging anyone?

First, test the instrumentation as behavior. A successful request must produce one success count and one evidence record. An unhandled route error must produce one failure count with the same request ID that appears in its evidence. A recovered retry must not become a customer-visible failure. Run those checks in CI against JSON parsing, not fragile substring matches.

Then exercise the alert path in a non-production environment by injecting a controlled terminal failure behind a short-lived flag. Confirm the counter rises, the rule changes state after its intended window, the notification includes the operation and runbook link, and recovery resolves rather than creating a second page. Remove the flag after the test.

Deployment order matters. Ship the metric and evidence fields before enabling the alert. Observe normal volume across the operating cycle, choose thresholds from that baseline and the service objective, and only then route notifications. During rollout, compare the derived metric with a bounded log query; disagreement can expose duplicate completion handlers, missing worker outcomes, or inconsistent operation names.

The operational checklist is prose because the work is sequential. Establish ownership for the alert, document the business meaning of failure, cap metric dimensions, redact evidence at emission, verify independent retention, link the notification to a query keyed by operation and time, and rehearse recovery. Finally, review which fields are actually used. Delete dead fields before they become permanent storage cost.

Cheap is a consequence of this design, not its headline. **Use low-cardinality metrics to detect, structured evidence to explain, and explicit attribution fields to assign cost.** For a Node.js property API, I would choose this pattern when the team needs prompt failure detection and customer-incident reconstruction, but customer identifiers do not belong in the paging metric.

## Sources

- https://opentelemetry.io/docs/specs/semconv/http/http-metrics/
- https://prometheus.io/docs/practices/naming/
- https://prometheus.io/docs/practices/alerting/
- https://owasp.org/www-project-logging-cheat-sheet/
- https://nodejs.org/api/crypto.html#cryptorandomuuidoptions
- https://expressjs.com/en/4x/api.html
- https://martinfowler.com/articles/feature-toggles.html
