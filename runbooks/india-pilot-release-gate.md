# India pilot — release gate

**Backend revision:** `b6be1e8` on `dev` · **QA verified at:** `a640e1b` via `/version`
**Gates:** 1105 backend tests passed, 9 skipped · `ruff` clean · no pending migrations
**Mobile:** rider and driver CI both green, both producing APK artifacts

GREEN / YELLOW / RED / BLOCKED. No numeric score — a number averages away the two items
that decide this.

---

## The hard gate

**A technical cash pilot cannot be GREEN while any of these is unresolved.** This is the
list the run was asked to hold itself to, answered against evidence.

| # | Condition | State | Basis |
|---|---|---|---|
| 1 | Self-admin vulnerability | **RESOLVED** | Demonstrated end to end, then closed by two independent fixes. 60 tests across every path — OTP step, login body, forged JWT claim, expired and malformed tokens, the password endpoint, rider and driver tokens. Verified closed on the running QA deployment: `role=admin` → 400. |
| 2 | Known active leaked credential | **UNRESOLVED — needs a human** | A database credential appeared in git history. Whether it is still valid cannot be determined from here. Removing it from HEAD is not rotation. **This alone holds the pilot RED.** |
| 3 | No operator | **UNRESOLVED — needs a human** | `bootstrap_qa_operator` is built and tested (16 tests) and has never been run. Railway CLI unauthenticated; the MCP server returns variable names only (`valuesRedacted: true`). |
| 4 | No driver onboarding path | **UNRESOLVED** | Blocked behind 3. The operator half of onboarding has never been exercised by a person. |
| 5 | No APK builds | **RESOLVED** | Rider CI and driver CI both produce debug APK artifacts from current HEAD (95 MB / 89 MB). |
| 6 | No production release architecture | **PREPARED, NOT APPLIED** | `india-pilot-production-package.md` gives the topology, the release strategy and the variable checklist. Nothing provisioned. |
| 7 | Unsafe financial duplication | **RESOLVED** | Across 10/25/50/100-ride load runs: every completed trip settled exactly once, no duplicate wallet movement, no duplicate settlement rows, commission exact to the paisa, `final_fare` NULL throughout. |
| 8 | Double assignment | **RESOLVED** | 30 concurrent riders against 10 drivers: exactly 10 accepted, 20 deterministically refused, zero overlapping trips. |
| 9 | SOS invisible | **RESOLVED** | The `SOSEvent` row is durable independently of its notification; the operator listing can reach it; proven during a broker outage. |
| 10 | Unrecoverable active rides | **RESOLVED for the driver's return** | A returning driver finishes their own abandoned ride with no operator and no shell — done on QA for trip 49. If the driver never returns, an operator is required, which is item 3. |

**Verdict: RED.** Items 2, 3 and 4 are open. Two of the three are single human actions,
and neither is an engineering defect.

---

## Scorecard

### Operations

| # | Item | Status | Evidence |
|---|---|---|---|
| 1 | QA operator account exists | **BLOCKED** | Command shipped and tested; never run. Two independent paths confirmed unavailable. |
| 2 | Operator login rehearsal | **BLOCKED** | Depends on 1. |
| 3 | Driver onboarding through the console | **BLOCKED** | Depends on 1. The ONBOARD link remains unexercised. |
| 4 | KYC approval workflow | **BLOCKED** | Depends on 1. Authorization on the endpoint is now proven; the workflow is not. |
| 5 | Stale-ride detection | **GREEN** | Branch B drill on QA: 12 minutes of silence, no false cancel, money untouched, PostgreSQL authoritative, 5/5. |
| 6 | Stale-ride operator queue | **YELLOW** | 51 tests including negative controls; the page has never been opened. |
| 7 | Abandoned ride recovered by the driver | **GREEN** | Trip 49 on QA: driver returned, completed, cash confirmed, `final_fare` None, supply restored. No operator, no shell. |
| 8 | Abandoned ride recovered by an operator | **BLOCKED** | Depends on 1, and it is the half that matters when a driver never returns. |
| 9 | Cross-user access (IDOR) | **GREEN** | 16 tests: cross-user reads 403, cross-user writes 404, owner and assigned driver 200, unknown id 404 not 500. |
| 10 | Unresolved payout presentation | **YELLOW** | `unresolved` implemented and tested, never presented as `FAILED`, no blind Retry. The UI has not been seen. |

