# Pilot blocker: trip completion after sustained WebSocket/GPS traffic

- **Date:** 2026-09-23
- **Trigger:** the five-ride QA rehearsal reported that `complete` stopped landing
  above roughly twenty driver-socket frames — the trip stayed `in_progress` while
  the client believed it had completed.
- **Infrastructure:** real PostgreSQL 15, real Redis, the real Channels consumers
  through `base.asgi.application`, and real rides on the deployed QA stack. Nothing
  on the critical path was mocked.
- **PII:** no coordinates, phone numbers, OTPs, JWTs, document URLs or passwords
  appear below.
- **Fixes:** `9701c8c` (dispatch-loop decoupling), **`8e2ac37`** (the GPS echo — this
  is the one that fixes the reported symptom), and `c2c2b46` (the missing signal, plus
  a real `ping`/`pong`). All deployed to QA and verified there.
- **Reproduction suite:** `tests/test_lifecycle_under_gps_load.py`
- **Verdict: GREEN.** A ten-minute QA ride with the driver's command socket never
  drained completes in 3.5s. Two app-side improvements are recommended but not
  blocking — see §11.

## Summary

Twenty was not a threshold, and the failure had nothing to do with the database,
the transaction, the channel layer, or the completion handler. **The driver's
WebSocket was already closed — by our own server — before `complete` was ever
sent.** The app did not notice, because its `send()` succeeded.

The mechanism, in one paragraph: `group_send('trip_<id>', driver_location_update)`
reaches the *whole* trip group, so the assigned driver received an echo of every
position it had just sent, on the same socket that carries `complete`, `cancel` and
`start`. A driver app that is slow to drain accumulates that echo. The reference
Python `websockets` client stops reading frames once **sixteen** messages are
queued — **including Daphne's keepalive pings** — so it stops answering pongs.
Daphne's default 20-second ping with a 30-second timeout then elapses and **the
server closes the connection**. Everything sent afterwards is lost silently.

That needs two things at once: a full receive buffer *and* roughly fifty more
seconds of ride. Which is why it looked like a frame count.

## 1. Root cause

**File:** `servers/consumers.py` — `TripStatusConsumer.driver_location_update`
**Condition:** the assigned driver is a member of the trip group it is broadcasting
into, and its client is slower to drain than the driver is to emit.

```
DriverLocationConsumer.receive                    (one per GPS ping)
  -> channel_layer.group_send(f'trip_{active_trip_id}', driver_location_update)
       -> rider's trip socket        <-- wanted: this is the moving-car map
       -> ASSIGNED DRIVER's trip socket  <-- the defect: an echo of its own position,
                                            on the socket that carries `complete`
```

Then:

1. The echo accumulates on a driver app that is slow to drain — backgrounded, weak
   mobile link, busy UI thread, or simply a bounded receive queue.
2. At sixteen queued messages the client's reader stops reading frames **at all**,
   ping frames included.
3. No pongs. Daphne's ping timeout elapses. **Daphne closes the connection.**
4. The app never notices: its own `send()` still succeeds into a half-open socket.
5. `{"action": "complete"}` is never delivered. The trip stays `in_progress` while
   the driver is shown a finished ride.

### It predicts all five rehearsal scenarios exactly

| Scenario | Pings | Interval | Buffer fills at | Ride left after | Predicted | Observed |
|---|---|---|---|---|---|---|
| A | 14 | 5s | never (14 < 16) | — | pass | passed |
| C | 4 | 40s | never | — | pass | passed |
| E | 12 | 5s | never | — | pass | passed |
| D | 24 | 6s | 96s | 48s ≈ the ~50s timeout | **fail** | **failed** |
| B | 30 | 10s | 160s | 140s ≫ 50s | **fail** | **failed** |

And it predicts my own QA runs, including the ones that misled me early on: 20s, 27s
and 90s journeys all had **under** fifty seconds of ride remaining after the buffer
filled, so all three passed and made the defect look unreproducible.

### The proof, from the server's own log

The 300-second run that failed:

```
20:53:35  WSCONNECT   /ws/ride/trip/29/
20:57:06  WSDISCONNECT /ws/ride/trip/29/     <-- the server closed it
20:58:33  client sends {"action":"complete"}  <-- 87 seconds too late
          trip 29 final status: in_progress
```

