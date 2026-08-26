# Daily Report Email Recovery: Queue Dead Letters Beat Cron Reruns for Shipment Updates

When a healthtech shipment update fails, the retry decision is about identity, not timing. **Short answer: use the daily schedule only to enqueue one idempotent message per recipient, then let bounded queue retries and a dead-letter queue handle failed sends; reserve a cron rerun for repairing the batch record, not replaying every email.**

That split keeps a timeout from sending the same delivery twice and lets an operator inspect the few recipients that still fail. It also gives a solo team a useful measurement point: count attempts, permanent failures, and duplicate-suppression hits before changing the design.

## Consent and audit integration for the delivery ledger

Start with a delivery key. For a shipment report, I use `shipmentId + reportDate + recipientId`, normalized and hashed before it enters the queue. The worker writes that key to a database table with a unique constraint before it calls the email provider, or records a successful provider message ID in the same transaction boundary. A second delivery sees the key and exits without sending.

That order matters. If the worker sends first and records later, a process crash between those actions creates an ambiguous result. If it records first and the provider rejects the request, the row must remain `pending` so a retry can try again. There is no universal transaction spanning your database and an email service, so make the state machine explicit: `pending`, `sent`, `permanent_failure`, and `dead_lettered` are enough for a first version.

I learned this while prototyping a smaller notification flow: a request timed out at 30 seconds, the scheduler reran the whole batch, and one subscriber received two copies. The log showed `ETIMEDOUT`, but it did not say whether the provider had accepted the message. That is the exact reason to make the send operation idempotent at your boundary and to attach the same key to every retry.

Short keys. Clear states.

## How can a Node.js example coordinate daily report email retries and a DLQ?

The scheduler should do very little: identify the report date, enumerate recipients, publish messages, and return. A worker owns retry policy. After a bounded attempt count, the queue moves the message to a dead-letter queue (DLQ) with the original payload, error class, and attempt number. A person or a controlled replay job can then correct a bad address, consent status, or template and redrive selected messages.

Here is a compact TypeScript shape. The endpoint names are intentionally generic; the contract is the important part.

```ts
type ReportMessage = {
  shipmentId: string;
  reportDate: string;
  recipientId: string;
  email: string;
  idempotencyKey: string;
};

type DeliveryState = "pending" | "sent" | "permanent_failure" | "dead_lettered";

async function enqueueDailyReport(reportDate: string, recipients: Array<{ shipmentId: string; recipientId: string; email: string }>) {
  const messages: ReportMessage[] = recipients.map((r) => ({
    ...r,
    reportDate,
    idempotencyKey: `${r.shipmentId}:${reportDate}:${r.recipientId}`,
  }));

  await queue.publishBatch("shipment-reports", messages, {
    deduplicationKey: `daily-report:${reportDate}`,
  });
}

async function deliver(message: ReportMessage): Promise<DeliveryState> {
  const existing = await db.delivery.findUnique({ where: { idempotencyKey: message.idempotencyKey } });
  if (existing?.state === "sent" || existing?.state === "permanent_failure") return existing.state;

  await db.delivery.upsert({
    where: { idempotencyKey: message.idempotencyKey },
    create: { ...message, state: "pending" },
    update: {},
  });

  try {
    const result = await mail.send({ to: message.email, template: "shipment-daily", data: message });
    await db.delivery.update({ where: { idempotencyKey: message.idempotencyKey }, data: { state: "sent", providerId: result.id } });
    return "sent";
  } catch (error) {
    if (isPermanentEmailError(error)) {
      await db.delivery.update({ where: { idempotencyKey: message.idempotencyKey }, data: { state: "permanent_failure" } });
      return "permanent_failure";
    }
    throw error; // the queue retries transient failures with backoff
  }
}
```

The queue should use exponential backoff with jitter, a visibility timeout longer than the normal send latency, and a maximum receive count. A `429`, connection reset, or temporary DNS failure is usually transient. An invalid mailbox, revoked consent, or a template validation error is deterministic and should be marked permanent without burning all attempts.

Cron reruns still have a job. If publishing stops halfway through, rerun the publisher with the same `daily-report:<date>` deduplication key and let the queue ignore messages it already accepted. Do not rerun the entire send loop inline; that couples the scheduler's timeout to the slowest recipient and makes overlap hard to reason about.

## Retry budget by error class

A queue plus DLQ is not a universal workflow engine. It is a poor fit when a report requires a long, ordered saga: wait for a carrier confirmation, update a ledger, obtain a clinician approval, then send three different notices with compensating actions. Use a durable workflow system for that, because it models timers and state transitions directly.

It is also unsuitable when a regulator requires a replayable event history longer than the queue retention window. An acknowledged queue message is gone; keep an append-only event stream or object-store record if auditors must reconstruct every attempt. Stick with a plain cron rerun when the batch is tiny, the operation is naturally idempotent, and losing per-recipient visibility is acceptable.

The catch is operational weight. You now own a worker, a delivery table, DLQ alerts, replay permissions, and a runbook. For a report sent to twelve internal addresses, that may be needless machinery. For thousands of patients or clinics, the visibility usually pays for itself.

## How do you measure retries before changing the replay policy?

Measure the distribution, not just the success rate. Track first-attempt success, retry count by error class, time spent in `pending`, DLQ age, duplicate-suppression count, and the number of messages manually redriven. Slice those metrics by recipient domain and template version; a single broken domain or template can hide inside a healthy aggregate.

I keep a fixture set with a deliberately slow SMTP response, a synthetic `429`, a malformed address, and a process kill after the provider call. The last case is the one most teams skip. In one staging drill, I killed the worker immediately after it received a provider response but before the database transaction committed; the queue redelivered the message, and the only way to tell if the second attempt was safe was to inspect the provider message ID alongside the idempotency row. That drill changed our runbook: operators now search the delivery key first, then the provider ID, and only then redrive a DLQ item. Your mileage may vary by provider, and I'm not sure any dashboard can infer acceptance after a timeout without a provider message ID, so test that ambiguity explicitly.

Test the ugly boundary.

Before copying the pattern, run a replay of one real report date in a staging mailbox. Verify that a second scheduler run creates zero additional sends, that permanent errors bypass further retries, and that a DLQ redrive preserves the original idempotency key. Those are the acceptance tests; a green cron metric alone is not.

## Rollback boundaries for a cron rerun

Use the simpler rerun when the batch is small, every send is naturally idempotent, and an operator can inspect the complete recipient list in one screen. A queue is the better fit when one slow or invalid address must not delay healthy recipients, or when the report needs per-recipient consent and audit state.

The choice should follow those observations, not a feature checklist.

## References

- Wikipedia, Cron: https://en.wikipedia.org/wiki/Cron
- Google Cloud Pub/Sub overview: https://cloud.google.com/pubsub/docs/overview
- RFC 5321, Simple Mail Transfer Protocol: https://www.rfc-editor.org/rfc/rfc5321
- RFC 7231, HTTP semantics and status codes: https://www.rfc-editor.org/rfc/rfc7231
