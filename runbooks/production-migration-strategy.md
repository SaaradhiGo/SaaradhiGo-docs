# Production release migrations — recommended strategy

- **Status:** design. **No production change made.** QA already has an interim
  pre-deploy step (disclosed below).
- **Date:** 2026-09-23
- **Requirements:** exactly one migration execution per release; failure blocks
  rollout; observable logs; no migration from every worker replica; no application
  revision assuming a schema its migration has not applied.

## What exists today

| Environment | Migration mechanism |
|---|---|
| QA `backend` | Pre-deploy command `python manage.py migrate --noinput` (added today) |
| QA `celery` | none |
| **production `backend`** | **none** |
| **production `celery`** | **none** |

Production has no automated migration step at all, and the production `backend`
service has no application variables set — it is not a live service in its current
state. Any strategy below is therefore being designed onto a clean slate, not
retrofitted onto a running release process.

## The two candidates

### A. Railway pre-deploy command

Railway runs the command once, before the new version receives traffic, and a
non-zero exit **blocks the deployment**. That maps exactly onto three of the four
requirements with no new infrastructure.

- *Exactly one execution per release* — yes. It is a property of the deployment,
  not of the replica count: pre-deploy runs once for the service regardless of how
  many replicas the new version will have.
- *Failure blocks rollout* — yes, natively. This is the part that a start-command
  migration (`migrate && daphne`) gets wrong: there, a failed migration crash-loops
  the container and every replica retries it.
- *Observable* — yes; it appears in the deployment logs. QA's shows
  `Applying ride.0012_triplocationpoint... OK`.
- *No migration from every worker replica* — yes, **provided it is attached to
  exactly one service.**

Cost: it is per-service configuration, so it must be set on `backend` and
deliberately **not** on `celery`. Nothing in the platform enforces that; it is a
convention someone can break by copying config between services.

### B. A dedicated release/migration service or job

A separate service whose only job is to run `migrate`, ordered before the others.

- Explicit and self-documenting; the release step is a thing you can point at.
- But Railway does not give ordering guarantees between services on a push: the
  `backend` deploy does not wait for a separate migration service to finish. You
  would have to enforce ordering out of band (a manual gate, or a cron-style job
  triggered by CI), which reintroduces the very race the requirement is about.
- It also doubles the build surface: another service building the same image on
  every push, which QA has already shown to be slow — builds went from ~6 to ~12
  minutes with three services on one repo.

## Recommendation: A, with three specific guards

The pre-deploy command, attached to **`backend` only**, is the simplest mechanism
that actually satisfies the requirements. B buys explicitness and pays for it with
an ordering problem it cannot solve on this platform.

The guards matter as much as the choice:

**1. Attach it to exactly one service, and write down why.**
`backend` (the web service), never `celery`. If both had it, two migrations would
race on every deploy. Postgres would serialise them and the loser would mostly
no-op, but "mostly" is not a property to rely on for schema.

**2. Keep `backend` at one replica during a schema-changing release, or make
migrations replica-safe.**
Pre-deploy runs once, so replicas do not race the *migration*. What they can race
is the **window**: with a rolling deploy, old-code replicas serve traffic against
the new schema while the new version rolls out. That is fine for additive
migrations and dangerous for destructive ones. Hence the next guard.

**3. Additive-only migrations in a single release; destructive changes take two.**
This is the discipline that makes the whole thing safe, and it is worth stating as
a rule rather than a preference:

- release N: add the column/table, backfill, start writing to it, keep reading the
  old one;
- release N+1: switch reads, stop writing the old one;
- release N+2: drop the old one.

Every migration proposed in the current workstream (`FarePricing` columns,
`TripSettlement`, `PromoRedemption` constraints) is additive and fits release N.

## Exact Railway steps, when you choose to do it

Nothing below has been performed on production.

1. **Production `backend` service → Settings → Deploy → Pre-deploy Command:**
   ```
   python manage.py migrate --noinput
   ```
2. **Confirm production `celery` has no pre-deploy command.** It must stay empty.
3. **Verify on the next deploy** that the logs contain `Running migrations:` and
   either `Applying ...` lines or `No migrations to apply`, and that a deliberately
   failing migration blocks the deploy. The cheapest way to prove the block: on a
   throwaway branch in QA, add a migration that raises, deploy, confirm the
   deployment fails and the previous version keeps serving. Do this once, in QA,
   before trusting it in production.
4. **Only then** consider raising production `backend` replicas, and only with the
   additive-only rule in force.

## The interim QA state, disclosed

QA `backend` already has this pre-deploy command, added today so that the GPS
trail's migration would apply at all — neither QA service ran migrations before,
so `ride.0012_triplocationpoint` would never have been created and the GPS
enablement step was impossible without it.

It has run successfully twice:
```
Applying ride.0012_triplocationpoint... OK
Applying pricing.0006_trip_fare_shadow... OK
```

QA `celery` was deliberately left without it. Production was not touched.

## The failure rehearsal — performed, and it works

Run in QA on 2026-09-23. The pre-deploy command was temporarily set to a migrate
invocation that cannot succeed and cannot change anything:

```
python manage.py migrate ride 0099_deliberately_missing --noinput
```

`manage.py` rejects an unknown migration name before touching the database, so no
DDL runs and no row moves. Exit code 1, verified locally first.

**Result — the rollout was blocked, exactly as required:**

| Observation | Evidence |
|---|---|
| Pre-deploy ran and failed | `CommandError: Cannot find a migration matching '0099_deliberately_missing' from app 'ride'.` then `Stopping Container` |
| The new revision did NOT go live | deployment `f47447e6` status **FAILED** |
| The previous healthy revision kept serving | deployment `8a4e99a0` stayed **SUCCESS**; `/healthz` returned 200 on every one of 12 probes across 300 s, with no gap |
| Nothing persistent changed | the command errors before any schema work; `showmigrations` unchanged |

The failing configuration was restored immediately afterwards.

### A finding that matters for the runbook

**Railway's `redeploy` reuses the previous deployment's configuration snapshot.**
The first attempt at this rehearsal set the failing command and then used
`redeploy` — and the pre-deploy container ran the *old* command
(`migrate --noinput`, logging `No migrations to apply`) and the deployment
succeeded. The experiment proved nothing until it was re-run through a real
deployment (a git push).

Two consequences:

1. **A pre-deploy command change only takes effect on the next real deployment.**
   Setting it and clicking redeploy does not apply it.
2. **Restoring it also needs a real deployment**, which is why an empty commit was
   pushed to bring QA back to the correct command.

Anyone configuring this in production must push a commit to verify it, not
redeploy.

### Worker behaviour during the failure

The `celery` service has no pre-deploy command and deploys independently, so it
was unaffected: it kept consuming from the broker against the old, unchanged
schema. That is the correct outcome here — but it is the reason the additive-only
rule matters. Had the blocked migration been destructive, the worker would have
been running old code against a schema the failed release never created, which is
harmless, while the *reverse* (worker on new code, migration not yet applied)
would not be. Migrations lead; code follows.