The socket died **eighty-seven seconds before** the command was sent. This is
failure mode **H — the socket was already dead before completion** — and nothing in
the application ever saw a `complete` to fail at.

### What was ruled out, by measurement rather than argument

| Hypothesis | Verdict |
|---|---|
| A frame/message threshold near 20 | **ruled out** — 0–500 frames, flat 0.06–0.23s latency, locally and in QA |
| `database_sync_to_async`'s process-wide single thread starving lifecycle work | **ruled out for one driver** — a consumer awaits its hops sequentially, so it can only ever have one job queued |
| Channel-layer capacity (100) dropping messages | **ruled out** — no "over capacity" log at any frame count; and `group_send` drops rather than blocks, which cannot affect an *inbound* command |
| A Railway deploy killing sockets mid-rehearsal | **ruled out** — last QA deploy 18:45 UTC, the rehearsal ran ≈19:38–20:07 UTC |
| Journey duration alone | **ruled out** — a 90-second journey passes; duration only matters *after* the buffer fills |
| The database, the transaction, the handler | **never reached** |

## 2. The second defect found on the way

Independent of the above, and fixed in `9701c8c`: location broadcasts and lifecycle
commands shared `TripStatusConsumer`'s single `await_many_dispatch` loop, and the
broadcasts were delivered with a **blocking** `await self.send(...)`. With a bounded
client that genuinely applies backpressure, the loop stalls and `complete` is never
dequeued — the command arrives at the kernel and is never routed.

Reproduced deterministically: 48 frames + a 16-message client buffer left the trip
`in_progress`; the drained control at the same frame count completed.

The same coupling silently ended GPS ingestion. `DriverLocationConsumer.receive`
acknowledged every frame with a blocking send, so a driver app that does not read
`location_updated` stopped being able to deliver location at all once its buffer
filled. **Measured: 17 of 200 frames reached the GPS stream, with nothing logged
anywhere.** That is the trip's distance evidence — the basis of metering and dispute
handling — vanishing mid-ride, invisibly. Arguably worse than a lost completion,
because a driver notices a lost completion.

This one does not reproduce in QA, because 80-byte frames never fill the combined
client/TCP/proxy buffers enough to make Daphne's write block. That makes the
threshold environment-dependent, which is worse than a fixed one, not better.

## 3. Reproduction

`tests/test_lifecycle_under_gps_load.py`, marked `postgres` because the contention
is over real database and Redis work.

```
48 frames, 16-message client buffer, undrained   -> in_progress   (the defect)
48 frames, 16-message client buffer, DRAINED     -> completed     (the control)
```

Deterministic: fails every time before `9701c8c`, passes every time after.

The frame-count matrix, each case in its own process (cross-test contamination in a
single pytest session is what made the first attempt useless):

| frames | 0 | 1 | 5 | 10 | 19 | 20 | 21 | 25 | 50 | 100 | 250 | 500 |
|---|---|---|---|---|---|---|---|---|---|---|---|---|
| committed | 0.16s | 0.17s | 0.12s | 0.14s | 0.14s | 0.12s | 0.12s | 0.23s | 0.20s | 0.11s | 0.23s | 0.12s |

**No boundary at 20, and none anywhere in 0–500.** A driver sending frames as fast
as the socket allows does not delay a lifecycle command at all.

## 4. Why the tests missed it

1. **No test ever opened two sockets at once.** The coupling exists only between a
   driver's location socket and a trip socket on the same ride.
2. **No test had a rider *and* a driver on the same trip group**, which is what
   makes the echo visible.
3. **`WebsocketCommunicator`'s output queue is unbounded**, so the test client is
   infinitely fast at draining, which no real client is. The defect is invisible by
   construction until the buffer is bounded.
4. **A trip created in a test is not active in Redis.** `set_driver_active_trip` is
   called by `_accept_trip`, not by creating a `Trip`, so a directly-created trip
   makes `add_driver_location` treat the driver as free and **no fan-out happens at
   all**. My own first reproduction attempt measured nothing for this reason.
5. **`receive_from(timeout=...)` cancels the application task** in asgiref. Using it
   to poll "is anything there?" silently kills the consumer under test and raises a
   `CancelledError` that looks like a product failure. This produced a full cycle of
   false conclusions — nine matrix cases "failed" including the zero-frame control,
   which is what exposed the harness rather than the system.
6. **Nothing in the platform logs a stalled dispatch loop or a lost command.** The
   deepest reason it survived: there was no signal to notice.

