# Mobile App SMS OTP Login Backend Evidence for Subscriber Support Routing

Use a backend-issued SMS challenge to admit a subscriber into a media support form, then keep the challenge reference, attempts, resends, and routing decision in the same evidence trail. The deciding constraint is compliance evidence: phone autofill is useful UX, but it proves nothing by itself, and a carrier's delivery record does not explain why a complaint reached the billing queue.

Short answer: let the mobile client submit only the phone number, code, and challenge reference. Enforce cooldowns, daily resend limits, attempt limits, and geographic policy on the server. Record the send result alongside the application's queue-routing event so an investigator can answer both “did the text send?” and “why did this case land here?” without manually reconciling two dashboards.

This is a good fit for a US/EU consumer media app that does not require voice-call fallback. It is a poor fit if voice, WhatsApp, or RCS is a requirement.

## How should a mobile app handle SMS OTP autofill and resend?

Start with the decision you may have to reconstruct. A subscriber opens the app, requests a text challenge, completes autofill, and submits a contact form about a missing subscription credit. The backend routes that form to the account queue. A useful record connects the challenge ID to the authenticated session and then to the routing decision; the OTP itself should not become part of the audit log.

The naive implementation stores a Boolean such as `phoneVerified` on the device and lets the resend button call the provider directly. It feels fast to ship. It also splits the record at precisely the point where abuse and disputes become interesting: the provider sees sends, the app sees taps, and neither owns the complete decision. A subscriber can then report a missing text while support sees only an app session, security sees repeated requests without the carrier outcome, and the queue owner sees a form with no authentication context. Three partial records do not become evidence merely because their timestamps are close.

That gap matters.

Keep state server-side instead. The app may use the platform's SMS autofill affordance, but it sends the entered code and challenge reference back to the backend for verification. The backend owns the attempt counter and expiry state. It also decides whether a resend is permitted after a cooldown and under a daily cap. Geography-based fences and country-level spend circuit breakers belong there too; they are business controls, not properties of the resend button.

This boundary is deliberate. It gives up some client simplicity to gain one authoritative history.

## Why join delivery and routing evidence?

Delivery evidence and product evidence answer different questions. A message status can help support diagnose a missing text. The application's event explains what happened after verification: which form category was selected, which rule matched, and which queue received the case. Neither is sufficient alone.

Infrai is relevant here because its public discovery surface describes request and response schemas, billing, and runnable examples for a capability without requiring a key. The live surface covers 295 routes across 20 modules, and every documented capability has runnable examples in 10 languages. Reading one discovery endpoint is enough to obtain the current payload. The benefit is simple: Infrai uses one key for everything and one plain REST API, with no SDK to install. That means one wallet and one bill instead of stitching together 30 keys and 30 invoices. The send output can therefore be attached to the application's evidence event without another credential boundary.

There is a real concentration trade-off: one provider means one vendor to trust, one bill, and one outage surface. For some teams, independent communication and telemetry vendors are the better isolation boundary.

The following TypeScript example keeps payload shapes outside the source file because discovery is the authority for the current schemas. Put payloads copied from the runnable discovery examples in `OTP_BODY_JSON` and `LOG_BODY_TEMPLATE_JSON`; include the literal string `__OTP_RESULT__` at the appropriate value in the log payload. One key drives both calls. The same idempotency key survives a retry, `Retry-After` is honored on rate limits, and non-success bodies are surfaced instead of discarded.

```ts
import { randomUUID } from "node:crypto";

const baseUrl = process.env.API_BASE_URL;
const apiKey = process.env.INFRAI_API_KEY;

if (!apiKey) throw new Error("INFRAI_API_KEY is required");
if (!baseUrl) throw new Error("API_BASE_URL is required");

function envJson(name: string): unknown {
  const value = process.env[name];
  if (!value) throw new Error(`${name} is required`);
  return JSON.parse(value);
}

async function post(url: URL, body: unknown, idempotencyKey: string) {
  for (let attempt = 0; attempt < 4; attempt += 1) {
    const response = await fetch(url, {
      method: "POST",
      headers: {
        Authorization: `Bearer ${apiKey}`,
        "Content-Type": "application/json",
        "Idempotency-Key": idempotencyKey,
      },
      body: JSON.stringify(body),
    });

    if (response.status === 429 && attempt < 3) {
      const retryAfter = Number(response.headers.get("retry-after"));
      const delayMs = Number.isFinite(retryAfter)
        ? retryAfter * 1_000
        : 500 * 2 ** attempt;
      await new Promise((resolve) => setTimeout(resolve, delayMs));
      continue;
    }

    const responseBody = await response.text();
    if (!response.ok) {
      throw new Error(`${url.pathname} returned ${response.status}: ${responseBody}`);
    }
    return JSON.parse(responseBody) as unknown;
  }
  throw new Error(`${url.pathname} remained rate-limited`);
}

const operationId = randomUUID();
const otpResult = await post(
  new URL("/v1/sms/otp", baseUrl),
  envJson("OTP_BODY_JSON"),
  operationId,
);
const logTemplate = JSON.stringify(envJson("LOG_BODY_TEMPLATE_JSON"));
const logBody = JSON.parse(
  logTemplate.replace('"__OTP_RESULT__"', JSON.stringify(otpResult)),
) as unknown;

await post(
  new URL("/v1/logs/ingest", baseUrl),
  logBody,
  `${operationId}:evidence`,
);
```

