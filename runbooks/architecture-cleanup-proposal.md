# Architecture cleanup proposal — boring architecture is good architecture

- **Status:** proposal. No refactor performed.
- **Date:** 2026-09-22
- **Principle:** domain logic callable without a WebSocket; consumers, views and
  Celery tasks become thin adapters.

## Measured, not asserted

| Module | LOC | fns | loc/fn | concerns touched |
|---|---|---|---|---|
| `admin_dashboard/views.py` | 6655 | 37 | **179** | db, http, push, money |
| `payments/views.py` | 2029 | 13 | **156** | db, http, money |
| `driver/services.py` | 1795 | 32 | 56 | db, money |
| **`consumers.py`** | 1758 | 52 | 33 | **db, redis, channels, celery, push, money** |
| `ride/views.py` | 1536 | 17 | 90 | db, redis, http, push |
| `redis_client.py` | 967 | 30 | 32 | redis only |
| `driver/views.py` | 802 | 14 | 57 | db, redis, http, money |
| `pricing/services.py` | 536 | 14 | 38 | db, redis, money |
| `ride/dispatch.py` | 535 | 16 | 33 | db, redis, channels, celery, push |

Two different problems, and they need different treatments.

**`consumers.py` — breadth.** Its functions are a healthy 33 lines, but it
touches every concern in the system. Trapped inside a WebSocket transport class:

```
_create_trip                  ride lifecycle + pricing + Redis cache + scheduling
_accept_trip                  the entire acceptance invariant
_update_trip_status           the whole state machine, incl. completion
_confirm_cash_payment         payments
_create_payment_on_complete   payments
_process_refund_on_cancel     payments
_current_trip_snapshot        a read model
_add/_remove_driver_location  location
```

None of that is transport. It is the ride domain, the payment domain, a read
model and a location service, all reachable only by opening a socket. That is
why PR 3's contention test had to reach into
`TripStatusConsumer.__dict__['_accept_trip'].func` — the invariant is real
business logic wearing a transport costume.

**`admin_dashboard/views.py` and `payments/views.py` — depth.** 179 and 156
lines per function. These are not mixing layers so much as having no internal
structure at all. `admin_dashboard` is also where the PR 1 authorization hole
lived, and 6655 lines is precisely why three missing decorators went unnoticed.

**What is already fine, and should be left alone.** `redis_client.py` is a clean
adapter — it mentions the channel layer only in a comment and imports nothing
from Channels. `pricing/services.py` is a genuine domain service and is the
model to copy. `ride/dispatch.py` (added in ADR-0008) touches several concerns
but by design: it is the dispatch domain service, callable from a Celery task
with no socket in sight. It is proof the pattern works here.

## Target structure

Only what the existing code actually supports — no speculative packages.

```
servers/ride/
    models.py
    dispatch.py            EXISTS. dispatch domain service.
    location_trail.py      EXISTS (branch). sampling policy.
    services/
        acceptance.py      from _accept_trip  (+ the Stage-1 invariant)
        lifecycle.py       from _update_trip_status (the state machine)
        creation.py        from _create_trip
        completion.py      completion side-effects, ordered
        read.py            from _current_trip_snapshot
    consumers.py           THIN: parse frame, call service, send frame
    views.py               THIN: parse request, call service, render

servers/payments/
    services/
        collection.py      charge, cash confirmation, gateway orders
        settlement.py      commission + driver credit (from driver/utils)
        refunds.py         from _process_refund_on_cancel
    views.py               THIN

servers/pricing/services.py   ALREADY the right shape. Leave.
servers/notifications/services.py   thin facade over the existing FCM task
```

The test for whether this has worked is simple: **can the acceptance invariant
be tested without constructing a consumer?** Today it cannot.

## Incremental extraction order

Ordered by risk-adjusted payoff, each step independently shippable. No
big-bang.

**1. `acceptance.py` — do this one first.**
Move `_accept_trip`'s body to `servers/ride/services/acceptance.py::accept_trip(trip_id, driver_user)`.
The consumer becomes four lines. Highest payoff and lowest risk, because PR 3
already gave this path the best test coverage in the codebase, including real
PostgreSQL contention tests — so the refactor is guarded before it starts, and
those tests stop reaching into `__dict__`.

**2. `completion.py`.**
Completion currently does payment creation, notification, push, receipt
scheduling and settlement inline in the state machine. The fare report shows why
this matters: `_create_payment_on_complete` and `credit_driver_wallet` both read
the fare inside that transaction, and ordering is load-bearing. Extracting makes
the order explicit and reviewable instead of implicit in a 120-line branch.
**Do this before fare finalisation**, not after.

**3. `settlement.py`.**
`credit_driver_wallet` lives in `driver/utils.py` — a utils module holding the
commission calculation. It is called from **seven** places. Moving it to
`payments/services/settlement.py` puts money logic where money logic belongs
and makes the seven callers visible.

**4. `lifecycle.py`.**
The remaining state machine. Larger and touches more paths, so it benefits from
1 and 2 landing first.

**5. `admin_dashboard/views.py` split by page.**
Purely mechanical: 37 functions into ~6 modules by dashboard area. No logic
change. Worth doing because 6655 lines is where the authorization hole hid, and
a reviewer cannot hold that file in their head.

**6. `payments/views.py`.**
156 lines per function, and it is the highest-consequence code in the system.
Deliberately last: it needs the most care and least time pressure, and steps 1–3
will have established the pattern.

## Rules for doing it

* **One module per PR.** No "and also".
* **Pure move first, improve second.** A refactor PR that also fixes a bug is a
  PR nobody can review.
* **Tests move with the code and must pass unchanged before any cleanup.** If a
  test needs editing to survive the move, the move changed behaviour.
* **Adapters get no logic.** If a consumer or view has an `if` about business
  state after the move, the extraction is incomplete.
* **Stop when it stops paying.** `pricing/services.py` and `redis_client.py`
  are already right. Not every file needs to move.

## What NOT to do

* No new apps, no microservices, no Kafka, no event bus. The problem is that
  domain logic lives in transport classes — that is fixed by moving functions
  between modules in the same process.
* Do not convert `TripStatus` to a CharField as part of this. It is a good idea
  (see ADR-0009 draft) but it is a data migration touching every status read,
  and mixing it with a structural refactor makes both unreviewable.
* Do not rename models or change the API shape. This is internal only; no
  client should be able to tell it happened.