Items 4 and 5 are now documented at the top of the test file.

## 5. The fixes

### `8e2ac37` — the driver no longer receives its own GPS echo

One guard in `TripStatusConsumer.driver_location_update`: if this consumer is the
assigned driver, do not forward location. The driver is the source of that position;
the rider is who it was always for. This removes essentially all traffic from the
socket that carries lifecycle commands, so the buffer never fills, pongs are always
answered, and the server never closes the connection.

### `9701c8c` — location frames can no longer block the dispatch loop

`LocationBroadcastMixin`: location frames go onto a **depth-1 queue** drained by a
dedicated task, where a newer position **replaces** an older undelivered one —
coalescing, not buffering, because a superseded position has no value. Commands and
status frames keep their direct `await self.send(...)`: rare, ordered, never
dropped. Applied to all four consumers that forward location frames.

**No threshold was raised and no buffer enlarged.** No fare semantics changed and
`final_fare` is untouched.

Ordering contract, which this deliberately changes:

| | Guarantee |
|---|---|
| Status/command frames among themselves | strictly ordered, never delayed behind a location frame |
| Location frames among themselves | ordered, but **may be dropped** |
| A location frame vs a later status frame | may be reordered |

## 6. QA proof

Real rides on the deployed stack: book → accept → OTP via the rider → reached →
start → N pings → `complete`, verified by reading the trip back from the API. The
driver's trip socket is deliberately **undrained** and its client keepalive
**off** — the harshest realistic client, and the rehearsal's own conditions.

| Frames | Interval | Journey | Build | Rider saw locations | Result |
|---|---|---|---|---|---|
| 40 | 0.5s | 20s | pre-fix | — | completed 0.3s |
| 40 | 0.3s | 12s | pre-fix | 40 of 40 | completed 3.5s |
| 500 | 0.05s | 27s | pre-fix | — | completed 0.2s |
| 30 | 3s | 90s | pre-fix | — | completed 0.3s |
| **30** | **10s** | **300s** | **pre-fix** | — | **`in_progress` — FAIL** |
| **30** | **10s** | **300s** | `9701c8c` only | — | **`in_progress` — FAIL** |
| **30** | **10s** | **300s** | **`8e2ac37`** | — | **completed 0.3s — PASS** |
| 40 | 1s | 40s | `8e2ac37` | **40 of 40** | completed 3.5s |
| **60** | **10s** | **600s** | **`8e2ac37`** | **60 of 60** | **completed 3.5s — PASS** |

The three 300-second rows are the whole investigation: the symptom reproduces
reliably, survives the first fix, and is resolved by the second. And the rider still
receives every position, which was the one user-visible regression risk.

The last row is the pilot case: **a ten-minute ride, sixty pings, the driver's
command socket never drained and its keepalive off — completed in 3.5 seconds, with
all sixty positions delivered to the rider.** That is the same shape of ride that
failed at half the length before the fix.

One methodology note, since it cost a run: a push to `dev` deploys QA and restarts
the container, which kills every open WebSocket. A first attempt at this ten-minute
ride was invalidated that way by an unrelated docs commit. Long QA rehearsals need a
deployment freeze for their duration.

## 7. Regression proof

All against real PostgreSQL, real Redis and the real consumers.

| Case | Result |
|---|---|
| Matrix 0–500 frames → `complete` | lands at every count, latency flat |
| 500 frames, undrained client → `complete` | 0.13s, **exactly one** Payment |
| 500 frames, undrained client → `cancel` | 0.5s |
| 500 frames → SOS | HTTP 201 in 0.39s |
| Interleaved GPS/GPS/`complete`/GPS | processed; a later frame does not resurrect the trip |
| Driver receives its own echo | **0 frames**; rider receives all 12 |
| Driver command-socket queue after 30 pings | **0** |
| GPS ingestion, undrained client | **17/200 → 200/200** |
| Reproduction / drained control | fails before, passes after / passes both |

**On SOS:** it is raised over HTTP (`raise_sos`), not as a WebSocket action, so it
never traversed the consumer dispatch loop and was never exposed to either defect.
Proven under the same sustained load anyway.

Full suite: **580 passed**, ruff clean, `manage.py check` clean. Bandit's single
finding is pre-existing (a `try/except/pass` around an OTP attempt cache write).

