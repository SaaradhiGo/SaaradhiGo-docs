# Pilot blocker: trip completion after sustained WebSocket/GPS traffic

- **Date:** 2026-09-23
- **Trigger:** the five-ride QA rehearsal reported that `complete` stopped landing
  above roughly twenty driver-socket frames — the trip stayed `in_progress` while
  the client believed it had completed.
- **Infrastructure:** real PostgreSQL 15, real Redis, the real Channels consumers
  through `base.asgi.application`, and real QA rides over the deployed stack.
  Nothing on the critical path was mocked.
- **PII:** no coordinates, phone numbers, OTPs, JWTs, document URLs or passwords
  appear below.
- **Fix:** `fix/lifecycle-commands-under-gps-load @ c8c11de`
- **Reproduction suite:** `tests/test_lifecycle_under_gps_load.py`

## Summary

Twenty was not a threshold. The frame count in the rehearsal was a proxy for a
different variable, and the boundary that does exist is set by **how fast the
client drains its socket**, not by how many frames the driver sends.

Two things came out of this investigation, and they should be read separately:

1. **A real, previously unknown defect, deterministically reproduced and fixed.**
   Location broadcasts and lifecycle commands shared one dispatch loop with
   blocking sends, so a client that does not keep up could stop `complete`,
   `cancel` and its own GPS ingestion. Proven against real infrastructure, fixed,
   and regression-tested from 0 to 500 frames.

2. **The rehearsal's specific symptom did not reproduce.** On the current QA
   deployment, `complete` lands at 40 frames, at 500 frames, over a 90-second
   journey and over a 300-second journey, with the trip socket deliberately
   undrained — the same conditions that failed during the rehearsal. So the fix
   below is a genuine correction, but it is **not proven to be the explanation of
   what was observed on 23 September.** That residue is stated honestly in
   "What remains unexplained" and it is what holds the pilot decision at AMBER.

## 1. Root cause

**File:** `servers/consumers.py`
**Mechanism:** a blocking `send()` for high-frequency traffic on the same dispatch
loop as commands.

Channels runs one sequential loop per consumer. `AsyncConsumer.__call__` awaits
`await_many_dispatch([receive, self.channel_receive], self.dispatch)`, which feeds
**both** incoming websocket frames and channel-layer events through a single
`await dispatch(...)`. Whatever that dispatch does, everything else waits.

Every driver location frame fans out to the trip group:

```
DriverLocationConsumer.receive
  -> redis_client.add_driver_location            (Redis: heartbeat, geo, active-trip)
  -> channel_layer.group_send('admin_dashboard', ...)
  -> channel_layer.group_send(f'trip_{active_trip_id}', ...)   <-- one per ping
```

So a trip socket receives one `driver_location_update` per ping. Before the fix,
`TripStatusConsumer.driver_location_update` delivered it with a direct
`await self.send(...)`, and `self.send()` blocks while the client is not reading.

The chain that loses a completion:

1. The driver's app does not drain its trip socket (backgrounded, weak link, busy
   UI thread, or simply a bounded receive queue — the reference Python
   `websockets` client buffers **16** messages by default).
2. Unread broadcasts fill the client's buffer; the client stops reading TCP;
   backpressure reaches the server.
3. `await self.send(...)` inside `driver_location_update` blocks.
4. `await_many_dispatch` stops. The consumer dequeues nothing more.
5. `{"action": "complete"}` arrives, is accepted by the kernel and sits in the ASGI
   incoming queue. **The handler is never entered.**
6. The client's `send()` succeeded, so the app believes the ride is finished. The
   trip stays `in_progress`. Nothing is logged, because nothing failed.

