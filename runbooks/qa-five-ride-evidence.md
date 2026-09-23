# QA five-ride evidence — the first proven ride → GPS → metrics → shadow chain

- **Date:** 2026-09-23
- **Environment:** QA only. Production untouched throughout.
- **Infrastructure:** real QA PostgreSQL, real QA Redis, real WebSockets, real Celery
  worker + Beat, real dispatch, real OTP lifecycle, real GPS stream, real durable
  trail writer, real actual-metrics task, real fare-shadow sweep. **Nothing on the
  critical path was mocked.**
- **PII:** no phone numbers, OTPs, raw coordinates, JWTs, document URLs or passwords
  appear below. Every figure came from the worker's own structured log events or
  from the trip API.

## The rides

All six trips reached `completed` and were verified by reading the trip back from
the API, not by trusting a WebSocket frame.

| Scenario | Trip | Vehicle | Zone | GPS frames sent | Durable points | Coverage | Max rejected segments |
|---|---|---|---|---|---|---|---|
| A — short normal | 7 | sedan | IN-TG-HYD | 14 | 12 | 0.86 | 0 |
| B — longer | 18 | sedan | IN-TG-HYD | 16 | 16 | 0.94 | 0 |
| C — sparse GPS | 13 | sedan | IN-TG-HYD | 4 | 4 | 0.75 | 0 |
| D — duplicate events | 19 | sedan | IN-TG-HYD | 16 (8 unique, each sent twice) | **8** | 0.87 | 0 |
| E — trip A then trip B | 15 | sedan | IN-TG-HYD | 12 | 10 | 0.91 | 0 |
| E — trip B | 16 | sedan | IN-TG-HYD | 8 | 7 | 0.75 | 0 |

## Metrics and fares

| Scenario | Trip | Quoted km | Actual km | Quoted min | Actual min | Estimated fare | Shadow actual | Metric Δ | Context Δ |
|---|---|---|---|---|---|---|---|---|---|
| A | 7 | 1.45 | 1.33 | 1 | 1.17 | 133.08 | 106.92 | **−2.02** | −24.14 |
| B | 18 | 1.67 | 1.67 | 2 | 2.67 | 142.29 | 118.83 | **+2.09** | −25.55 |
| C | 13 | 0.33 | 0.33 | 2 | 2.67 | 100.00 | 100.00 | 0.00 | 0.00 |
| D | 19 | 0.78 | 0.78 | 1 | 0.80 | 106.35 | 100.00 | 0.00 | −6.35 |
| E | 15 | 1.22 | 1.22 | 1 | 1.00 | 124.15 | 104.05 | 0.00 | −20.10 |
| E | 16 | 0.78 | 0.67 | 1 | 0.67 | 106.35 | 100.00 | 0.00 | −6.35 |

`final_fare` is **NULL on every one of the six trips**, read back from the API after
all processing completed.

## What the numbers actually say

**1. Metering barely moves these fares.** The metric delta — the part attributable
to distance and duration alone — is between **−₹2.02 and +₹2.09** across all six
observed rides, and exactly 0.00 on four of them. Both directions occur.

**2. The context delta is ten times larger.** −₹6.35 to −₹25.55, from surge and
pricing context drifting between booking and observation. **If the two had been
collapsed into one number, the conclusion would have been "metering collects ₹24–26
less per ride", which is false.** The two-quote design exists for exactly this, and
this is the first evidence that it was necessary rather than fastidious.

**3. The minimum fare absorbs metering on short rides.** Trips 13, 16 and 19 all
settled at exactly 100.00 on both legs: the min-fare floor bound, so distance
differences had no fare effect at all. Any revenue model for metering must exclude
floor-bound trips or it will overstate the impact.

**4. Duplicate GPS events are idempotent, end to end.** Trip 19 sent every ping
twice — 16 frames — and produced **8** durable points and a distance matching 8
unique positions. The `(trip, source_event_id)` unique constraint did the work; no
application logic had to notice.

**5. Approach movement is stored but never billed.** Every scenario sent 3 pings
before `start`. They were persisted (dispute evidence) and excluded from the
measured distance, which is the OTP-gated `started_at → completed_at` window.

**6. Post-completion pings are refused.** The drain logged `no_active_trip=3` in the
window after trip 7 completed — the terminal state stopped collection, as designed.