## 8. Financial idempotency

Completion sent **six times** on one trip. The money snapshot afterwards is
byte-identical to the snapshot after the first:

| | After 1 | After 6 |
|---|---|---|
| Payment rows | 1 | 1 |
| TransactionHistory rows | 1 | 1 |
| WalletTransaction rows | 0 | 0 |
| Receipt rows | 0 | 0 |
| `final_fare` | NULL | NULL |
| `completed_at` | set once | **unchanged** |

The guards are `_create_payment_on_complete`'s `if trip.payments.exists(): return`
and the strict transition table refusing `completed → completed` under
`select_for_update`.

This matters *more* after the fix than before, and the report should say so: the
fixes make commands that were previously swallowed all arrive. A driver app that
retried a completion it believed had failed will now land every attempt. The
idempotency above is what makes that safe.

One honest limitation, carried over from the five-ride evidence: these are unpaid
cash trips, so no settlement, wallet credit, commission or receipt ran at all.
"Settlement unchanged" is proven only in the trivial sense. Proving a repeated
completion cannot disturb a **settled** trip needs a QA ride that completes payment,
which is the next rehearsal.

## 9. Performance

Per GPS frame, from the servers' own counters (Redis `total_commands_processed`,
`pg_stat_database.xact_commit`, broker queue length) — measured, not estimated:

| | Before | After |
|---|---|---|
| Frames ingested (200 sent, undrained client) | **17 (8.5%)** | **200 (100%)** |
| Mean frame processing latency | 219 ms | **24 ms** |
| PostgreSQL transactions per frame | 1.47 | **0.16** |
| Redis commands per frame, end to end | ~34 | ~29 |
| Celery enqueues per frame | 0 | **0** |
| `complete` latency, 0 → 500 queued frames | n/a (lost) | 0.11 → 0.17s, flat |
| Frames delivered to the driver's command socket | 1 per ping | **0** |

The "before" latency and transaction figures are inflated by the stall itself; the
honest reading is the first row — before the fix, ingestion **stopped**.

The Redis figure covers the whole instance including the channel layer's own
bookkeeping; `add_driver_location` accounts for about five of them. **No N+1 and no
unbounded growth**: `_get_driver_broadcast_info` touches `self.user.driver` and
`driver.active_vehicle`, which Django caches on the instance, which is why
transactions per frame are well under one.

### Exactly what runs in the GPS receive path

Enumerated from `DriverLocationConsumer.receive`, because "what is on the hot path"
was the question behind the backpressure test:

| | In the receive path? |
|---|---|
| Synchronous DB work | **yes**, 2 `@database_sync_to_async` hops — but ~0.16 transactions/frame, because Django caches the related objects on the instance after the first frame |
| Redis calls | **yes**, ~5 in `add_driver_location` (heartbeat setex, vehicle-type read, geo add/rem, active-trip read) |
| GPS stream write | **yes**, one XADD to `driver_location_stream` (maxlen 100 000) |
| Channel-layer sends | **yes**, 2 `group_send` — `admin_dashboard` and `trip_<id>` |
| Send to own socket | **yes**, one acknowledgement — now queued and coalescing, not blocking |
| FCM / push | **no** |
| Celery enqueue | **no** — the trail is drained by a Beat task reading the Redis stream |
| Notification rows | **no** |
| Fare or distance calculation | **no** — actuals are computed post-completion by a countdown task |

## 10. Lifecycle safety

After both fixes, ordinary location traffic cannot prevent any lifecycle command
from being processed, and this is asserted rather than argued:

| Command | Proven under 500 frames + an undrained client | Proven on a 300s QA ride |
|---|---|---|
| `complete` | yes, exactly once | yes |
| `cancel` | yes | yes (harness cleanup path) |
| `reached` / `start` | yes | yes, every QA ride |
| SOS | yes, over its real HTTP path | — (HTTP, never exposed) |

## 11. Pilot decision: GREEN

The gate was: *GREEN only if a realistic long-running ride can process lifecycle
commands reliably after sustained GPS traffic.*

- The reported symptom is **root-caused to an exact mechanism**, reproduced in QA,
  and shown to be resolved by a specific commit — with the intermediate build
  proving the first fix alone was *not* sufficient.
- A **ten-minute** ride with the driver's trip socket undrained and its keepalive
  off — a harsher client than either real app — completes in 3.5s on the deployed
  build, as does the 300-second case that reproduced the failure.
