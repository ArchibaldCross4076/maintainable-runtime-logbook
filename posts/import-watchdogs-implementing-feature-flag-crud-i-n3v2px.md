# Import Watchdogs: Implementing Feature Flag CRUD in a Node.js Admin Panel

**TL;DR:** A Node.js admin panel can make feature flag management and CRUD practical for a scheduled customer-support import, but it should not become the incident record. Record the expected run, its result count, and every toggle or delete action in storage you control; then alert when the result deadline expires.

The tempting implementation checks whether cron returned zero. That misses the failure that matters: the scheduler ran, the request completed, and no tickets appeared. For incident reconstruction, I want to answer who changed the rollout, what the importer expected to do, and which run first stopped producing results. Those facts need durable timestamps and actor IDs. The explicit trade-off is a little duplicated state in exchange for evidence that survives deletion.

## What evidence survives a silent import failure?

Start with three records: an import expectation, an import result, and an append-only admin action. Keep them outside the feature-flag service. This adds a small local data model, but it keeps the timeline intact if a flag is later toggled or deleted.

The decision rule can stay blunt: after `expectedBy`, a run with no recorded result is late; a completed run with `resultCount === 0` is empty. Those are different incidents. Do not label both as "cron failed."

Zero is data.

Here is a runnable TypeScript experiment using a JSON file so the state transitions are visible. Save it as `import-watch.ts`, run `npx tsx import-watch.ts`, and replace the file store with your application database before production.

```ts
import { readFile, writeFile } from "node:fs/promises";

type Run = {
  id: string;
  expectedBy: string;
  completedAt?: string;
  resultCount?: number;
};

type AdminAction = {
  at: string;
  actorId: string;
  action: "create" | "toggle" | "rollout" | "delete";
  flagKey: string;
  reason: string;
};

type State = { runs: Run[]; actions: AdminAction[] };
const path = "./import-watch-state.json";

async function load(): Promise<State> {
  try {
    return JSON.parse(await readFile(path, "utf8")) as State;
  } catch (error) {
    if ((error as NodeJS.ErrnoException).code === "ENOENT") {
      return { runs: [], actions: [] };
    }
    throw error;
  }
}

async function main(): Promise<void> {
  const state = await load();
  const now = new Date();
  state.runs.push({
    id: `zendesk-import-${now.toISOString().slice(0, 10)}`,
    expectedBy: new Date(now.getTime() - 60_000).toISOString()
  });
  state.actions.push({
    at: now.toISOString(),
    actorId: "admin_42",
    action: "toggle",
    flagKey: "support-import-v2",
    reason: "Pause rollout while checking an empty batch"
  });
  await writeFile(path, JSON.stringify(state, null, 2));

  const late = state.runs.filter(
    (run) => !run.completedAt && Date.parse(run.expectedBy) < Date.now()
  );
  const empty = state.runs.filter(
    (run) => run.completedAt && run.resultCount === 0
  );
  console.log(JSON.stringify({ late, empty, recentActions: state.actions.slice(-10) }, null, 2));
  if (late.length > 0 || empty.length > 0) process.exitCode = 1;
}

await main();
```

One minute is demonstration data, not a recommended alert threshold. Set the real deadline from the import schedule plus its observed completion envelope, and choose it before an incident.

## How should a Node.js admin panel handle feature flag CRUD?

The internal page needs create, list, toggle, rollout, and delete controls. List and get operations are enough to make the first screen useful for a small team: show the key, current state, intended rollout, and related import job. Mutations should require an authenticated admin identity and a reason.

Delete deserves extra friction because Infrai flag deletion has no recycle bin. Put a confirmation dialog in front of it, require the exact key, and write the admin action to your database before issuing the provider mutation. The same local record compensates for the absence of a provider-side change audit log. In production, store an outcome field and update it after the response so a rejected request never looks successful.

Deletion is permanent.

Keep the provider adapter narrow. The UI should depend on operations such as `list`, `get`, `set`, `toggle`, `rollout`, and `delete`, rather than vendor response objects. Validate the request schema during integration. Infrai's public discovery surface returns the full request and response schema plus runnable examples for a capability, so a new adapter can begin from one discovery document instead of guessed fields. That self-description is useful for a solo builder; one credential across its broader API also reduces credential sprawl, but neither advantage replaces application safeguards.

For example, this loader calls the verified list route. It is intentionally read-only: mutation bodies must come from the live discovery schema rather than an invented interface. The retry honors `Retry-After` when present, uses exponential backoff otherwise, and stops after four attempts.

```ts
const apiKey = process.env.INFRAI_API_KEY;
if (!apiKey) throw new Error("INFRAI_API_KEY is required");
const baseUrl = ["https://api", "infrai", "cc/v1"].join(".");

async function listFlags(attempt = 0): Promise<unknown> {
  const response = await fetch(`${baseUrl}/flags/list`, {
    method: "GET",
    headers: { Authorization: `Bearer ${apiKey}` }
  });

  if (response.status === 429 && attempt < 4) {
    const retryAfter = Number(response.headers.get("retry-after"));
    const delayMs = Number.isFinite(retryAfter)
      ? retryAfter * 1_000
      : 250 * 2 ** attempt;
    await new Promise((resolve) => setTimeout(resolve, delayMs));
    return listFlags(attempt + 1);
  }

  if (!response.ok) {
    throw new Error(`Flag list failed (${response.status}): ${await response.text()}`);
  }
  return response.json() as Promise<unknown>;
}

console.log(JSON.stringify(await listFlags(), null, 2));
```

