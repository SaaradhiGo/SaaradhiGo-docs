# OPS-1 — Celery production readiness

- **Status:** design complete; one low-risk change implemented, service split awaiting human action
- **Priority:** BLOCKS PRODUCTION PILOT
- **Date:** 2026-09-22
- **Context:** [ADR-0008](../adr/0008-durable-celery-dispatch.md) moved driver
  dispatch onto Celery, so the worker is now on the critical booking path. A
  dead worker used to mean no receipts; it now means **no rider can be matched
  with a driver at all.**

## Current architecture, as measured

```
          ┌──────────────┐        ┌──────────────┐        ┌────────────────────────┐
  rider ──│   backend    │───────▶│    Redis     │◀───────│  celery  worker + BEAT │
          │  daphne ASGI │        │  broker db0  │        │  -B  --concurrency 2   │
          │  replicas: 1 │        │  cache  db1  │        │  replicas: 1           │
          │  /healthz ✓  │        │  channels db4│        │  healthcheck: NONE     │
          └──────────────┘        │  geo/offers  │        └────────────────────────┘
                                  │  RDB only    │
                                  │  --save 60 1 │
                                  └──────────────┘
```

| | backend | celery | Redis |
|---|---|---|---|
| Replicas | 1 (unset) | **1 (unset)** | 1 |
| Restart policy | ON_FAILURE ×10 | ON_FAILURE ×10 | ON_FAILURE ×10 |
| Healthcheck | `/healthz`, 600s | **none** | none |
| Sleep mode | off | off | off |
| Persistence | — | — | **RDB only, `--save 60 1`** |

Effective Celery config: broker Redis db0, **no result backend**,
`task_acks_late=False` globally, `worker_prefetch_multiplier=4`,
`broker_transport_options={}` (so `visibility_timeout` is the 3600s default).

### The five problems, in severity order

1. **`-B` runs beat inside the worker container.** This is what pins replicas
   to 1: a second replica would run a second scheduler and double-fire every
   periodic task — payment reconciliation, withdrawal reconciliation, the
   presence sweep, the licence-expiry sweep. Availability therefore cannot be
   improved at all until beat is split out. `settings.py` already anticipates
   this in a comment.
2. **No worker healthcheck.** `ON_FAILURE` only restarts a process that
   *exits*. A worker that is alive but wedged — hung broker socket, deadlocked
   pool — is never detected and never restarted, and the service reports
   healthy while dispatch silently stops.
3. **Broker has no AOF.** `--save 60 1` is RDB snapshots only, so up to 60
   seconds of queued messages are lost if Redis restarts: in-flight dispatch
   waves and pending timeouts. `docker-compose.prod.yml` specifies
   `--appendonly yes`, so the Railway deployment has drifted from documented
   production intent.
4. **ETA tasks live in worker memory.** `apply_async(countdown=…)` is delivered
   to a worker immediately and held until due. With `acks_late=True` a graceful
   shutdown restores the unacked message and it is redelivered promptly; after
   a hard kill it waits out `visibility_timeout` (3600s default), far beyond
   the 90s trip timeout, so that trip simply times out. Degraded, not corrupt.
5. **No result backend.** Nothing can `.get()` a task result. Correct for this
   design — dispatch is fire-and-forget — but it means task outcomes are only
   observable through logs.

## Task classification

Required before any global ack-policy change. `acks_late` converts "lost on
worker death" into "possibly executed twice", which is only an improvement for
tasks that tolerate a second execution.

| Task | Class | acks_late now | Duplicate execution does what | Verdict |
|---|---|---|---|---|
| `ride.dispatch_wave` | **A** | ✅ True | re-reads Trip, no Postgres writes at all; at worst re-sends an offer | correct as-is |
| `servers.ride.tasks.auto_cancel_trip` | **A** | ✅ **True (changed here)** | re-reads inside the lock, finds terminal, stands down; `cancelled_at` untouched | **fixed** |
| `ride.issue_receipt_for_trip` | **A** | ✅ True | `issue_receipt` returns the existing Receipt | correct as-is |
| `driver.sweep_stale_driver_presence` | **C** | False | Redis-only eviction, naturally idempotent | fine either way |
| `driver.block_expired_driver_licenses` | **C** | False | sets an already-set blocked flag | safe to enable, low value |
| `payments.reconcile_stuck_payments` | **C** | False | **needs review**: locks `Payment` and uses idempotency keys, but it is a money path and may run long enough to interact badly with `visibility_timeout` | **do not enable without review** |
| `payments.reconcile_stuck_withdrawals` | **C** | False | currently a **no-op** (see defect below) | do not enable |
| `auth_user.send_push_notification_task` | **D** | ✅ True | duplicate push notification | acceptable |
| `base.utils.send_otp_via_sns` | **D** | False | **duplicate OTP SMS** — cost and user confusion | leave False |
| `sos.dispatch_sos` | **B/D** | False | duplicate FCM to ops **and duplicate SMS to the rider's emergency contact** | **needs idempotency work before late ack** |

**Conclusion: do not set `task_acks_late` globally.** Two tasks (`send_otp_via_sns`,
`sos.dispatch_sos`) would start sending duplicate SMS, and one money reconciler
needs review first. Per-task is the right granularity, which is how it is now.

