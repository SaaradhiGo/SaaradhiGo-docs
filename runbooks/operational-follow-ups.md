# Operational follow-ups — raised by the PR 2 durable-dispatch rehearsal

- **Status:** Open
- **Raised:** 2026-09-22
- **Source:** QA rehearsal of `SaaradhiGo-backend@41af60f` (ADR-0008)

Deliberately not fixed inside PR 2. Each is an operational change with its own
blast radius.

---

## 1. Separate Celery beat and improve worker availability — before pilot

**Priority: highest of the three. Blocks production pilot.**

ADR-0008 moved dispatch onto Celery, which means **the worker is now on the
critical booking path**. Before this change a dead worker meant no receipts
and no pushes; now it means no rider can be matched with a driver at all.

Observed QA configuration:

| | |
|---|---|
| Start command | `celery -A base worker -B -l INFO --concurrency 2` |
| Replicas | **1** |
| Restart policy | `ON_FAILURE`, max 10 retries |
| Healthcheck | **none** (`healthcheckPath: null`) |
| Sleep mode | disabled |

Problems, in order of severity:

1. **`-B` runs beat inside the worker container.** One replica is therefore
   mandatory: a second replica would double-fire every scheduled task —
   payment reconciliation, withdrawal reconciliation, the driver presence
   sweep and the licence-expiry sweep. So availability cannot be improved by
   scaling until beat is split out. The existing note in `settings.py`
   already anticipates this: *"when we scale to multiple workers, beat must
   move to a standalone container (or adopt django-celery-beat)."*
2. **No healthcheck on the worker.** `ON_FAILURE` only restarts a process
   that *exits*. A worker that is alive but wedged — a hung broker
   connection, a deadlocked pool — is never detected and never restarted.
   Dispatch would silently stop while the service reports healthy.
3. **`auto_cancel_trip` is not crash-durable.** It carries
   `acks_late=False, max_retries=0`. If the worker dies holding that message,
   the message is acked and gone, and the trip is never timed out — it sits
   in `requested` forever. This is pre-existing behaviour that PR 2 did not
   change, but PR 2 makes it matter more because that task is now the single
   durable bound on the whole search.
4. **The broker has no AOF persistence.** QA Redis runs
   `redis-server --requirepass … --save 60 1` — RDB snapshots only, no
   `appendonly yes`. Up to 60 seconds of queued tasks can be lost if Redis
   restarts, including in-flight dispatch waves and pending timeouts. Note
   `docker-compose.prod.yml` *does* specify `--appendonly yes`, so the
   Railway deployment has drifted from the documented production intent.
5. **ETA tasks live in worker memory.** `apply_async(countdown=20)` is
   delivered to a worker immediately and held until due. Because
   `ride.dispatch_wave` sets `acks_late=True`, a graceful shutdown restores
   the unacked message to the queue and it is redelivered. After a hard kill
   it is only redelivered once the Redis visibility timeout expires — the
   default is 3600s, far beyond the 90s trip timeout, so in practice that
   trip just times out. Degraded, not corrupt.

Proposed work:

* Split beat into its own service (`celery -A base beat`), leaving the worker
  as `celery -A base worker`, then raise the worker to **2+ replicas**.
* Add a worker liveness check (a `celery inspect ping` wrapper, or a
  heartbeat key written by a beat task and checked by an external monitor).
* Give `auto_cancel_trip` `acks_late=True` and a small `max_retries`.
* Enable `appendonly yes` on the QA and production Redis to match
  `docker-compose.prod.yml`.
* Consider `broker_transport_options = {'visibility_timeout': 120}` so a hard
  worker kill redelivers dispatch waves inside the trip's own lifetime rather
  than an hour later.

---

## 2. Structured dispatch log fields are not rendered in worker output

PR 2 emits `dispatch_started`, `dispatch_wave_started`,
`dispatch_wave_candidates`, `dispatch_wave_offers`,
`dispatch_wave_successor_scheduled`, `dispatch_wave_skipped`,
`dispatch_terminal` and `dispatch_offer_cleanup` with structured `extra`
fields (`trip_id`, `epoch`, `wave_index`, `radius`, `candidates`).

**Those fields do not appear in the Celery worker's logs.** Celery's
`worker_hijack_root_logger` defaults to `True`, so it replaces the root
handler and Django's `LOGGING` config — including `base.logging_filters.
JSONFormatter`, which is what promotes `extra` to top-level JSON keys — never
applies to worker output. Setting `DJANGO_LOG_FORMAT=json` on the worker was
verified to make no difference for this reason.

The event *names* are present and greppable, and Celery's own
`Task … succeeded in …s: {…}` line carries the return value (which includes
`wave_index`, `radius`, `candidates`, `delivered`, `pushes_queued`), so the
rehearsal could still be traced end to end. But `trip_id` is not in either
place, which makes per-trip tracing in production harder than intended.

Fix is one setting: `CELERY_WORKER_HIJACK_ROOT_LOGGER = False`. It changes
the format of *all* worker log output, so it wants its own PR and a look at
whatever log ingestion is configured.

Also worth noting: it is the `PIIRedactionFilter` that is attached to
Django's handler, so worker output is currently not passing through that
filter either. The PII scan of this rehearsal came back clean (zero phone
numbers, OTPs, coordinates or JWTs in either service), but that is because
the dispatch code does not log those values — not because the filter caught
them.

---

## 3. Two deployment stories — retire or repair the EC2 path

The repository has two deployment mechanisms and only one of them works:

* **`.github/workflows/deploy.yml` → EC2.** Has never succeeded. The `deploy`
  job fails at `Setup SSH` because the `EC2_HOST_KEY` secret is unset and the
  `ssh-keyscan` fallback cannot reach `EC2_HOST`. Confirmed failing
  identically on every recent run.
* **Railway.** The QA environment actually in use, deploying from `dev`.

Consequence: **a red CI result currently carries no information.** `test` and
`lint` can both be green and the overall run still shows failure, so nobody
can use "CI is green" as a merge gate. It also trains reviewers to ignore red
builds, which is how a real failure gets missed.

Decide one of:

* repair the EC2 path (populate `EC2_HOST_KEY` from `ssh-keyscan`, confirm
  `EC2_HOST` reachability) and keep both, or
* delete the `deploy` job and let Railway own deployment, adding a Railway
  deploy status check if a gate is wanted.

Either is fine; having both, with one permanently broken, is not.

Related: the QA `backend` service's start command and `RAILWAY_DOCKERFILE_PATH`
live only in the Railway dashboard. A committed `railway.json` would make the
deployment reproducible and reviewable alongside the code.