### Security and authorization

| # | Item | Status | Evidence |
|---|---|---|---|
| 11 | Operator privilege boundary | **GREEN** | Escalation demonstrated then closed; 60 multi-path tests; verified on the running QA deployment. |
| 12 | One definition of "operator" | **GREEN** | There were **five** divergent gates. Now one `is_operator`, with a test asserting the console imports the same object rather than a copy. |
| 13 | Authorization via direct API calls | **GREEN** | Every privileged endpoint refuses rider, driver, anonymous, forged-claim and role-without-staff callers, asserted by direct calls. |
| 14 | Ops login brute force | **GREEN** | Was 12 failures in 7.8 s unthrottled. Now bounded per phone and per IP on **both** operator sign-in surfaces, with a non-enumerating refusal. |
| 15 | Operator MFA | **YELLOW** | Implemented with django-otp, enforced where the credential is issued, no production override, recovery codes and an audited break-glass reset. 28 tests. **No operator has enrolled, because no operator exists.** |
| 16 | MFA cannot be bypassed | **GREEN** | Three bypasses closed and tested: straight at the API, via the SMS path, and session-cookie-only. |
| 17 | RBAC | **GREEN** | Four operator roles, nine capabilities. A support account cannot approve a payout — asserted by direct API call, and 6 tests fail if the capability gates are removed. |
| 18 | Privileged-action audit | **GREEN** | Actor, action, target, timestamp and reason on KYC, payouts, driver deletion, DPDP erasure, MFA enrolment and reset, and now pricing — which had no trail at all. |
| 19 | Credential hygiene | **GREEN** | No FCM token, TOTP secret, recovery code or phone number in any log or audit row. Asserted, including on failure paths. |
| 20 | Leaked DB credential | **RED** | See the hard gate. Needs a human. |
| 21 | Maps key restriction | **RED** | One unrestricted key. Three restricted keys, a quota and an alert are required before either app ships. |

### Notifications

| # | Item | Status | Evidence |
|---|---|---|---|
| 22 | Driver FCM registration | **GREEN** | Token-erasure defect fixed; narrow write; token no longer echoed in responses; 11 HTTP tests, 6 of which fail pre-fix. |
| 23 | FCM token rotation | **RED** | No `onTokenRefresh` listener. A long-signed-in driver can hold a token the server does not have. |
| 24 | Background ride offer | **YELLOW** | Server path tested, client handler present and degrading quietly. **No real device has been observed receiving one.** Deliberately not GREEN. |
| 25 | OTP delivery truthfulness | **GREEN** | A broker failure returns 503, not a fake success. Retry classification tested; a retry signal is never swallowed into a success. |
| 26 | OTP push to the rider | **GREEN, with a measured caveat** | The accept-time OTP push is **best-effort**: 6 of 50 riders missed it at 50 concurrent. All 6 recovered via the pull path, and the rider app now refetches when it reaches a state that should have an OTP and does not. |

### Celery and Redis

| # | Item | Status | Evidence |
|---|---|---|---|
| 27 | Worker-loss durability | **GREEN** | Real SIGKILL against a real broker: message survives, is redelivered, completes. 14/14. |
| 28 | Recovery objective per task | **GREEN** | Nine tasks classified, the classification enforced by tests rather than written in a runbook. |
| 29 | The accept deadline specifically | **GREEN** | Was ~15-17 min via broker redelivery. Now ~3 min via a database-backed sweep that does not depend on the broker at all. 14 tests, mostly negative controls. |
| 30 | Duplicate-execution safety | **GREEN** | All six ride/money/safety tasks run twice: no second receipt, notification, SOS event or GPS point. |
| 31 | Celery Beat topology | **YELLOW** | Still `-B` inside the worker in the pilot deployment — correct for one worker, doubles every scheduled task at two. Run as a separate service in the load stack, so the target topology is exercised. |
| 32 | Redis outage matrix | **GREEN** | All seven stages: before booking, during dispatch, after assignment, in progress, before completion, during stale detection, and recovery. 22 tests. |
| 33 | Redis reconstructable | **GREEN** | Repaired from PostgreSQL in both directions, no shell, no operator. |

### Load and performance