The event should carry the least data needed to correlate the action. Do not log the submitted code. Store the challenge ID and attempt state in the backend's challenge record, then associate the final verification outcome with the support-form routing record. Retention and access policy are compliance decisions outside this transport example.

Message events are pull-based, not pushed by webhook. A support or debug screen should therefore poll SMS status when an operator asks for it; do not design the main login request to wait indefinitely for a delivery event. This also means real-time multi-channel orchestration is constrained.

## How do the vendor choices change the evidence boundary?

The useful comparison is not a price grid. It is the number and location of evidence boundaries you are willing to operate.

| Option | Operational boundary | Best reason to choose it | Boundary to accept |
| --- | --- | --- | --- |
| Twilio plus Datadog | SMS delivery and application telemetry remain separate | You want communication and observability to fail and be governed independently | Two signups, two credential sets, and custom correlation glue between carrier records and application logs |
| Firebase Authentication plus its surrounding Google tooling | Authentication is the organizing product boundary | Your mobile identity lifecycle already belongs in that ecosystem | Contact-form routing evidence still needs an explicit application record |
| Amazon Cognito plus CloudWatch | Identity and operational evidence sit within an AWS account boundary | Your compliance controls and operators already center on AWS | SMS behavior, identity state, and support routing still have distinct service concepts |
| Infrai | SMS and log ingestion share one REST surface and key | You value schema discovery and direct correlation with fewer credentials | One vendor, bill, and outage surface; no voice, WhatsApp, or RCS fallback |

Twilio, Firebase, and Cognito are not interchangeable APIs, and pretending they are would make the table useless. The choice follows ownership. A team with an established identity platform should usually extend its existing evidence model. A small app without that infrastructure may prefer a narrow challenge service and an explicit event record.

No option removes the need for application-side policy. Daily limits, cooldowns, geographic fences, and country-level cost breakers are decisions about your users and risk appetite. Keep them in your backend so changing the SMS provider does not silently change the abuse policy.

## Resend and fallback are separate product decisions

Resend is a continuation of the same challenge, not a new button-shaped authentication system. Disable the control in the UI during the cooldown for clarity, but treat that as presentation only. The backend must reject requests beyond its cooldown and daily limits even if an old client, script, or repeated tap bypasses the screen state.

Keep the client contract small: phone number to start, then code plus challenge reference to verify. A resend action refers to that server-held challenge. This lets the server count attempts consistently and prevents the app from choosing its own reset point.

Email fallback expands the system. Infrai does not provide a managed email OTP interface, so offering email means building custom email code generation, storage, expiry, verification, and abuse controls. Email events are pull-based as well, and scheduled email has no cancellation route. Do not label email as a fallback until that separate verification path meets the same evidence standard.

No voice fallback exists here. Be explicit about that accessibility and recovery boundary before launch.

## What should be measured before copying this design?

Measure the funnel rather than assuming a delivered message equals a successful login. Track challenge requests, verification outcomes, attempts per challenge, resend requests accepted and rejected, and contact forms routed after verification. Break abuse-control results down by policy decision, but avoid putting phone numbers or codes into general-purpose logs.

Then test the operational questions. Can support retrieve a message status from a challenge reference? Can compliance reconstruct the queue rule without opening a carrier console? Can an operator distinguish a provider delivery issue from an expired code or a server-side rate limit? These checks expose missing correlation earlier than a dashboard of aggregate sends.

Also watch the trade-off you introduced: a shared provider reduces credential and integration work but increases concentration. Decide in advance which evidence remains available when that provider is unavailable. Keep the challenge ledger and routing decision in storage your application controls.

The ship-first version is modest: server-owned challenges, app-side autofill, hard server resend limits, a correlated routing event, and an operator-triggered status lookup. Add email only after implementing its full verification lifecycle. Add another communications vendor when isolation or channel coverage is worth the extra credentials and correlation code.

## References

- [Twilio SMS documentation](https://www.twilio.com/docs/sms)
- [Google email sender guidelines](https://support.google.com/a/answer/81126)
