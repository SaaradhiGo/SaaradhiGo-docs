# PR 2 durable dispatch — QA rehearsal evidence

- **Status:** PR 2 COMPLETE
- **Date:** 2026-09-22
- **Commit under test:** `41af60f` (backend **and** worker, SHAs matched)
- **Environment:** Railway QA (`SaaradhiGo` / QA), test accounts only
- **Design:** [ADR-0008](../adr/0008-durable-celery-dispatch.md)

Preserved because it is the operational proof that dispatch no longer dies
with the rider's socket. Worker-clock timestamps (IST) are authoritative;
the client clock differed slightly from Railway's.

## Deploy sequence — worker first

The backend was held with a non-matching `watchPatterns` so the push deployed
only the worker, then released and redeployed. Observed intermediate state:
`celery=SUCCESS@41af60f` while `backend=SUCCESS@669ce7a`.

Task registration on the new worker — all 10 present, including the new one:

```
ride.dispatch_wave                    servers.ride.tasks.auto_cancel_trip
ride.issue_receipt_for_trip           auth_user.send_push_notification_task
sos.dispatch_sos                      payments.reconcile_stuck_payments
payments.reconcile_stuck_withdrawals  driver.block_expired_driver_licenses
driver.sweep_stale_driver_presence    base.utils.send_otp_via_sns
```

Note for future readers: Railway's log pipeline **reorders lines**, which
split the Celery `[tasks]` banner and made the registry look incomplete on a
first read. Always extract order-independently before concluding a task is
missing.

## Scenario: rider disconnect (the acceptance test) — PASS

Trip 1, fare ₹138.42 (Hyderabad zone + rate card resolved).

| Event | Worker clock (IST) |
|---|---|
| Trip committed to PostgreSQL | `09:13:01.2` |
| `dispatch_started` (backend process) | `09:13:01.798` |
| **Wave 0**, radius 1500 m | `09:13:01.843` |
| **Rider WebSocket destroyed** | `~09:13:01.4` |
| **Wave 1**, radius 3000 m — rider gone | `09:13:21.927` |
| **Wave 2**, radius 5000 m — rider gone | `09:13:42.004` |
| `dispatch_terminal` (final wave sent) | `09:13:42.037` |
| Rider reconnect → `current_trip` | `09:13:56.8` |
| `auto_cancel_trip` → cancelled | `09:14:31.832` |

Wave gaps 20.1 s and 20.1 s against `DISPATCH_WAVE_SECONDS = 20`.

**Waves 1 and 2 executed 20 s and 40 s after the `RideRequestConsumer`
instance ceased to exist.** Reconnect returned authoritative state from
PostgreSQL: `status=requested, driver=None, otp_present=False`.

## Scenario: 90-second timeout — PASS

Fired `09:14:31.832`, i.e. 90.6 s after creation and after all three waves.
`dispatch_offer_cleanup` → `dispatch_terminal` → trip `cancelled`,
`final_fare=None`, `otp=None`, `/ride/active/` → `NO_ACTIVE_TRIP`.

## Scenario: REST rider cancellation + stale wave — PASS

Trip 3: wave 0 at `09:17:39.4`; REST cancel at `09:17:43.8`; backend emitted
`dispatch_offer_cleanup` at `09:17:43.8` — the PR 2 fix firing on the REST
path, which previously cleaned up nothing. The already-queued wave 1 fired at
`09:17:59.480` and returned:

```
{'executed': False, 'reason': 'status_cancelled'}
```

A stale scheduled wave is harmless when it runs, without any task revocation.

## Scenario: no duplicate terminal transition — PASS

`auto_cancel_trip` on both rider-cancelled trips logged *"already cancelled,
standing down"*. `cancelled_at` retained the rider-cancel times (`09:15:55`,
`09:17:43`), not the later timeout times.

## PII log scan — PASS

400 lines from each of backend and worker: **0** E.164 phone numbers, **0**
OTP-context digits, **0** coordinate pairs, **0** JWTs, **0** rider names.

## QA restored after the rehearsal

`TEST_PHONE_NUMBERS` cleared (bypass confirmed rejected),
`DJANGO_LOG_FORMAT` back to `text`, `watchPatterns` restored. All four
services `SUCCESS@41af60f`, `/healthz` green, the PR 1 admin routes still
302.

## PR2-QA-FOLLOWUP — QA acceptance debt

Three legs were not rehearsed operationally. `Driver.approved` defaults to
`False` and the only approval paths are admin-gated
(`admin_dashboard/views.py:618`, driver-admin maker-checker), and QA security
posture was deliberately left untouched — no public database, no temporary
proxy, no admin bootstrapped, no approval bypass.

1. An approved driver receives and accepts a live offer.
2. Candidate drivers receive the `trip_taken` dismissal.
3. Controlled Redis-loss / degradation rehearsal.

All three are covered by automated tests in
`tests/test_durable_dispatch.py`. Re-run them operationally once an approved
QA driver exists through the normal administrative workflow. **Does not block
PR 3.**

Partial indirect evidence for (3): every wave in this rehearsal ran with
`candidates: 0`, which is the same code path a GEO outage produces, and the
trip lifecycle stayed correct throughout.