### Implemented tonight

`auto_cancel_trip` → `acks_late=True`. It is class A, it is the *only* durable
bound on a driver search, and its idempotency was verified against QA rather
than assumed — the live log reads *"auto_cancel_trip: trip 2 already cancelled,
standing down"* with `cancelled_at` preserved. `max_retries` deliberately left
at 0 so this stays a one-line durability fix rather than also changing error
handling. Tests added, including a negative control on the flag and a
triple-delivery test.

### Recommended but NOT implemented

* `broker_transport_options = {'visibility_timeout': 300}` — would redeliver a
  hard-killed dispatch wave inside the trip's own lifetime instead of an hour
  later. **Not applied**: with `acks_late=True`, any task that runs longer than
  the timeout gets redelivered while still running, and the payment reconcilers
  loop over rows and could exceed 300s at scale. Needs a per-task runtime
  measurement first.
* `task_reject_on_worker_lost = True` — requeues immediately on worker loss
  instead of waiting out the visibility timeout. **Not applied**: a task that
  kills its worker (OOM) would then loop forever.
* `auto_cancel_trip` retries for transient DB errors — safe given idempotency,
  but a separate behavioural change.

## Target architecture

```
          ┌──────────────┐        ┌──────────────┐        ┌────────────────────┐
  rider ──│   backend    │───────▶│    Redis     │◀───────│  celery-worker × N │
          │  replicas: N │        │  broker      │        │  (no -B)           │
          │  /healthz    │        │  AOF on      │        │  healthcheck       │
          └──────────────┘        │  noeviction  │        └────────────────────┘
                                  └──────────────┘        ┌────────────────────┐
                                                  ◀───────│  celery-beat × 1   │
                                                          │  replicas: 1 ONLY  │
                                                          └────────────────────┘
```

The only structural change is splitting the scheduler out. Nothing else about
the topology needs to change.

## Exact steps for morning — requires human action

Creating a Railway service incurs cost and changes infrastructure, so this was
not done autonomously. The sequence matters: beat must never run twice.

1. **Create service `celery-beat`** in the QA environment, same repo
   (`SaaradhiGo/SaaradhiGo-backend`), same branch `dev`.
2. Copy **every** environment variable from the existing `celery` service
   (`DB_*`, `REDIS_URL`, `DJANGO_SECRET_KEY`, `JWT_SIGNING_KEY`,
   `CASHFREE_WEBHOOK_SECRET`, `ALLOWED_HOSTS`, `DEBUG_ENV`, `ENVIRONMENT`,
   `RAILWAY_DOCKERFILE_PATH=dockerfile`, `DJANGO_LOG_FORMAT=json`, …).
3. Start command:
   ```
   sh -c "exec celery -A base beat -l INFO"
   ```
   Keep `sh -c "exec …"` — Railway's runtime does not guarantee a shell, which
   is the trap that cost a day during PR 2.
4. Replicas: **1. This must never be scaled.**
5. **Only once beat is confirmed running**, change the `celery` service start
   command to drop `-B`:
   ```
   sh -c "exec celery -A base worker -l INFO --concurrency 2"
   ```
   Between steps 4 and 5 both schedulers run briefly, so periodic tasks
   double-fire for that window. All four are idempotent (two reconcilers, two
   sweeps), so the window is tolerable — but keep it short, and do not perform
   the split during a payout run.
6. Only after 5, raise `celery` replicas to 2+.
7. Enable AOF on Redis to match `docker-compose.prod.yml`: append
   `--appendonly yes` to its start command.
8. Add a worker liveness signal. Simplest that works on Railway: a beat task
   writing a heartbeat key every minute plus an external monitor reading it,
   since Railway healthchecks are HTTP and a Celery worker serves no HTTP.

## Behaviour during deploy and restart, as it stands today

| Event | Effect |
|---|---|
| Worker graceful restart (normal deploy) | in-flight `acks_late` tasks are restored to the queue and redelivered; queued messages survive in Redis |
| Worker hard kill | `acks_late` tasks redelivered only after `visibility_timeout` (3600s); the affected trip times out instead |
| Worker down entirely | **no dispatch waves run at all** — riders cannot be matched. Trips still commit and still time out durably |
| Redis restart | up to 60s of queued messages lost (RDB only). Durable Trip state is unaffected |
| Backend deploy | dispatch unaffected; the socket is not the owner (ADR-0008) |
| Beat down | periodic reconcilers and sweeps stop; dispatch and timeouts unaffected |

## Related defect found while measuring this

**`payments.reconcile_stuck_withdrawals` is a no-op in production.** It returns
`{'ok': False, 'reason': 'no get_payout_status'}` on every run because
`CashfreeGateway` does not implement `get_payout_status`; the code fails soft
and logs a warning, by design. Observed live in QA every 10 minutes.

Consequence: driver payouts stuck in `processing`/`approved` are **never
reconciled**, contradicting the launch-readiness claim that a job "sweeps stuck
states every 10 min". Money can be stranded with no automated recovery. Not
fixed here — implementing a payout-status call changes payment semantics and is
explicitly out of scope for unattended work.
