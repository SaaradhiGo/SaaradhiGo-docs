# Testing policy — mocks prove policy, real infrastructure proves plumbing

- **Status:** policy, in force from 2026-09-23.
- **Applies to:** dispatch, driver assignment, GPS, fare, settlement, payout.

For these six domains, a change needs **both** kinds of coverage. Not one or the
other, and not "integration tests later".

## The distinction, and why it is not academic

**Fast unit tests prove the policy.** Given these inputs, is the decision right?
Is the discount capped? Is the journey window the OTP-gated one? Is an unknown
provider status treated as "skip" rather than "failed"? These are cheap, run on
SQLite, and should cover every branch.

**Real PostgreSQL and Redis tests prove the plumbing.** Do the pieces actually
connect? Does the writer read the database the producer wrote to? Does the
constraint exist in the schema, or only in the model class? Does the drain find the
trip when it runs *after* completion, which is when it always runs?

This project has a concrete record of why both are required. Every defect below
was invisible to a full, passing, mocked unit suite, and each was found by a test
using the real thing:

| Defect | Found by |
|---|---|
| The trail writer read Redis **db 2** while the producer wrote to **db 3** — it would have persisted zero rows, silently | integration test over real Redis |
| The drain matched pings to the driver's status **at drain time**, so the end of every journey was discarded | end-to-end chain test over real Redis |
| Surge demand counted **zero** for Decimal coordinates; the exception was caught and 0 is also a legitimate answer | integration test with a populated geo index |
| The withdrawal reconciler's in-lock re-check used the **old status vocabulary**, so it would have resolved nothing even with a working provider | test with an injected gateway |

Three of those four are the same shape: something returned a plausible value
instead of failing. A mock cannot catch that, because the mock *is* the plausible
value.

## The rule

A change in the six domains ships when it has:

1. **Unit coverage of every decision branch.** SQLite, mocks at the boundaries,
   fast enough to run on every save.
2. **Integration coverage of the critical path** against real PostgreSQL and real
   Redis, marked `@pytest.mark.postgres` where it needs the database's real
   constraint behaviour so CI's `postgres-concurrency` job runs it.
3. **A negative control for the invariant the change exists to protect.** Revert
   the fix, watch the test fail, put it back. If the test passes either way it is
   not testing the fix. This has caught two tests of mine that were asserting their
   own comments.

## What counts as a critical path

Not everything needs the real thing. These do:

- **dispatch / driver assignment** — anything touching row locks, offer
  generations, or the one-active-trip invariant. `SELECT FOR UPDATE` is a no-op on
  SQLite, so contention tests there prove nothing.
- **GPS** — anything crossing the Redis stream boundary, and anything relying on
  the `(trip, source_event_id)` unique constraint for idempotency.
- **fare** — the canonical quote path, and any constraint on the fare snapshot
  (partial unique indexes do not exist on SQLite in the same form).
- **settlement** — the idempotency key, and anything inside
  `credit_driver_wallet`'s transaction.
- **payout** — candidate selection against the real state machine; provider calls
  stay mocked until the contract is verified in sandbox.

## How it runs today

| Gate | Command | Scope |
|---|---|---|
| Unit | `pytest` | everything except `postgres`-marked; SQLite |
| Integration | `pytest` with `DB_*` set | the same suite against real PostgreSQL |
| Contention | `pytest -m postgres` | row locks, partial constraints |
| CI | `test`, `lint`, `postgres-concurrency` jobs | `deploy` needs all three |

Local Redis is used by the integration tests directly; they `pytest.skip` when it
is unavailable rather than passing vacuously — **a skipped plumbing test must never
read as a green plumbing test.**

## Two anti-patterns this policy exists to stop

**Asserting on source text.** A test that greps a module for a string passes when
the string is in a comment. Assert on behaviour, or read `__code__.co_names` if you
genuinely must test that a call site exists.

**Mocking the thing under test.** If the fix is "read the right Redis database"
and the test mocks the Redis client, the test cannot fail. The mock should sit at
the far side of the boundary being crossed, never on it.