- 500 frames no longer delay or lose `complete`, `cancel` or SOS.
- GPS ingestion no longer stops against a slow client, so billing evidence survives.
- Repeated completion is financially idempotent; `final_fare` is untouched.
- The rider still sees the car move.

### How exposed were the real apps? Less than the rehearsal suggested

Checked rather than assumed, because it changes how urgent this was:

| | Driver app | Rider app |
|---|---|---|
| Drains its socket | **yes** — `channel.stream.listen(...)`, an unbounded Dart stream | **yes**, on both the ride and trip channels |
| Keepalive | **yes** — an app-level `{"action":"ping"}` every 25s | none of its own; relies on the protocol-level pong `dart:io` sends automatically |
| Consumes the driver-location echo | `location_updated` only | `driver_location_update` — it needs it |

So neither app has the bounded 16-message receive queue that killed the QA harness;
a Dart `StreamSubscription` has no such cap. **The rehearsal's exact trigger was
harness-specific.** Real exposure needs the app to stop draining entirely — an
isolate suspended by iOS or Android when the app is backgrounded or the screen
locks — for long enough to fill the OS socket buffer, which at eighty bytes a frame
is thousands of frames, *and* ~50 more seconds of ride.

That makes the real-world probability much lower than the rehearsal implied, and it
should be said plainly. It does not make either fix wrong: the echo was pointless
traffic on the socket carrying `complete`, backgrounding is an ordinary thing a
driver's phone does, and the GPS-ingestion stall (§2) has no client-side mitigation
at all.

### Two things the apps should still change

1. **Treat "send succeeded" as no evidence.** `send()` on a half-open socket
   succeeds — that is precisely why the rehearsal reported completed rides that had
   not completed. Every lifecycle command needs an acknowledgement, a timeout and a
   retry, and the retry is safe because completion is idempotent (§8). The driver app
   must not show a finished ride until the server says so. **This is the one that
   would have turned a silent data-integrity failure into a visible, recoverable
   one**, and it is worth more than either server fix.
2. ~~**Give the keepalive a real action.**~~ **Done server-side in `c2c2b46`.** The
   driver app pings with `{"action":"ping"}`, which used to be rejected as an invalid
   action — so every 25 seconds the app received an `{"type":"error"}` frame,
   indistinguishable from a real command failure by any client that surfaces errors.
   It now returns `{"type":"pong"}`; verified on the deployed build. The action
   allow-list is otherwise unchanged, which is asserted. **No app change needed** —
   the existing ping now gets a clean answer.

### Remaining follow-ups

- ~~**Observability is the real gap.**~~ **Done in `c2c2b46`.** Nothing logged when
  a command channel was lost, which is the deepest reason this went undetected: the
  failure had no server-side symptom at all. `TripStatusConsumer` now logs
  `trip_command_socket_lost` at WARNING (so it survives production's log level) with
  the trip id, the participant's role, the trip status and the close code — and stays
  silent when a ride ends normally, which is asserted, because a warning that fires on
  every healthy ride is noise. `location_frames_coalesced` is emitted at disconnect.
  **Had this existed, the original diagnosis would have been one log line.**
- **`database_sync_to_async` is process-wide single-threaded.** Not the cause here,
  but with N drivers each pinging every 2.5s, every location frame and every
  lifecycle command in the process queue through **one thread**. At 100 concurrent
  drivers that is ~80 submissions/second on one thread. This needs a load test
  before scale, and it is a stronger argument for extracting the lifecycle service
  out of `consumers.py` than any tidiness argument.
- **`servers/redis_client.py` logs raw coordinates at DEBUG.** Inert in production
  where `LOG_LEVEL` is `WARNING`, but a latent location-privacy leak one environment
  variable away. Worth removing on principle.
- **`/api/v1/ride/trip/<id>/cancel/` and `/driver-cancel/` return 400 for an
  `in_progress` trip** while the WebSocket `cancel` succeeds. Possibly intentional,
  but the inconsistency cost time during cleanup and should be either documented or
  reconciled.
- **The rehearsal harness used `/api/v1/ride/my-trips/`, which does not exist** and
  returned 404 silently, so its inter-scenario cleanup never ran. `/api/v1/ride/active/`
  is the real endpoint. Harness defect, recorded so the next rehearsal does not
  inherit it.
