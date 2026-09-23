# PR 3 Stage 2 prerequisite — driver active-trip conflict audit

- **Status:** OPEN — blocks Stage 2
- **Raised:** 2026-09-22
- **Context:** PR 3 Stage 1 added the application-level guard. Stage 2 would add
  a durable uniqueness constraint, and `AddConstraint` **fails mid-migration**
  if any violating row exists.

## What must be true before Stage 2 is approved

No driver may hold more than one Trip in `accepted`, `reached` or
`in_progress`.

## The audit query — read-only, no PII

Returns IDs and counts only: no phone numbers, names or coordinates. Safe to
run against production on a replica.

```sql
SELECT t.driver_id_id                          AS driver,
       COUNT(*)                                AS active_trips,
       ARRAY_AGG(t.id ORDER BY t.id)           AS trip_ids,
       ARRAY_AGG(DISTINCT s.status_code)       AS statuses
FROM ride_trip t
JOIN ride_tripstatus s ON s.id = t.status_id_id
WHERE t.driver_id_id IS NOT NULL
  AND s.status_code IN ('accepted', 'reached', 'in_progress')
GROUP BY t.driver_id_id
HAVING COUNT(*) > 1;
```

Django equivalent, if a shell is easier than SQL:

```python
from django.db.models import Count
from servers.ride.models import Trip, DRIVER_ACTIVE_TRIP_STATUSES
(Trip.objects
   .filter(driver_id__isnull=False,
           status_id__status_code__in=DRIVER_ACTIVE_TRIP_STATUSES)
   .values('driver_id')
   .annotate(n=Count('id'))
   .filter(n__gt=1))
```

**Zero rows is the required result.** Any rows must be resolved by hand — decide
which trip is real and move the others to a terminal status — before the
constraint is added.

## Status by environment

| Environment | Result | Notes |
|---|---|---|
| Local / CI | **0 violations** | verified; no migration or fixture creates Trip rows, so violations can only arise at runtime |
| QA (Railway) | **0 violations** | holds only the three cancelled PR 2 rehearsal trips, all with `driver IS NULL` |
| **Production** | **UNKNOWN — must be run** | no access was arranged, and none should be: do not expose production PostgreSQL to obtain this |

Production has **not** been audited and must not be described as clean. Obtain
the result through an existing operational channel — a read replica, an
existing DBA/ops path, or a scheduled read-only job. Do not expose production
PostgreSQL publicly and do not create a temporary proxy for it.

## Why a plain unique constraint will not do

`Trip.driver_id` is deliberately **never nulled**: completed and cancelled trips
keep their driver for history and settlement (`credit_driver_wallet` reads it).
So uniqueness must be conditional on status.

A PostgreSQL partial index cannot express that condition on the current schema:
the predicate would have to join `ride_tripstatus`, and index predicates permit
neither joins nor subqueries. Hardcoding status IDs is not an option either —
there is **no seed migration for `TripStatus`**, so its rows are created lazily
by `get_or_create` and its integer IDs differ per environment. QA has so far
materialised only `requested` and `cancelled`.

Stage 2's candidate design is therefore a nullable `active_driver` column with a
plain `UniqueConstraint(fields=['active_driver'])` — unlimited NULLs are allowed
in a PostgreSQL unique index, so no predicate is needed. That carries a
synchronisation obligation which Stage 2 must enumerate before it is authorised.
