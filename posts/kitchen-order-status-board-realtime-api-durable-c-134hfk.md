# Kitchen Order Status Board Realtime API — Durable Cursor Security Boundaries

TL;DR: Push each kitchen transition through a server-owned stream, but treat that stream as a hint rather than the order ledger. Give every kiosk a short-lived, display-only credential scoped to one location, attach a monotonically increasing sequence to each event, and reconnect with the last applied sequence. The server then backfills the gap before resuming live delivery. This is the least complex design that keeps a restaurant queue screen current without polling or granting it kitchen-write privileges.

For an order status board, I would choose a one-way server-to-kiosk channel over a peer-to-peer topology. The flow is plain: the kitchen writes a transition to the authoritative order service; that write and its event receive a durable position; a fanout process sends the event to screens for the same location; each screen applies it once and remembers the position. Reconnect repair comes from the event history, not from hoping the network retained every message.

## How should a kitchen order status board use a realtime API?

A green connection indicator is not proof that the board is correct. A display can disconnect after order 184 changes from `preparing` to `ready`, reconnect seconds later, and receive order 185 as its first live event. Without a cursor, the screen looks healthy while one ticket remains in the wrong column.

Missed means stale.

The useful contract has four parts. Events are ordered within a location. Every accepted kitchen transition has a stable event ID and sequence. A kiosk persists the highest contiguous sequence it has applied. On reconnect, the server returns all later events before it joins the live tail. If the retained history no longer reaches that cursor, the server returns a fresh location snapshot plus its sequence watermark instead.

Contiguous matters. Suppose a screen has applied sequences 410 and 412 but never saw 411. Saving 412 as its cursor would hide the hole forever. Hold 412 briefly, request from 410, and advance the durable cursor only after 411 and 412 have both been applied.

Walk that example through the full failure instead of treating it as a socket reconnect. The kiosk has rendered 410 and persisted 410. Event 411 reaches the fanout service while the access point is changing channels, so the display never receives it. Event 412 arrives after the connection recovers, but the client sees the gap and leaves its durable cursor at 410. It may keep 412 in a bounded buffer, yet it must not render 412 if doing so would show a later state ahead of 411. The client reconnects with 410 and a newly issued credential for the same location. The server fixes a replay watermark, returns 411 and 412 in that order, and only then releases events beyond the watermark. The kiosk applies both, stores 412, and can discard its buffered duplicate by event ID. Now change one condition: retained history begins at 500. Replay cannot repair a cursor at 410, so the server sends a current snapshot stamped with its own watermark. The kiosk replaces its projection atomically and resumes after that watermark. This timeline is more work than reconnecting a socket, but every step has an observable input and output; a mysterious stale ticket becomes a cursor gap, a replay, or a snapshot fallback.

Keep the state machine narrow as well. A kiosk should render transitions such as `accepted`, `preparing`, `ready`, and `collected`; it should not infer them from timestamps or rearrange them locally. The writer validates whether a transition is allowed. The display consumes the resulting fact. This separates kitchen authority from presentation and makes replay deterministic.

That is the boundary.

## A small replay contract in TypeScript

The following example isolates the two mechanisms that matter: scoped authorization and cursor-based replay. It uses Node's standard cryptography module, so the security boundary is visible rather than hidden behind a framework. The in-memory event array is intentionally replaceable; production durability belongs in the same transaction boundary as the order transition.

```ts
import { createHmac, timingSafeEqual } from "node:crypto";

type OrderState = "accepted" | "preparing" | "ready" | "collected";

type DisplayClaim = {
  audience: "order-board";
  locationId: string;
  permissions: ["orders:read"];
  expiresAt: number;
};

type OrderEvent = {
  id: string;
  locationId: string;
  sequence: number;
  orderId: string;
  state: OrderState;
};

const secret = Buffer.from(process.env.DISPLAY_TOKEN_SECRET ?? "", "utf8");
if (secret.length < 32) throw new Error("DISPLAY_TOKEN_SECRET must be at least 32 bytes");

function encode(value: unknown): string {
  return Buffer.from(JSON.stringify(value)).toString("base64url");
}

function sign(claim: DisplayClaim): string {
  const payload = encode(claim);
  const signature = createHmac("sha256", secret).update(payload).digest("base64url");
  return `${payload}.${signature}`;
}

function verify(token: string, expectedLocation: string, now = Date.now()): DisplayClaim {
  const [payload, supplied] = token.split(".");
  if (!payload || !supplied) throw new Error("Malformed display token");

  const expected = createHmac("sha256", secret).update(payload).digest();
  const actual = Buffer.from(supplied, "base64url");
  if (actual.length !== expected.length || !timingSafeEqual(actual, expected)) {
    throw new Error("Invalid display token");
  }

  const claim = JSON.parse(Buffer.from(payload, "base64url").toString("utf8")) as DisplayClaim;
  if (claim.audience !== "order-board") throw new Error("Wrong audience");
  if (claim.locationId !== expectedLocation) throw new Error("Wrong location");
  if (claim.permissions.length !== 1 || claim.permissions[0] !== "orders:read") {
    throw new Error("Display token is not read-only");
  }
  if (claim.expiresAt <= now) throw new Error("Expired display token");
  return claim;
}

const events: OrderEvent[] = [];

function replay(locationId: string, after: number): OrderEvent[] {
  return events
    .filter((event) => event.locationId === locationId && event.sequence > after)
    .sort((a, b) => a.sequence - b.sequence);
}

const token = sign({
  audience: "order-board",
  locationId: "location-17",
  permissions: ["orders:read"],
  expiresAt: Date.now() + 5 * 60_000
});

verify(token, "location-17");
console.log(replay("location-17", 410));
```

