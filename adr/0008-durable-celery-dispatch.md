# ADR-0008: Dispatch is owned by Celery, not by the rider's WebSocket

* Status: Accepted
* Date: 2026-09-22
* Deciders: Architect, Founder
* Supersedes: none
* Superseded by: none

## Context

Driver search ran as an `asyncio.create_task` inside `RideRequestConsumer`,
and `disconnect()` cancelled that task:

```python
async def disconnect(self, close_code):
    task = getattr(self, '_dispatch_task', None)
    if task is not None and not task.done():
        task.cancel()
```

The wave loop itself slept between waves in the consumer's event loop
(`await asyncio.sleep(20)`), so the 20-second gaps between the 1500 m, 3000 m
and 5000 m waves only existed for as long as the socket did.

Three consequences, all of them user-visible:

1. **A rider whose phone dropped connection mid-search lost the search.**
   Waves 2 and 3 never fired. `auto_cancel_trip` then closed the trip with
   `cancellation_reason = 'no_driver_accepted'` — a claim that was simply
   untrue, because most drivers were never asked. On Indian mobile networks a
   drop inside a 40–60 second search window is routine, so this was a
   self-inflicted "no cars available" rate.
2. **Any deploy or Daphne restart killed every in-flight search.**
3. **Dispatch could not scale.** It was pinned to whichever process held the
   socket, while ADR-0007 had already established that long-running work
   belongs in Celery. Dispatch was the one workflow that most needed it and
   the one that had been left behind.

Worth recording what was *already* durable, because it made the fix small:
the `Trip` row, the `auto_cancel_trip` deadline, and a Postgres-backed
"is this trip still open?" check. Only wave scheduling and execution were
ephemeral.

## Decision

**Celery is the sole owner of dispatch wave execution.** The WebSocket may
start a search and observe its progress; it never runs the loop, and its
lifetime has no bearing on the outcome.

`ride.dispatch_wave(trip_id, wave_index, dispatch_epoch, radii)` executes one
wave and schedules its successor with Celery's `countdown`. A worker is never
held sleeping. Business behaviour is unchanged: the same expanding radii, the
same gap, the same per-vehicle-type geo isolation, the same 90-second accept
timeout.

We explicitly rejected keeping the asyncio path as an enqueue-failure
fallback. Two dispatch engines would mean two sets of race conditions and a
bug reproducible in only one of them. If enqueueing fails we log it, tell the
rider if they are connected, and let the durable timeout resolve the trip.

### The two kinds of idempotency

Celery gives at-least-once delivery, so these are kept separate on purpose:

* **Business idempotency — mandatory, PostgreSQL-backed.** A wave re-reads
  the `Trip` row on every execution and **performs no PostgreSQL writes at
  all**. It reads, then sends offers. A duplicate delivery, a retry after a
  worker crash, or a wave left over from an abandoned search therefore either
  finds the trip ineligible and no-ops, or re-sends an offer. It cannot
  assign a driver twice, reopen a cancelled trip, extend the search, or touch
  money.
* **Notification de-duplication — best effort, Redis-backed.** Not offering
  the same driver the same ride twice within one search. If Redis is flushed,
  a driver may see a duplicate card. That is cosmetic; their acceptance is
  still gated by the row-locked checks in `_accept_trip`.

Because a wave never writes, correctness does not depend on revoking Celery
tasks. A stale task is harmless if it runs.

### The dispatch generation ("epoch") is deliberately not persisted

`dispatch_epoch` is a task argument only. It scopes the notification
de-duplication set so a rider-triggered retry re-reaches drivers who ignored
the first search — which is what the old code did implicitly, since each call
began with an empty in-process `offered` set.

We considered a `Trip.dispatch_generation` column so PostgreSQL could
distinguish generations, and rejected it, because **no lifecycle decision
reads the epoch.** The invariant is:

> A trip is dispatchable if and only if `Trip.driver_id IS NULL` **and**
> `Trip.status_id.status_code IN (NULL, 'requested')`. Nothing else — not the
> epoch, not any Redis key — participates in that decision.