| # | Item | Status | Evidence |
|---|---|---|---|
| 34 | Query plans at scale | **GREEN** | 20,000 trips on real PostgreSQL 15; no query sequentially scans a table of 5,000+ rows. |
| 35 | List endpoint bounds | **GREEN** | 120 rows behind each endpoint; every one returns a page, verified through HTTP. |
| 36 | Concurrent ride load | **GREEN** | 10/10, 25/25, 50/50 and 100/100 completed on an isolated stack with real PostgreSQL, Redis, Daphne, worker and beat. |
| 37 | Load financial invariants | **GREEN** | All six invariants PASS at every stage. |
| 38 | Driver exclusivity under contention | **GREEN** | 30 riders / 10 drivers: 10 accepted, 20 refused, 0 overlaps. |
| 39 | Production capacity | **NOT CLAIMED** | Deliberately. One machine, localhost networking, `fsync=off`, warm cache. The harness prints that caveat every run. No capacity number appears anywhere in these documents. |

### Production and release

| # | Item | Status | Evidence |
|---|---|---|---|
| 40 | Production environment exists | **BLOCKED** | One service, marked deleted, zero variables, no domain, no database, no Redis, no worker. |
| 41 | Production release trigger | **RED** | Production builds from `dev`. Twelve builds triggered, all failed. Harmless today; an accident generator the moment production is configured. |
| 42 | Production build parity | **RED** | Production uses Railpack on Python 3.13 and fails on `psycopg2-binary`; QA builds the Dockerfile on 3.12. Even the image is unproven. |
| 43 | Production boot guards | **GREEN** | Refuses `DEBUG=True`, a missing secret key, `TEST_PHONE_NUMBERS`, any `QA_ADMIN_BOOTSTRAP_*`, and QA-pointing URLs. 26 tests, exercised as real subprocess boots, each with a negative control. |
| 44 | Revision visibility | **GREEN** | `/version` from the platform SHA, naming its source, and saying `unknown` rather than guessing — confirmed on the load stack, which correctly reported `unknown`. |
| 45 | Backup and restore | **RED** | Never rehearsed, in any environment. An unrehearsed restore is an assumption. |
| 46 | Rollback | **RED** | Every production deployment reports `canRollback: false`, because none has ever succeeded. There is no known-good target. |
| 47 | S3 isolation and privacy | **RED** | Requirement written; nothing verified against a real bucket. No public KYC is a requirement, not yet a fact. |
| 48 | Dependency CVEs | **RED, and blocked** | 14 advisories in 2 packages, both pinned by `cashfree-pg==3.2.12`. No fixed `urllib3` exists inside that constraint. |
| 49 | Online payments | **OFF, intended** | Cash pilot. |
| 50 | Automatic payouts | **OFF, intended** | Cash pilot. |
| 51 | Observability | **RED** | `SENTRY_DSN` unset in every environment. A production exception would be visible only in container logs. |

---

## What changed this run

Eleven commits across three repositories. The ones that move the gate:

- **A privilege escalation that let anyone with a phone become an operator** — found by
  attacking the API rather than reading the permission class, demonstrated, closed, and
  verified closed on the running deployment.
- **A fifth divergent definition of "operator"**, guarding pricing, found while building
  RBAC. There is now one definition and a test that fails if anyone copies it again.
- **An unthrottled operator password endpoint** that also enumerated accounts.
- **MFA**, with the enforcement at the point the credential is issued rather than at the
  login form, which is the difference between a control and a form.
- **RBAC**, so a support account cannot release money.
- **Load evidence**, which was the largest engineering gap: 100 rides through an
  isolated stack with every financial invariant reconciled afterwards.
- **The accept deadline no longer depends on the broker**, taking a rider-visible
  failure from ~17 minutes to ~3.
- **The OTP push is best-effort** — measured, not assumed — and the rider app now
  recovers from a missed one.
- **The Redis outage matrix**, all seven stages.

Two findings I got wrong and corrected rather than shipped: a query-plan "seq scan" that
was my paraphrase rather than the real query, and a `/ride/active/` "defect" that is a
documented post-ride fallback. Both are recorded at the code.

---

## What this gate is waiting on

Nothing on this list is an architectural defect.

1. **Rotate or clear the database credential** (or confirm it is already dead).
2. **Run `bootstrap_qa_operator`** against QA — one command, unblocks four items.
3. **Move the production deploy trigger off `dev`** before configuring production.
4. **Watch a real handset receive a backgrounded ride offer.**
5. **Rehearse a database restore** before a real rider exists.
