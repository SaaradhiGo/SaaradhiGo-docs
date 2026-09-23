# ADR-0009 (DRAFT — NOT AUTHORIZED): durable one-driver-one-trip constraint

* Status: **DRAFT. Not authorized. Blocked on the production conflict audit.**
* Date: 2026-09-22
* Deciders: pending
* Relates to: [ADR-0008](0008-durable-celery-dispatch.md),
  [pr3-stage2-prerequisite-audit](../runbooks/pr3-stage2-prerequisite-audit.md)

PR 3 Stage 1 (merged, `8ea4b98`) added the application guard inside the Driver
row lock, with real PostgreSQL contention proof. This records the design for
Stage 2 — the database-level guarantee — so it can be reviewed before any
schema change is written. **No code and no migration has been produced.**

## Why a constraint is still wanted after Stage 1

Stage 1 covers `_accept_trip`, which the investigation proved is the only
*application* path that assigns a driver. It does not cover:

* Django admin (`TripAdmin`), where Stage 1 added only form validation — which
  takes no lock and cannot win a race;
* raw SQL, a data migration, or a `QuerySet.update()` added in future;
* any new assignment path a later change introduces without knowing the rule.

An application check is a rule people must remember. A constraint is a rule the
database remembers.

## Why the obvious constraint is impossible

`Trip.driver_id` is **never nulled** — completed and cancelled trips keep their
driver for history and settlement (`credit_driver_wallet` reads it). So
`UNIQUE(driver_id)` would forbid a driver from ever taking a second ride.

A *partial* unique index cannot express the condition either. The predicate
would have to mean "status is active", but status lives in `ride_tripstatus`,
and a PostgreSQL index predicate permits neither joins nor subqueries.
Hardcoding status IDs is not a workaround: **`TripStatus` has no seed
migration**, its rows are created lazily by `get_or_create`, and its integer IDs
therefore differ per environment. QA has so far materialised only `requested`
and `cancelled`.

## Proposed design

Add a nullable column that *is* the invariant, and put a plain unique
constraint on it. PostgreSQL permits unlimited NULLs in a unique index, so no
predicate is needed.

```python
active_driver = models.ForeignKey(
    'driver.Driver', null=True, blank=True,
    on_delete=models.DO_NOTHING, related_name='active_trip_slot',
)

class Meta:
    constraints = [
        models.UniqueConstraint(fields=['active_driver'],
                                name='trip_one_active_per_driver'),
    ]
```

It is denormalised state, and that is the whole cost of the design: `Trip.driver`
+ `Trip.status` → `Trip.active_driver`. It is worth it only because the derived
value is a **pure function** of two columns on the same row:

```python
def expected_active_driver(trip):
    code = trip.status_id.status_code if trip.status_id else None
    return trip.driver_id if code in DRIVER_ACTIVE_TRIP_STATUSES else None
```

## The exact transition-maintenance matrix

Every production surface that writes `Trip.status_id`, found by AST scan (5
sites; no management command and no Trip data migration exists).

| # | Surface | Transition | `active_driver` before → after | Save style | Action needed |
|---|---|---|---|---|---|
| 1 | `consumers._accept_trip` L1259 | `requested → accepted` | `NULL → driver` | `trip.save()` full | **automatic** via `save()` override |
| 2 | `consumers._update_trip_status` L1526 → `reached` | `accepted → reached` | `driver → driver` (unchanged) | `trip.save()` full | automatic |
| 3 | same → `in_progress` | `accepted\|reached → in_progress` | `driver → driver` (unchanged) | `trip.save()` full | automatic |
| 4 | same → `completed` | `in_progress → completed` | `driver → NULL` | `trip.save()` full | automatic |
| 5 | same → `cancelled` | `requested\|accepted\|reached\|in_progress → cancelled` | `driver or NULL → NULL` | `trip.save()` full | automatic |
| 6 | `ride.tasks.auto_cancel_trip` L155 | `requested → cancelled` | `NULL → NULL` (no driver by definition) | `save(update_fields=[4 cancel cols])` | **add `'active_driver'`** |
| 7 | `ride.views.driver_cancel_trip` L995 | `accepted\|reached → cancelled` | `driver → NULL` | `save(update_fields=[...])` | **add `'active_driver'`** |
| 8 | `ride.views.rider_cancel_trip` L1137 | `requested\|accepted\|reached\|in_progress → cancelled` | `driver or NULL → NULL` | `save(update_fields=[...])` | **add `'active_driver'`** |
| 9 | Django admin `TripAdmin` | arbitrary | recomputed from the submitted driver+status | ModelForm full save | automatic |
| 10 | `Trip.objects.create(...)` (tests, and any future caller) | → `requested` | `NULL` | full insert | automatic |
| 11 | `trip.save(update_fields=['payment_status'])` ×6 | no status/driver change | unchanged | narrow | **no action** — correctly untouched |

