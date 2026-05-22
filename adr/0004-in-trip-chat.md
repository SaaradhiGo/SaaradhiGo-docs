# ADR-0004: In-trip chat over WebSocket + persisted history

* Status: Accepted
* Date: 2026-05-22

## Context

Riders and drivers need to coordinate during a trip (pickup confirmation, "I'm at the gate", "I see your car"). Today the only channel is the OS dialer launched via `tel:` deep link — fine while we work on a phone-masking proxy, but it doesn't capture text, doesn't help when calls fail on poor connectivity, and leaves no record for a lost-item or safety dispute.

## Decision

Add a per-trip chat channel:

* **WebSocket**: `ws/ride/trip/<id>/chat/` joins a Django Channels group (`trip_chat_<id>`). Both rider and driver authenticate via the existing JWT WS middleware. Send `{"action": "send", "body": "..."}`; receive `{"type": "message", "id", "sender_role", "body", "created_at", "is_system"}`.
* **REST history**: `GET /api/v1/ride/trip/<id>/chat/` (paginated) for backfill on screen-open and for support reading historical disputes. Marking the caller's unread peer messages as read is a side-effect of the fetch.
* **Persistence**: every message lands in `ChatMessage` with `sender_role` denormalised (so reads don't re-resolve user → role) and an `is_system` flag for server-emitted lines.
* **Lifetime**: chat closes for new messages once the trip is `completed` or `cancelled`; the WS consumer refuses to persist after that. History stays readable indefinitely.

System messages emitted by other services (e.g. "Driver arrived at pickup") write into the same stream with `is_system=True`. The mobile UI lanes them as inline system-italics, not as a bubble.

## Consequences

### Positive
* Coordination works without exposing phone numbers.
* Dispute investigations (lost-item, fare, safety) get a stored conversation.
* A future support-side chat console can read + reply on the same thread by writing a `sender_role='support'` row.

### Negative
* Two more moving parts: a Channels consumer + a REST endpoint, both authenticated against the existing Trip participant check.
* Storage grows linearly with completed trips; cleanup policy deferred to Phase-1 (retain at least 90 days for safety/legal review).

### Neutral
* No E2E encryption. Messages are encrypted in transit (WSS / HTTPS) and at rest in Postgres but readable by ops + support. That's the right posture for a regulated transport platform — investigators need access — and is consistent with how Uber / Ola handle it. If future regulation tightens, add per-trip key escrow.

## Alternatives considered

1. **Twilio Conversations or Sendbird** — managed chat, ~$0.05–0.10 per active user per month. Rejected for Phase-0 cost; the in-house implementation is ~250 lines and a clean Phase-1 cutover.
2. **No chat; rely on phone-masking proxy only** — punts on text-only riders / poor-network situations and loses dispute records.
3. **Push-notification based "quick reply"** — fragile UX; ride is a moving real-time context, not an email thread.

## Implementation

Backend: `servers/ride/chat_consumer.py` + `ChatMessage` model + `trip_chat_history` REST view. Routing wired in `servers/routing.py`.

Mobile: `lib/services/chat_service.dart` (WS + REST), `lib/screens/home/trip_chat_screen.dart`. Triggered from the chat icon on the ride-in-progress card.

## Follow-ups

* Read-receipt UI on each bubble.
* Quick-reply chips ("On my way", "Reached the gate", "5 mins").
* Push notification when a peer message arrives while the app is backgrounded.
* Support-side reply surface in the ops console.