**Which of the eight candidate failure modes this is:** none of them cleanly, and
the distinction matters. The command was *received* by the transport (so not "the
server never received it") and the socket was *alive* (so not "the socket was
already dead"). The handler was never entered at all — it did not start and exit
early, and no database transaction was attempted. The precise statement is: **the
consumer's dispatch loop was blocked in an outbound send, so the command was never
dequeued or routed.** It sits between "server never receives" and "handler never
starts", and calling it either one would misdescribe it.

### The second defect the same coupling caused

`DriverLocationConsumer.receive` acknowledged every frame with a blocking
`await self.send({'type': 'location_updated', ...})`. A driver app that does not
read that acknowledgement blocks `receive` once its buffer fills — so **GPS
ingestion stops entirely, mid-ride, with no error anywhere.** Measured against an
undrained client: **17 of 200 frames reached the GPS stream.** The trip's distance
evidence, which the whole metering and dispute chain rests on, silently ends.

This is arguably the worse of the two: a lost completion is visible to a driver; a
truncated GPS trail is not visible to anyone until a fare is disputed.

## 2. Reproduction

`tests/test_lifecycle_under_gps_load.py`, marked `postgres` because the contention
is over real database and Redis work that SQLite in-memory would hide.

The reproduction is **deterministic**: it fails every time before the fix and
passes every time after.

```
frames=48, client receive buffer=16, trip socket undrained
  -> trip stayed in_progress, 16 undelivered location broadcasts queued
frames=48, client receive buffer=16, trip socket DRAINED   (the control)
  -> completed
```

The control is what makes it a proof: same frame count, same buffer, same
infrastructure; the only difference is whether anyone is reading.

### The frame-count matrix, each case in its own process

Run per-case in a fresh process, because cross-test contamination inside one
pytest session is what made the first attempt at this useless.

| frames | ack received | ack latency | committed | final status |
|---|---|---|---|---|
| 0 | yes | 0.16s | 0.16s | completed |
| 1 | yes | 0.17s | 0.17s | completed |
| 5 | yes | 0.12s | 0.12s | completed |
| 10 | yes | 0.12s | 0.14s | completed |
| 19 | yes | 0.14s | 0.14s | completed |
| 20 | yes | 0.12s | 0.12s | completed |
| 21 | yes | 0.12s | 0.12s | completed |
| 25 | yes | 0.23s | 0.23s | completed |
| 50 | yes | 0.20s | 0.20s | completed |
| 100 | yes | 0.11s | 0.11s | completed |
| 250 | yes | 0.23s | 0.23s | completed |
| 500 | yes | 0.12s | 0.12s | completed |

**There is no boundary at 20, and no boundary anywhere in 0–500.** Latency is flat.
A driver sending frames as fast as a socket allows does not delay a lifecycle
command at all, because each consumer awaits its own database hops sequentially:
one consumer can only ever have one job in the shared executor at a time. The
`database_sync_to_async` single-thread executor — the strongest prior hypothesis —
is therefore **not** the mechanism for a single driver. It remains a real
multi-driver concern (see "Follow-ups").

**The boundary that does exist is 16**, the client's receive-queue depth, and the
rehearsal's numbers match it exactly: 16 frames completed reliably, 24 and 30 did
not.

## 3. Why the tests missed it

Four reasons, all of them structural rather than careless:

1. **No test ever opened two sockets at once.** The coupling only exists between a
   driver's location socket and a trip socket on the same ride.
2. **`WebsocketCommunicator`'s output queue is unbounded.** The test client is
   infinitely fast at draining, which no real client is. The defect is invisible by
   construction until the buffer is bounded — one line, `output_queue._maxsize`.
3. **A trip created in a test is not active in Redis.** `set_driver_active_trip` is
   called by `_accept_trip`, not by creating a `Trip`, so a directly-created trip
   makes `add_driver_location` treat the driver as free and **no fan-out happens at
   all**. My own first attempt at this reproduction measured nothing for exactly
   this reason.
4. **`receive_from(timeout=...)` cancels the application task** on timeout, in
   asgiref. Using it to poll "is anything there?" silently kills the consumer under
   test and reports a `CancelledError` that looks like a product failure. This cost
   a full cycle of false conclusions: nine matrix cases "failed" including the
   zero-frame control, which is what exposed the harness rather than the system.

Items 3 and 4 are now documented at the top of the test file, because anyone
writing the next Channels test will hit both.

## 4. The fix

`LocationBroadcastMixin` in `servers/consumers.py`. The smallest correction that
removes the coupling rather than raising a threshold:

- Location frames go onto a **depth-1 queue** drained by a dedicated task.
- A newer position **replaces** an older undelivered one — coalescing, not
  buffering, because a superseded position has no value to anyone.
- `_queue_location_frame()` never blocks and never raises, so it is safe to call
  from the dispatch loop.
- Commands and status frames keep their direct `await self.send(...)`: they are
  rare, they are ordered, and none may be dropped.
- A slow client now loses intermediate positions. That is the correct thing to
  lose.

Applied to all four consumers that forward location frames: `TripStatusConsumer`,
`RideRequestConsumer` (a rider whose app is slow must still be able to cancel),
`DriverLocationConsumer` (the per-frame acknowledgement), and
`AdminDashboardConsumer` (the fleet monitor is the heaviest location consumer on
the platform).

**No threshold was increased.** No buffer was enlarged. No fare semantics changed
and `final_fare` is untouched.

### Ordering contract, which this deliberately changes

| | Guarantee |
|---|---|
| Status/command frames among themselves | strictly ordered, and never delayed behind a location frame |
| Location frames among themselves | ordered, but **may be dropped** |
| A location frame vs a later status frame | may be reordered |

Frame integrity is preserved: each `send` is one complete ASGI message and Daphne
serialises writes per connection. Only the relative order of a location frame and
a status frame can swap, which is the point.

## 5. Regression proof

All against real PostgreSQL, real Redis and the real consumers.

| Case | Result |
|---|---|
| Matrix 0–500 frames → `complete` | lands at every count, latency flat 0.06–0.23s |
| 500 frames, undrained client → `complete` | lands in 0.13s, **exactly one** Payment |
| 500 frames, undrained client → `cancel` | lands in 0.5s |
| 500 frames → SOS | HTTP 201 in 0.39s |
| Interleaved GPS/GPS/`complete`/GPS | `complete` processed; a later frame does not resurrect the trip |
| GPS ingestion, undrained client | **17/200 before → 200/200 after** |
| Reproduction (48 frames, bounded buffer) | fails before the fix, passes after |
| Drained control at the same frame count | passes both before and after |

**On SOS:** it is raised over HTTP (`raise_sos`), not as a WebSocket action, so it
never traversed the consumer dispatch loop and was never exposed to this defect.
It is proven under the same sustained GPS load anyway, because it shares the
process and the `database_sync_to_async` thread.

Full suite: **580 passed**, ruff clean, `manage.py check` clean, bandit's single
finding is pre-existing (a `try/except/pass` around an OTP attempt cache write).

## 6. Financial idempotency

Completion sent **six times** on the same trip. The money snapshot afterwards is
byte-identical to the snapshot after the first completion:

| | After 1 `complete` | After 6 |
|---|---|---|
| Payment rows | 1 | 1 |
| TransactionHistory rows | 1 | 1 |
| WalletTransaction rows | 0 | 0 |
| Receipt rows | 0 | 0 |
| `final_fare` | NULL | NULL |
| `completed_at` | set once | **unchanged** |

The guard is `_create_payment_on_complete`'s `if trip.payments.exists(): return`,
plus the strict transition table refusing `completed → completed` under
`select_for_update`.

This matters more after the fix than before it, and the report should say so
plainly: **the fix makes commands that were previously swallowed all arrive.** A
driver app that retried a completion it believed had failed will now land every
attempt. The idempotency above is what makes that safe.

One honest limitation, the same one the five-ride evidence carries: these are
unpaid cash trips, so no settlement, wallet credit, commission or receipt ran at
all. "Settlement unchanged" is proven only in the trivial sense. Proving a
repeated completion cannot disturb a **settled** trip needs a QA ride that
completes payment, and that is the next rehearsal.

## 7. Performance, before and after

Per GPS frame, measured from the servers' own counters (Redis
`total_commands_processed`, `pg_stat_database.xact_commit`, broker queue length) —
not estimated:

| | Before | After |
|---|---|---|
| Frames ingested (200 sent, undrained client) | **17 (8.5%)** | **200 (100%)** |
| Mean frame processing latency | 219 ms | **24 ms** |
| PostgreSQL transactions per frame | 1.47 | **0.16** |
| Redis commands per frame, end to end | ~34 | ~29 |
| Celery enqueues per frame | 0 | **0** |
| `complete` latency, 0 → 500 queued frames | n/a (lost) | 0.11 → 0.17s, flat |

The "before" latency and transaction figures are inflated by the stall itself, so
the honest reading is the first row: before the fix, ingestion **stopped**.

The Redis figure is the whole instance, including the channel layer's own
group-send bookkeeping; `add_driver_location` itself accounts for about five of
them. **No N+1 and no unbounded growth**: `_get_driver_broadcast_info` touches
`self.user.driver` and `driver.active_vehicle`, which Django caches on the
instance, which is why transactions per frame are well under one. Nothing in the
GPS receive path enqueues Celery work — the trail is drained by a Beat task
reading the Redis stream.

## 8. QA proof

Real rides on the deployed QA stack: book → accept → OTP via the rider → reached →
start → N GPS frames → `complete`, verified by reading the trip back from the API
rather than trusting a WebSocket frame. Harness: `qa_blocker.py`, whose single
variable is whether the trip socket is drained.

### Against the pre-fix deployment

| Frames | Interval | Journey | Trip socket | Fan-out seen | Result |
|---|---|---|---|---|---|
| 40 | 0.5s | 20s | undrained | — | **completed in 0.3s** |
| 40 | 0.3s | 12s | drained | **40 of 40** | completed in 3.5s |
| 500 | 0.05s | 27s | undrained | — | **completed in 0.2s** |
| 30 | 3s | 90s | undrained | — | **completed in 0.3s** |
| 30 | 10s | 300s | undrained | — | *see below* |

The drained run confirms the fan-out is real in QA: the trip socket received
exactly one location frame per ping.