Given that, a wave from an abandoned generation can only re-offer a trip that
genuinely is still searching. It cannot extend the search either: the durable
deadline is `auto_cancel_trip`, scheduled exactly once at trip creation and
never rescheduled by a wave or a retry. A column that nothing consults would
be schema pretending to be a safeguard.

If a future change *does* make a lifecycle decision depend on which
generation a task belongs to — for example per-generation driver scoring, or
allowing a retry to extend the deadline — that invariant breaks and the
column becomes justified. This ADR should be revisited at that point.

### Offer dismissal is now one mechanism

`dismiss_outstanding_offers(trip_id, reason, exclude_driver_id=None)` pops the
Redis offer set and sends the existing `trip_taken` event to each candidate.
It is idempotent (the pop is destructive), safe when Redis is unavailable or
the key has expired, and never touches the durable lifecycle.

Every terminal transition now uses it: driver acceptance, WebSocket
cancellation, REST rider cancellation, REST driver cancellation, and the
timeout. Previously only acceptance and the timeout cleaned up, so **a rider
who cancelled over REST left every candidate driver holding a live request
card** and left the offer set behind in Redis. UI cleanup is secondary
defence; PostgreSQL remains authoritative.

### Rider reconnect

Dispatch now outlives the socket, so a rider can miss the real-time frame
announcing acceptance. `RideRequestConsumer.connect()` therefore replays a
`current_trip` snapshot built from `TripDetailSerializer` — reused rather than
reinvented, so its existing rule that only the trip's own rider ever sees
`otp` continues to hold, and the driver's phone number is not part of that
representation. The query is scoped to `user_id=<connecting rider>`, so one
rider can never be handed another's trip.

### `auto_cancel_trip` hardening

The timeout read the trip without a lock and then called a bare
`trip.save()`, which writes every column from the in-memory instance. An
acceptance committing between that read and that save was silently reverted —
the trip was cancelled and `driver_id` reset to NULL while a driver was
already driving to the pickup. Both the timeout and acceptance land at ~90
seconds, so this was reachable.

The row is now locked with `select_for_update`, the driver and status are
re-checked **inside** the transaction, and only the four cancellation columns
are written via `update_fields`.

## Consequences

**Good**

* A rider losing connectivity no longer loses their driver search. This is
  the point of the change.
* Dispatch survives deploys and restarts and is no longer pinned to one
  process.
* No migration. Dispatch state is derived from columns that already existed.
* One offer-dismissal path instead of three partial ones.
* The per-driver `Driver.objects.get()` inside the wave loop is gone,
  replaced by a single bulk query. FCM was already asynchronous via
  `auth_user.send_push_notification_task`, so no notification work was
  needed.

**Costs and risks**

* Dispatch latency now includes broker round-trip time. At Phase-0 volumes
  this is immaterial next to the 20-second wave gap.
* Dispatch correctness now depends on the Celery worker being alive. A dead
  worker means no searches start — previously it meant no receipts or pushes.
  The worker is a single Railway service with `-B`, so it is a single point of
  failure that now sits on the critical booking path. Worth a second replica
  with beat split out before scale-up.
* **Deploy order matters.** The new backend enqueues `ride.dispatch_wave`; an
  old worker does not know that name and will reject it. Deploy the **worker
  first**, then the backend. In the reverse order, searches do not start
  until the worker catches up — degraded, not corrupt, and bounded by the
  durable timeout. There are no stale dispatch tasks to drain, because
  dispatch was never a task before this change, and the Redis key formats are
  unchanged and backward compatible.

## References

* `servers/ride/dispatch.py` — the state machine and the invariant above
* `servers/ride/tasks.py` — `ride.dispatch_wave`, hardened `auto_cancel_trip`
* `tests/test_durable_dispatch.py` — 36 tests, including the stale-generation
  and acceptance-vs-timeout cases
* [ADR-0007](0007-launch-hardening-batch.md) — established the
  "no network I/O under a row lock, side effects after commit" rule this
  follows