So the synchronisation obligation is **three explicit `update_fields`
additions**. Everything else falls out of a single `Trip.save()` override (or
`pre_save` signal) that recomputes the column.

Important subtlety, and the reason the table distinguishes save styles: a
`pre_save` signal **cannot** rescue a narrow save. Django passes `update_fields`
to the signal but mutating it does not change what is written, so a site using
`update_fields` must name `active_driver` itself or the recomputed value is
silently dropped. That is exactly the failure mode that would produce stale
rows, so sites 6–8 are the ones to review hardest.

All eight writing sites already run inside `transaction.atomic()` with the Trip
row locked, so maintenance is same-transaction by construction — no
`on_commit`, no window where status and `active_driver` disagree.

## Reconciliation detection — detect, never auto-repair

```sql
SELECT t.id AS trip, t.driver_id_id AS driver, s.status_code,
       t.active_driver_id AS actual,
       CASE WHEN s.status_code IN ('accepted','reached','in_progress')
            THEN t.driver_id_id ELSE NULL END AS expected
FROM ride_trip t
JOIN ride_tripstatus s ON s.id = t.status_id_id
WHERE t.active_driver_id IS DISTINCT FROM
      (CASE WHEN s.status_code IN ('accepted','reached','in_progress')
            THEN t.driver_id_id ELSE NULL END);
```

`IS DISTINCT FROM` rather than `!=` so NULL mismatches are caught.

Run it as a low-frequency Celery beat task that **logs and alerts only**.
Deliberately no automatic repair: a drift row means a code path was missed, and
silently rewriting it would hide the bug that produced it while mutating
business state. Emit a structured `trip_active_driver_drift` event with
`trip_id`, `status_code`, `expected`, `actual` — all IDs, no PII.

## Migration and rollback

Ordering matters in both directions, and they are not mirror images.

**Forward**

1. `AddField(active_driver, null=True)` — no constraint yet. Nullable add with
   no default, so no table rewrite.
2. Deploy the maintenance code **in the same release**. The migration step runs
   before the new code serves traffic, so the column exists when first written.
   Old code in the window writes NULL, which is harmless: the constraint does
   not exist yet.
3. Backfill, as a data migration or a one-shot command:
   ```sql
   UPDATE ride_trip t SET active_driver_id = t.driver_id_id
   FROM ride_tripstatus s
   WHERE s.id = t.status_id_id
     AND t.driver_id_id IS NOT NULL
     AND s.status_code IN ('accepted','reached','in_progress');
   ```
   A subquery/join is fine here — it is forbidden only in an index predicate.
4. **Run the conflict audit and the drift query. Both must return zero rows.**
5. `AddConstraint(UniqueConstraint(fields=['active_driver']))`. On a large
   `ride_trip` prefer `AddIndexConcurrently` inside
   `SeparateDatabaseAndState` to avoid the `ACCESS EXCLUSIVE` lock; at
   Phase-0 volumes the plain form is fine.

**Rollback — the asymmetry that matters**

`RemoveConstraint` then `RemoveField` are both fast and safe. But **the
constraint must be dropped before the code is rolled back.** Old code does not
maintain `active_driver`, so a trip completing under rolled-back code would
leave the column populated; the driver's next acceptance would then violate the
constraint and raise `IntegrityError` on a rider-facing path. Rolling back code
first turns a latent bug into an outage.

Record that as the operational rule: **constraint off → code back**, never the
reverse.

## Risks

* **A stale `active_driver` blocks a driver from all future rides.** That is a
  worse failure than today's silent double-booking: it is loud and it stops a
  driver earning. Mitigated by same-transaction maintenance, the three explicit
  `update_fields` additions, and the drift reconciler — but it is the reason
  this design is not obviously worth it and needs a human decision.
* `IntegrityError` becomes reachable on the acceptance path. Stage 1's guard
  should catch the case first and return `driver_already_on_trip`; the
  constraint is the backstop. The handler must translate it into the same
  response rather than a 500.
* `on_delete=DO_NOTHING` matches the existing `driver_id`, so the column
  inherits the same dangling-FK caveat already present on this table.
* Adds a second FK from `Trip` to `Driver`, which will confuse anyone reading
  the model without the comment. The comment must be explicit that
  `active_driver` is an invariant slot, not a second assignment.

## Alternative considered and not preferred

Replacing the `TripStatus` FK with a `CharField` would make a genuine partial
index possible (`WHERE status_code IN (...)`), remove a JOIN from every status
read, and delete the `get_or_create`-per-trip default. It is the better
long-term shape and would make this ADR unnecessary. It is also a much larger
migration touching every status read in the codebase, so it is recorded here as
the option to revisit rather than the one to do now.

## STOP

This design stops here. It cannot proceed until the **production conflict
audit** returns zero rows — see
[pr3-stage2-prerequisite-audit.md](../runbooks/pr3-stage2-prerequisite-audit.md).
Production has not been audited and must not be assumed clean. Do not expose
production PostgreSQL to obtain that result.