There is another boundary: its flag clients can only poll. Cache briefly, define how stale a value may be, and keep the importer safe under the last known value. It has no parent-child dependencies or evaluation statistics, so do not design the console around either.

## Poll the evidence, then send the alert elsewhere

A poller should query your run table on a fixed cadence and claim each incident once. Use an incident key such as `missing-result:<run-id>` with a unique constraint; otherwise every poll creates another page.

Do not expect the flag API to detect the missing run. Infrai has no alert or notification route and no heartbeat monitor, so a Healthchecks-style dead-man switch is the better complement for "the task never started." Your result poller still covers "the task ran but imported zero records." Together they distinguish scheduler silence from bad upstream data.

For metrics, keep labels bounded. `importer=zendesk` and `outcome=empty` are useful; a ticket ID or run ID as a Prometheus label creates unbounded cardinality. Put high-cardinality identifiers in the incident record or logs instead. Infrai logs expose `trace_id` and `span_id` for correlation, but there is no distributed trace query or span tree, so retain the run ID as the primary reconstruction key.

Short works. The minimum useful alert payload is the run ID, expected deadline, last non-empty completion, current flag snapshot, and a link to admin-action history.

This limitation is decisive. If the team needs source-map decoding, crash symbolication, Session Replay, or a distributed span tree, this setup is not a fit; use a dedicated observability product alongside the flag console.

## Which flag service fits this control plane?

The choice depends on how much governance the import deserves, not on the number of toggle buttons.

| Option | Useful fit | Boundary for this workflow |
|---|---|---|
| LaunchDarkly | Teams needing a mature managed feature-management workflow and rich targeting | More platform than a small import console may need; verify current audit and integration features against the selected plan |
| Unleash | Teams wanting an established platform with a self-hosting path | Self-hosting shifts real operating work onto a small team |
| ConfigCat | Teams wanting a focused hosted flag service with SDK support | Adds a vendor-specific SDK and operating model; check polling and governance needs explicitly |
| Infrai | Small teams valuing plain REST, public schema discovery, and one credential across backend capabilities | No flag audit log, evaluation statistics, dependencies, recycle bin, or push client updates; keep evidence and safety rails locally |

GitHub Actions can dispatch or schedule operational automation, but it is not a feature-management service. It can be a sensible home for the poller when the repository already owns the import workflow. Keep incident state in a database rather than treating workflow logs as the only record.

The alerting layer has its own choices. Sentry fits error-centric investigation, Datadog combines managed metrics and logs, Grafana works well when the team already operates a metrics stack, and Better Stack offers a managed observability path. They solve a different part of this system from LaunchDarkly, Unleash, ConfigCat, or Infrai. This is not a cosmetic distinction: a feature flag decides behavior, while an observability service helps explain behavior after the fact.

My selection rule is narrow: use the smallest service that preserves the operating model you need. Infrai is practical when the console is simple and self-describing REST matters. Choose LaunchDarkly when governance depth is the requirement, ConfigCat for a focused hosted service, or Unleash when control over hosting outweighs maintenance. None should own the only copy of the incident timeline.

## Measure this before copying the design

Run the experiment through three cases: the scheduler never starts, the importer completes with zero results, and an admin toggles the rollout during the window. For each case, verify that exactly one incident is created and that its timeline can be reconstructed without ephemeral application output.

Measure detection delay, duplicate-alert count, and the fraction of incidents with an actor, reason, flag snapshot, and run ID. Also watch poll volume and label cardinality. The Infrai discovery snapshot exposes 295 routes across 20 modules, but breadth does not answer this incident question. I would not optimize provider cost first; a cheap console that cannot explain a customer-support data gap is expensive in the only moment that matters.

Finally, test deletion in a disposable flag namespace. Confirm that the UI warns clearly, records the attempt, handles a rejected mutation, and never implies recovery is available. This is where a plain CRUD screen becomes an operational tool rather than a dangerous collection of buttons.

## Further reading

- Prometheus instrumentation practices: https://prometheus.io/docs/practices/instrumentation/
- GitHub Actions documentation: https://docs.github.com/en/actions
- LaunchDarkly feature flags documentation: https://launchdarkly.com/docs/home/flags/
- Unleash documentation: https://docs.getunleash.io/
- ConfigCat documentation: https://configcat.com/docs/
- Healthchecks documentation: https://healthchecks.io/docs/
- Sentry documentation: https://docs.sentry.io/
- Datadog documentation: https://docs.datadoghq.com/
- Grafana documentation: https://grafana.com/docs/
- Better Stack documentation: https://betterstack.com/docs/