**7. The delayed-drain, two-trip case works.** Scenario E completed trip 15, then
began and completed trip 16 while 15's pings were still in the stream. Each trip
received its own points (10 and 7), and the two trails did not mix. Attribution came
from each event's `recorded_at` against each trip's collection window, not from the
driver's state when the drain ran.

**8. No implausible segments in any ride.** `rejected_segments=0` throughout, which
is the expected result for synthetic straight-line movement — the filter is proven
by unit tests, not by these rides.

## Telemetry quality, conceptually

Using only what these rides show, and **not** proposing production thresholds:

| Class | What it looked like here |
|---|---|
| GOOD | trips 18, 15, 7, 19 — coverage 0.86–0.94, 8–16 points, gaps at the sampling interval |
| DEGRADED | trip 16 — coverage 0.75 with 7 points over a 40-second journey |
| INSUFFICIENT | **not observed.** Trip 13 was *sparse* (4 points) but still produced a distance equal to the quote |

## The most important negative finding: coverage_ratio does not measure density

Trip 13 sent 4 pings 40 seconds apart and scored **coverage_ratio 0.75** — better
than trip 16, which sent 7. That is correct by definition and useless as a
sparseness gate: `coverage_ratio` is *span* (first point to last, over journey
duration). Four points spread across a journey score well; four points bunched at
the start score badly.

**So coverage_ratio alone cannot be the quality gate.** A usable gate needs at least
`points`, and probably `points per minute` plus `max_gap_seconds`. This is exactly
the sort of thing five synthetic rides can establish and no amount of reasoning
could.

`max_gap_seconds` is recorded but was not surfaced in the log event — that is a gap
in the observability, listed in the follow-ups.

## Operational findings from running this

**Completion silently failed above roughly 20 driver-socket frames.** Scenario B at
30 frames and scenario D at 24 both reported completion yet left the trip
`in_progress`; at 16 frames both completed reliably. A, C and E were always under
the threshold. The trip's own socket was idle during the journey while the driver
socket carried the pings. **This needs investigation before pilot** — a real driver
on a 20-minute trip will send far more than 30 pings. It is recorded as a pilot
blocker, not worked around.

**A worker restart delays ETA/countdown tasks.** Merging during the rehearsal
restarted the worker, and `compute_trip_actuals` (a countdown task) did not fire
until much later — Celery's prefetched ETA tasks return to the queue only after the
broker visibility timeout, which defaults to an hour. Practical consequence:
`auto_cancel_trip` can be similarly delayed by a deploy.

**Railway's pre-deploy command runs without a shell.** A command joined with `&&`
silently executed only the first half and the deployment still reported SUCCESS.
The same trap this project already hit with the start command.

**The Railway CLI can fail silently on a multi-line variable value.**
`railway variables --set KEY=<multi-line>` reported nothing and changed nothing; the
MCP/API path handled it correctly.

## Harness defects found and fixed (not product defects)

1. Read the trip OTP from the **driver's** socket. It is not there, correctly: the
   rider's `trip_update` carries the OTP because the rider reads it aloud. Fixed by
   holding the rider socket open.
2. Requested a ride in the same instant the driver's socket opened, so dispatch found
   0 candidates — the geo-index entry is written in the background after accept.
   Fixed with a settle delay.
3. Did not verify completion, so a failed `complete` looked like a success and the
   next scenario's cleanup cancelled the trip. Fixed by reading the trip back.
4. No cleanup between scenarios, so one stuck trip blocked every later ride via the
   one-active-trip invariant — the invariant behaving correctly.

## Billing invariants

Asserted after all six rides, by reading each trip back through the API:

| Invariant | Result |
|---|---|
| `final_fare IS NULL` | ✅ all six trips |
| `estimated_fare` unchanged | ✅ matches the value quoted at booking |
| Shadow changed no Payment | ✅ payment_status untouched; cash trips still `pending` |
| Shadow changed no wallet or ledger row | ✅ no WalletTransaction exists for these trips |
| Shadow changed no commission or settlement | ✅ no settlement ran — these trips were never paid |
| Shadow changed no receipt amount | ✅ no receipt issued (unpaid cash trips) |
| Shadow registered no demand | ✅ `record_demand=False` on every shadow quote, asserted per call in unit tests |
| Terminal state stopped GPS collection | ✅ `no_active_trip` counted the post-completion pings |

One honest limitation: because these were cash trips that were never marked paid, no
settlement or receipt ran at all — so "settlement unchanged" is proven only in the
trivial sense. Proving the shadow cannot disturb a **settled** trip needs a QA ride
that completes payment, which is the next rehearsal.