This token format is an illustrative application contract, not a claim that a homegrown token is universally preferable. The important checks are explicit: signature, audience, location, permission, and expiry. A mature deployment may use an established signed-token format or opaque session credentials, but the verifier still needs equivalent boundaries. Never put a long-lived kitchen credential on a kiosk merely because the device stays mounted to a wall. Physical access changes the threat model.

The sample uses a five-minute lifetime to make rotation behavior concrete, not to prescribe a universal value. Pick the duration from the kiosk's unattended period, the credential refresh path, and the time needed to revoke a lost device. Refresh before expiry while the current stream is alive. If refresh fails, retain the last rendered state, mark it stale, and retry authentication without pretending new orders are arriving.

## Reconnect is a data problem, not a socket setting

Transport selection comes after the recovery contract. A one-way stream is sufficient when a display only receives events. A bidirectional connection is justified if the kiosk must send application messages over the same channel, although acknowledgements alone do not remove the need for durable replay. WebRTC exposes a peer-connection model; a restaurant board whose source of truth already lives on a server usually gains little from making kitchen displays peers.

No transport promises that an unattended screen will remain connected. Wi-Fi roaming, a sleeping browser, a process restart, or an expired credential can all create a gap. The application therefore needs to define the order of reconnect operations:

1. Load the last contiguous sequence from durable local storage.
2. Obtain a valid credential for the same kiosk and location.
3. Request replay after that sequence.
4. Apply events in sequence, ignoring already-applied IDs.
5. Persist the new contiguous cursor, then attach to the live tail.

There is a race between replay and live subscription if those operations are unrelated. Close it server-side: establish the live boundary at a known watermark, replay through that watermark, then release buffered live events. Another valid shape is a single subscribe operation that accepts a cursor and performs both phases. The public API can differ; the invariant cannot.

Backpressure deserves an explicit rule. A kiosk that cannot consume quickly enough should be disconnected with its cursor intact, then replay from durable history. An unbounded per-screen memory queue converts one slow display into a server memory problem. Coalescing can be safe for a full snapshot, but it is risky for transitions when audit order matters.

The trade-off is extra storage, cursor bookkeeping, and a snapshot path. This approach is not a fit when the screen can always rebuild from a tiny current-state response and a missed intermediate transition has no operational value; a periodic snapshot after reconnect is then easier to reason about. It is also a poor fit for direct browser-to-browser media, where peer connection and media transport are the actual job. For an order board that must preserve state changes across unattended disconnects, I would accept the added replay machinery because correctness is testable. I would not accept it merely to make a connection counter look impressive.

## Scope credentials to the screen's actual job

A dining-room board needs less authority than the kitchen terminal that changes an order. Its credential should identify the intended audience, one location, and read-only order-display permission. If separate screens show pickup orders and drive-through orders, add a server-validated channel or view claim rather than trusting a client-supplied filter. The server derives subscription scope from the verified credential.

Do not embed guest names, phone numbers, payment data, or internal notes in a broad event and rely on CSS to hide them. Emit a display projection containing only what the board renders, such as a public order label and state. Authorization limits who receives an event; projection limits what a compromised authorized device can reveal.

This is also where a cheap shortcut becomes expensive. One shared token across every restaurant makes provisioning easy, but revoking one stolen kiosk interrupts all of them and a location-filter bug has a much larger blast radius. Per-device credentials add issuance and inventory work. I would still take that trade because device-level revocation and an attributable connection log are useful operational controls, while location scope prevents a screen from subscribing to another restaurant.

Token expiry must not erase replay identity. The credential can rotate while the stable device record and location-bound cursor remain. Conversely, possession of an old cursor grants no data access; replay is authorized again on every connection.

## Operate the board as a projection you can prove

Track correctness signals, not just connection count. For each kiosk, record the last issued watermark, last acknowledged contiguous sequence, replay count, replayed event count, authentication failures, and time since the last applied event. Keep event payloads out of routine logs when the identifiers are enough to diagnose delivery.

Test with forced gaps. Deliver 700, withhold 701, deliver 702, restart the client, rotate its credential, and verify that the board requests after 700 and finishes at 702 in order. Then expire retained history and verify the snapshot path. Test a valid token for the wrong location and a read-write claim presented to the display endpoint; both must fail before subscription begins. A final test should disconnect a slow consumer and confirm that another kiosk's delivery latency is unaffected.

Deployment needs compatibility discipline. Additive event fields are easier to roll out than renamed states. Include an event schema version, keep old kiosk code able to ignore unknown fields, and deploy consumers before producers when introducing a new state. If an event version is unsupported, stop advancing the cursor and expose the stale condition. Silent skipping produces a board that appears current but is not.

The operational checklist is short in practice. Before rollout, verify that order writes and event positions cannot diverge, replay is bounded and monitored, snapshots carry a watermark, credentials are location- and audience-scoped, display payloads exclude private fields, and stale screens are obvious to staff. During rollout, watch cursor lag and snapshot fallbacks by software version. Afterward, rehearse device revocation and history-expiry recovery. Those checks provide more confidence than a dashboard showing a large number of open connections.

The durable decision is therefore not a particular realtime library. It is a contract: server-owned state, least-privilege display access, ordered events, a persisted contiguous cursor, and a defined snapshot fallback. With those pieces in place, transport and hosting can change without changing what the kitchen staff are promised.

## Further reading

- W3C, WebRTC 1.0: Real-Time Communication Between Browsers: https://www.w3.org/TR/webrtc/
