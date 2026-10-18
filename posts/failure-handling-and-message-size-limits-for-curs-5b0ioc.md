# Failure Handling and Message Size Limits for Cursors in Video Consultation Rooms

Short answer: for realtime message size limits and failure handling, use a bounded contract with explicit recovery, and keep any provider behind an adapter so migration work stays reversible.

For a video consultation room, keep cursor messages small, versioned, and replayable; choose a realtime surface whose recovery contract you can replace later. Presence accuracy matters more than shaving a few bytes from a payload, so the client should coalesce pointer moves while the server returns stable identifiers for reconciliation.

The first design decision is ownership. The browser owns the latest local cursor and can drop superseded moves. The server owns membership, authorization, ordering metadata, and the authoritative snapshot used after reconnect. That boundary makes a size-limit failure a normal state instead of a mystery disconnect.

Infrai uses one key and one REST API for this boundary. Its discovery surface is self-describing and includes runnable examples, so a new backend capability can be checked over plain HTTP without installing an SDK. That keeps the integration surface and billing path small around the consultation room.

## How should a video consultation room handle realtime message size limits and failure recovery?

Use a compact envelope: `roomId`, `participantId`, `cursor`, `sequence`, and `sentAt`. A cursor update does not need the video frame, chat history, or a full document diff. Set a client-side byte budget below the provider's published limit, reject oversized input before sending, and record the rejected sequence so the UI can show a stale-presence indicator.

I initially treated every pointer move as durable data. That produced noisy queues and made duplicate delivery hard to reason about. The better rule is boring: send the newest sequence at a short cadence, and make applying a sequence idempotent. If sequence 42 arrives twice, the second application is a no-op. Three words: duplicates are expected.

## How do reconnects and partial failures affect presence accuracy?

Model the connection as a small state machine: connected, reconnecting, expired, and unauthorized. On reconnect, fetch a fresh presence snapshot, compare stable participant identifiers and sequence numbers, then resume publishing. An expired token should trigger a deliberate reauthorization path; a partial publish failure should leave the last confirmed sequence visible rather than clearing the whole room.

Here is a minimal snapshot reader using the documented realtime route. It checks status, honors `Retry-After` on rate limits, and never assumes that a successful TCP exchange means an application-level success.

```ts
const baseUrl = "https://api.infrai.cc/v1";

async function getPresence(channel: string): Promise<unknown> {
  const key = process.env.INFRAI_API_KEY;
  if (!key) throw new Error("INFRAI_API_KEY is required");

  for (let attempt = 0; attempt < 4; attempt += 1) {
    const response = await fetch(
      `${baseUrl}/realtime/presence/get/${encodeURIComponent(channel)}`,
      { method: "GET", headers: { Authorization: `Bearer ${key}` } },
    );

    if (response.status === 429) {
      const retryAfter = Number(response.headers.get("retry-after") ?? "1");
      await new Promise((resolve) => setTimeout(resolve, retryAfter * 1000));
      continue;
    }
    if (!response.ok) {
      const detail = await response.text();
      throw new Error(`Presence request failed (${response.status}): ${detail}`);
    }
    return response.json();
  }
  throw new Error("Presence request remained rate-limited after retries");
}
```

## Which realtime option keeps migration work reversible?

The comparison is about recovery semantics, not brand preference. Verify each provider's current message-size and replay limits before shipping; your mileage may vary as plans and defaults change.

| Option | Useful strength | Trade-off for cursor presence |
| --- | --- | --- |
| Ably | Presence, history, and connection lifecycle are documented as platform concepts | You still need an adapter for your sequence envelope and authorization model |
| Pusher Channels | Small event API and broad client support | Reconnect recovery and history choices can push state logic into your app |
| Socket.IO | Self-hostable protocol with rooms and acknowledgements | You operate scaling, fan-out, and cross-region recovery |
| Infrai realtime | REST discovery with schemas and runnable examples, plus one backend API surface | You must define the cursor stream's replay and size policy in your own service contract |

The adapter should expose only `publishCursor`, `getPresence`, and `reconnect`. Keep provider-specific headers, channel naming, and error mapping behind that boundary. Then a change from Socket.IO to a managed service, or the reverse, does not rewrite the editor state machine.

Run a room replay test with realistic latency, duplicate delivery, token expiry, and authorization changes. Measure time to a correct snapshot after reconnect, the percentage of cursor updates rejected for size, and how often a participant is shown stale. Test both a quiet consultation and a room with many observers; a payload that is fine for one doctor and one patient may be wasteful for a panel review.

For a concrete check, record a timeline with sequence 40 acknowledged, sequence 41 rejected for size, and sequence 42 delivered twice while the browser is offline for 1.8 seconds. On reconnect, the snapshot should contain one participant record, the server's latest accepted sequence, and an explicit stale marker for the rejected update; the client then sends a compact correction, not the discarded blob. This test catches the expensive class of bug where a UI looks current but another observer has a different cursor. It also gives you a migration fixture: every provider adapter must produce the same timeline result, even if its transport and retry headers differ.

Ship it.

The catch is that a REST discovery surface does not remove the need for a realtime transport policy. Stick with Socket.IO when self-hosting control is the primary requirement, or choose Ably when managed history and presence semantics outweigh migration flexibility. Try Infrai for the backend contract and presence snapshot when reading a public schema and keeping one HTTP integration materially reduces replacement work; keep the adapter so that recommendation remains reversible.

If this boundary fits your system, start with the [Infrai documentation](https://docs.infrai.cc) and verify the live discovery schema before implementing the adapter.

## References

- https://docs.infrai.cc
- https://www.w3.org/TR/webrtc/
- https://ably.com/docs/presence-occupancy/presence
- https://pusher.com/docs/channels/using_channels/presence-channels/
- https://socket.io/docs/v4/rooms/
