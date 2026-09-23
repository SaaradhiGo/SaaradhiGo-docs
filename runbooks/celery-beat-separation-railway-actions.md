# Celery Beat separation — exact Railway actions

- **Status:** instructions for a human. **No infrastructure changed by me.**
- **Date:** 2026-09-23
- **Project:** SaaradhiGo (`ce6d83ea-92d8-4be3-b59c-72c69179c7a9`)
- **Note:** `ops/celery-production-readiness` (`f47e9ed`) remains **unmerged** and
  its task-semantics diff is still awaiting review. Nothing below depends on it.

## The problem, stated precisely

The QA `celery` service runs:

```
celery -A base worker -B -l INFO --concurrency 2
```

`-B` embeds the Beat scheduler in the worker process. That is convenient and it is
why the scheduled sweeps work today. It also means:

- **The worker cannot be scaled.** Two replicas means two Beat schedulers, and
  every periodic task fires twice. With the GPS drain now on a 60-second schedule
  and the fare shadow on 15 minutes, duplicate firing is duplicated work against
  the same Redis stream and the same rows. The idempotency in those tasks makes it
  survivable, not correct.
- **A worker restart is a scheduler restart.** Beat's schedule state lives in the
  worker, so a deploy can skip or double-fire a tick.
- **There is no liveness signal.** The service has no healthcheck path; Railway
  only knows the process is running, not that it is consuming.

This is why scaling the combined service must not happen before the split.

## The five actions, in order

Do not reorder. Each step's verification is what makes the next one safe.

### 1. Create a dedicated Beat service

In the SaaradhiGo project, **QA environment** first:

- New service → deploy from the same GitHub repo, `SaaradhiGo/SaaradhiGo-backend`,
  branch `dev`.
- **Start command:**
  ```
  celery -A base beat -l INFO
  ```
- **Variables:** copy the existing `celery` service's variables verbatim. It needs
  the same `REDIS_URL`, `DB_*`, `DJANGO_SECRET_KEY`, `CASHFREE_WEBHOOK_SECRET`,
  `ALLOWED_HOSTS` and `RAILWAY_DOCKERFILE_PATH`. Beat imports Django settings, so a
  missing `CASHFREE_WEBHOOK_SECRET` will stop it booting — `payments.apps` refuses
  to start without it.
- **Replicas: 1.** This is not a preference; two Beat processes double every
  scheduled task.
- No healthcheck path (Beat serves no HTTP).

**Verify before continuing:** the new service's logs show `beat: Starting...` and
then `Scheduler: Sending due task ...` lines naming the tasks in
`CELERY_BEAT_SCHEDULE`.

### 2. Verify there is exactly one Beat

At this point there are **two** — the new service and the `-B` inside the worker.
That is expected and temporary, and it is why step 3 follows immediately.

Confirm the new one is genuinely scheduling before removing the old one, so there
is never a window with zero schedulers:

- In the new Beat service's logs, look for `Sending due task
  gps-trail-drain-every-minute`.
- In the `celery` worker's logs, look for the matching `Task
  ride.persist_location_trail ... received`.

If the worker receives the task, the new Beat is reaching the broker correctly.

### 3. Remove `-B` from the worker

Change the `celery` service's start command to:

```
celery -A base worker -l INFO --concurrency 2
```

Redeploy. **Verify:** the worker's logs no longer contain `beat:` lines, and tasks
are still received on schedule (sent by the Beat service, not by itself).

Now exactly one Beat exists.

### 4. Add a worker liveness strategy

Railway's healthcheck is HTTP-only, so a Celery worker cannot use it directly.
Two options, in order of preference:

**(a) Restart policy plus log alerting — do this now.**
- Restart policy: `ON_FAILURE`, max retries 10, on both the worker and Beat.
- Alert on the absence of `ride.persist_location_trail` in the worker logs for
  more than 5 minutes. With the drain on a 60-second schedule, silence for five
  minutes means the worker is not consuming, which is the condition that matters.

**(b) A real liveness probe — do this when the split has settled.**
Add a tiny task that writes a timestamp to Redis, scheduled every minute, and a
`/healthz/worker` endpoint on the **backend** service that returns 503 if that
timestamp is older than three minutes. This makes worker liveness visible to the
same monitoring as the web service, without giving the worker an HTTP port.

Option (b) needs a small code change and belongs in a reviewed PR, not in a
dashboard session.

### 5. Only then scale the worker

With one Beat and a liveness signal, the worker is safe to scale. Before doing it,
check whether it is actually the bottleneck: `--concurrency 2` with the current task
mix is probably not saturated, and scaling a non-bottleneck hides the real one.

If scaling: raise `--concurrency` first (cheaper, same process), and add replicas
only if the queue depth stays high.

**Do not** scale the `celery` service before step 3 is verified.

## After QA, production

Repeat steps 1–3 in the production environment **separately**. Railway service
configuration is per-environment, so the QA change does not propagate — which is
deliberate here: verify the split in QA for a day first, then repeat it.

## What I changed, so you are not surprised

One QA-only configuration change, made while integrating the GPS stack:

**A pre-deploy command on the QA `backend` service:**
```
python manage.py migrate --noinput
```

Neither service ran migrations before, so the GPS trail table would never have been
created and step 3 of the GPS plan (enabling the trail in QA) could not have worked.
Railway runs a pre-deploy command once before the new version goes live, and a
failure blocks the deploy — which is the behaviour you want for migrations.
Confirmed working: the merge-1/3 deploy logs show
`Applying ride.0012_triplocationpoint... OK`.

**This was applied to QA only.** Production's `backend` service is untouched and
still has no pre-deploy command. Whether production should get the same is your
call — the alternative is a manual migrate step in the release process, and the
argument for the pre-deploy is that a deploy which cannot migrate should not go
live.

Nothing else was created, scaled, deleted or repointed.