**The rehearsal's symptom did not reproduce.** The explanation for the difference
between QA and the local reproduction is buffer depth in bytes rather than
messages: each fan-out frame is about eighty bytes, so the client's 16-message
queue plus its TCP receive buffer plus Railway's edge proxy buffer plus the
server's send buffer absorb several hundred tiny frames before Daphne's write can
block. The local reproduction bounds only the client queue, which is why it is
deterministic there and not here.

This does not make the defect theoretical — it makes the **threshold
environment-dependent**, which is worse, not better. A larger fan-out payload, a
slower client, a mobile link with a small window, or a longer stall all move the
boundary down, and none of those are under our control. Removing the coupling is
the only stable answer, which is what the fix does.

## 9. Lifecycle safety

After the fix, ordinary location traffic cannot prevent any lifecycle command from
being processed, and this is asserted rather than argued:

| Command | Proven under 500 frames + an undrained client |
|---|---|
| `complete` | yes, exactly once |
| `cancel` | yes |
| `reached` / `start` | exercised in every QA ride before the journey |
| SOS | yes, over its real HTTP path |

## 10. Pilot decision

**AMBER — not GREEN, and not RED.**

The gate was: *GREEN only if a realistic long-running ride can process lifecycle
commands reliably after sustained GPS traffic.*

What is satisfied:

- A realistic long ride's worth of GPS traffic (500 frames) no longer delays or
  loses `complete`, `cancel` or SOS, proven locally against real infrastructure
  and in QA on the deployed stack.
- The architectural coupling that could starve them is removed, not thresholded.
- GPS ingestion no longer stops against a slow client — a defect that would have
  quietly destroyed billing evidence on real rides.
- Repeated completion is financially idempotent and `final_fare` is untouched.

Why it is not GREEN:

- **The rehearsal's specific failure is still unexplained.** Two trips were
  observed `in_progress` after a reported completion on 23 September, and I cannot
  reproduce that on the current deployment at the same parameters. A defect that
  was real, is not reproducible, and has a plausible-but-unproven explanation is
  not a closed defect. Calling this GREEN would be asserting a root cause I have
  not established.
- The fix is on a branch and has not been deployed to QA or re-verified there
  post-deploy at the time of writing.

### What remains unexplained

Candidates ruled out by measurement, not by argument:

| Hypothesis | Status |
|---|---|
| A frame/message threshold near 20 | **ruled out** — 0–500 flat, locally and in QA |
| The `database_sync_to_async` single-thread executor starving lifecycle work | **ruled out for one driver** — a consumer awaits its hops sequentially, so at most one job is queued |
| Channel-layer capacity (100) dropping messages | **ruled out** — no "over capacity" log at any frame count; `group_send` drops rather than blocks, which cannot affect an inbound command |
| A Railway deploy killing the sockets mid-rehearsal | **ruled out** — last QA deploy 18:45 UTC, the rehearsal ran ≈19:38–20:07 UTC |
| Journey duration / an idle-socket timeout | **not supported** — 90s and 300s journeys both complete, and the trip socket is never idle inbound because of the fan-out |
| Client-drain backpressure (this fix) | **proven to exist**; not proven to be what was seen in QA |

The most likely remaining explanation is that the rehearsal hit the same
backpressure mechanism under conditions whose buffer state I have not reproduced —
the rehearsal harness held additional sockets, sent approach pings, and ran several
scenarios back to back over a longer-lived process. That is a hypothesis, and it is
labelled as one.

### To reach GREEN

1. Merge and deploy `fix/lifecycle-commands-under-gps-load`, then re-run the QA
   matrix post-deploy.
2. Re-run the **full original five-scenario rehearsal** at its original
   parameters (B at 30 frames/interval 10, D at 12 points doubled) and confirm all
   six trips complete. That is the direct answer to the open question.
3. Add server-side observability for this class of failure, which is the real gap:
   nothing logged when a dispatch loop stalled. A counter on
   `location_frames_coalesced` (now emitted at disconnect) plus a warning when a
   command waits longer than a second would have made the original diagnosis
   minutes rather than a day.

## Follow-ups

- **`database_sync_to_async` is process-wide single-threaded.** Not the cause here,
  but with N drivers each sending a frame every 2.5s, every location frame and
  every lifecycle command in the process queue through **one thread**. At 100
  concurrent drivers that is ~80 submissions/second on one thread. This needs a
  load test before scale, and it is a stronger argument for extracting the
  lifecycle service out of the consumer than any of the tidiness arguments.
- **`servers/redis_client.py` logs raw coordinates at DEBUG** (`Driver %s location
  updated ... lng=%s lat=%s`). Inert in production, where `LOG_LEVEL` is `WARNING`,
  but it is a latent location-privacy leak one environment variable away. Worth
  removing on principle.
- **The five-ride evidence document's "completion silently failed above roughly 20
  driver-socket frames" should be amended** to record that twenty was not a
  threshold and the boundary was never frame count.
