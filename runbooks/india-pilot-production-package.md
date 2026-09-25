# Production package — target topology, release strategy, and the variable checklist

**Prepared, not applied.** Nothing in this document has been provisioned or deployed.
It is the exact configuration to create, in the order to create it, so that whoever
does it is making one decision at a time instead of thirty at once.

Derived from the QA architecture that is actually proven, and from an audit of the live
Railway project rather than from intent.

---

## 1. Where production is today

| | Observed |
|---|---|
| Services in the `production` environment | **one** |
| That service's state | `staged-delete` (`isDeleted: true`) |
| Uncommitted staged change against it | 1, `STAGED` since 2026-09-22, destructive (`Is Deleted → REMOVED`) |
| Environment variables | **zero** |
| Domains (custom or service) | **none** |
| Volumes | none |
| Successful deployments, ever | **none** (12 on record, all `FAILED`) |
| Source | `SaaradhiGo/SaaradhiGo-backend`, branch **`dev`** |

There is no production PostgreSQL, no Redis, no Celery worker, no beat and no ops
console. Production is not misconfigured; it does not exist.

### Why the twelve builds failed, which is its own finding

QA sets `RAILWAY_DOCKERFILE_PATH`, so it builds from the repository `dockerfile`
(`python:3.12-slim`). Production has no variables, so Railway falls back to the
**Railpack** builder, which selects **Python 3.13**. `psycopg2-binary` publishes no
cp313 wheel, so pip compiles from source and the build dies:

```
Building wheel for psycopg2-binary (pyproject.toml): finished with status 'error'
error: failed-wheel-build-for-install
process "pip install -r requirements.txt" did not complete successfully: exit code 1
```

So "what QA validates" and "what production builds" are already two different
artifacts. Whatever eventually runs in production must be built the same way QA builds
it, or QA has been validating something else.

---

## 2. Target topology (§17)

Seven resources. Same shapes QA runs, because those are the ones with evidence behind
them.

| # | Service | Image / build | Notes |
|---|---|---|---|
| 1 | `backend` | **Dockerfile** (`python:3.12-slim`) | Daphne ASGI. `RAILWAY_DOCKERFILE_PATH` must be set, or Railpack takes over and the build fails as above. |
| 2 | `worker` | same Dockerfile | `celery -A base worker -l INFO --concurrency 4`. Prefork. **Must be Linux** — a Windows worker never restores a dead worker's reserved messages (`should_use_eventloop` excludes Windows), so tasks would be lost silently. |
| 3 | `beat` | same Dockerfile | `celery -A base beat -l INFO`. **Its own service**, not `-B` inside the worker. Embedded beat is correct for exactly one worker and doubles every scheduled task at two. Exercised this way in `docker-compose.load.yml`. |
| 4 | `postgres` | Railway PostgreSQL 15 | Matches QA and the load stack. |
| 5 | `redis` | Railway Redis 7 | Broker, cache, channel layer and geo index. See §6 below for what it must survive. |
| 6 | `ops-web` | the Next.js console | **Optional for the pilot.** The standing decision is that the Django console is canonical and the Next console must not be an operator's only route to any control. |
| 7 | S3 bucket | Railway bucket or AWS | Private. See §8. |

The Django console needs no service of its own: it is mounted at the backend root.

### What runs where, and why it is not one container

The worker cannot share a container with Daphne. A `--concurrency 4` prefork worker and
an ASGI server compete for the same process supervisor, and a worker restart would drop
every WebSocket. They are separate services in QA and in the load stack, and the load
run measured a Celery backlog of 0 through 100 concurrent rides with that shape.

---

## 3. Release strategy (§18)

**Production must not follow `dev`.** It does today, which is why every push during this
run triggered a production build.

| Environment | Trigger | Rationale |
|---|---|---|
| QA | push to `dev` | Current behaviour, correct, keep it. |
| production | **a tag, or a manual deploy from `main`** | A production release must name an immutable commit. |

Two acceptable shapes; pick one and write it down:

- **Tag-triggered.** `main` is protected, releases are `v*` tags, the production service
  deploys tags only. The tag is the immutable identifier.
- **Manual.** The production service is set to manual deploys and an operator picks the
  commit. Simpler, and adequate for a pilot with a handful of releases.

What must NOT remain true is production watching a branch that receives day-to-day work.

**Do this before giving production any variables or a domain.** Today those builds fail
harmlessly. The moment production is configured, every merge to `dev` becomes a release,
and that is a configuration that produces accidents rather than decisions.

Also resolve the destructive staged change pending since 2026-09-22 — commit the
deletion deliberately, or discard it. Leaving a staged delete against the only
production service is a trap for whoever next clicks "apply".

### Verifying a release landed

`GET /version` returns the revision from `RAILWAY_GIT_COMMIT_SHA` and names the source
that answered. It says `unknown` rather than something plausible when nothing resolves —
verified on the load stack, which correctly reported `revision: unknown` with no
platform SHA injected. Never confirm a deploy from a `/healthz` 200: that has already
produced one wrong defect report, from an old container answering while the new
deployment was still queued.

---

## 4. Variable checklist (§19)

**Names only. No values appear in this document, and none should be pasted into it.**

Extracted from every `os.environ` read in `base/` and `servers/`, so it is the set the
code actually consults rather than a remembered list.

Legend: **BOOT** = production refuses to start without it · **FEATURE** = required only
when that feature is on · **OPT** = has a working default · **FORBIDDEN** = production
refuses to start *with* it.

### Django core

| Variable | | |
|---|---|---|
| `DJANGO_SECRET_KEY` | **BOOT** | Guard refuses an empty one. |
| `ALLOWED_HOSTS` | **BOOT** | Guard refuses empty when `DEBUG=False`. |
| `ENVIRONMENT` | **BOOT** | Must be `production`. Everything below keys off it. |
| `DEBUG_ENV` | **FORBIDDEN** as `True` | |
| `CSRF_TRUSTED_ORIGINS` | **BOOT** for the console | The console is a browser session. |
| `CORS_ALLOWED_ORIGINS` | **BOOT** for the mobile apps | |
| `BACKEND_URL`, `FRONTEND_URL` | **BOOT** | Guard refuses values pointing at QA, staging or localhost. |
| `JWT_SIGNING_KEY` | **BOOT** in practice | Falls back to `DJANGO_SECRET_KEY`; set it separately so a leak of one does not mint API tokens. |
| `JWT_ISSUER` | OPT | |
| `DJANGO_SECURE_SSL_REDIRECT`, `DJANGO_HSTS_SECONDS` | OPT, should be set | |
| `DJANGO_LOG_FORMAT`, `LOG_LEVEL` | OPT | |
| `PORT` | platform-injected | |

### PostgreSQL

`DB_HOST` **BOOT** · `DB_NAME` **BOOT** · `DB_USER` **BOOT** · `DB_PASSWORD` **BOOT** ·
`DB_PORT` OPT · `DB_SSLMODE` **BOOT**, must not be `disable` · `DB_SSLROOTCERT` OPT

A production boot with `DB_HOST` unset used to fall back to a local empty SQLite file
and serve traffic against it. That is now a boot guard.

### Redis

`REDIS_URL` **BOOT** — broker (db 0), cache (db 1), channel layer (db 4).

### Celery

`CELERY_VISIBILITY_TIMEOUT_SECONDS` OPT (default 900) — **do not lower without reading
`tests/test_celery_recovery_policy.py`.** Its floor is 540 s and below that recovery
becomes duplicate execution.

### S3

`AWS_ACCESS_KEY_ID` / `AWS_SECRET_ACCESS_KEY` **FEATURE** · `AWS_S3_BUCKET_NAME`
**FEATURE** · `AWS_S3_ENDPOINT_URL`, `AWS_S3_REGION`, `AWS_REGION`,
`AWS_DEFAULT_REGION`, `AWS_S3_ADDRESSING_STYLE`, `AWS_QUERYSTRING_EXPIRE` OPT

### SMS / push / maps

`AWS_SNS_SENDER_ID` **FEATURE** (OTP SMS) · `GOOGLE_MAPS_API_KEY` **FEATURE** — see §9 ·
FCM credentials arrive as a **file** (`firebase-credentials.json`), not a variable, and
its absence is logged and degrades rather than crashing.

### Cashfree — all FEATURE, and all off for the pilot

`CASHFREE_APP_ID` · `CASHFREE_SECRET_KEY` · `CASHFREE_WEBHOOK_SECRET` (**BOOT** — the
payments app refuses to start without it, even with payments off) ·
`CASHFREE_ENVIRONMENT` · `CASHFREE_API_VERSION` · `CASHFREE_PG_BASE_URL` ·
`CASHFREE_PAYOUTS_BASE_URL` · `CASHFREE_PAYOUTS_CLIENT_ID` ·
`CASHFREE_PAYOUTS_CLIENT_SECRET` · `PAYMENT_GATEWAY` · `PAYOUT_GATEWAY`

### Razorpay — a residual worth cleaning up

`RAZORPAY_KEY_ID`, `RAZORPAY_KEY_SECRET`, `RAZORPAY_WEBHOOK_SECRET` are still read in
`base/settings.py` although the operations-console decision records Razorpay as removed
and only Cashfree remains. They default to empty and nothing uses them, so this is dead
configuration rather than a live second gateway — but leave them **unset** in production
and delete the reads, so nobody later assumes a second gateway is wired.

### Security / operations

| Variable | | |
|---|---|---|
| `OPS_MFA_ENFORCED` | ignored in production | Production always enforces; there is deliberately no off switch. |
| `OPS_LOGIN_MAX_ATTEMPTS`, `OPS_LOGIN_LOCKOUT_SECONDS` | OPT | Defaults 8 / 900 s. |
| `TEST_PHONE_NUMBERS` | **FORBIDDEN** | An OTP bypass that also skips throttling. Boot guard refuses it. |
| `QA_ADMIN_BOOTSTRAP_PHONE`, `QA_ADMIN_BOOTSTRAP_PASSWORD` | **FORBIDDEN** | Creates an admin with payout and KYC authority. Boot guard refuses them. |

**The reason those four are guards rather than notes:** production is meant to be
configured by copying QA's variable set, and copying QA's set verbatim ships an
authentication bypass and an admin back door.

### Brand, pricing, behaviour — all OPT with defaults

`PLATFORM_BRAND_NAME` · `PLATFORM_LEGAL_ENTITY` · `PLATFORM_CONTACT_DOMAIN` ·
`PLATFORM_COMMISSION_PERCENT` · `TRIP_ACCEPT_TIMEOUT_SECONDS` ·
`TRIP_UNACCEPTED_SWEEP_GRACE_SECONDS` · `TRIP_STALE_AFTER_SECONDS` ·
`TRIP_ACTUALS_DELAY_SECONDS` · `TRIP_ACTUALS_MAX_*` · `DISPATCH_WAVE_SECONDS` ·
`DISPATCH_RADIUS_WAVES_M` · `GPS_TRAIL_*` · `RIDER_CREDIT_*` · `FARE_SHADOW_*`

### Deliberately OFF for a cash pilot

`WALLET_TOPUPS_ENABLED` · `FARE_SHADOW_ENABLED` (observe-only either way; it writes to
`TripFareShadow` and charges nobody) · every Cashfree credential

### Observability

`SENTRY_DSN` · `SENTRY_TRACES_SAMPLE_RATE` — both OPT, both should be set. There is
currently no error aggregation in any environment, so a production exception would be
visible only in container logs.

### The variable set has been rehearsed

`tests/test_production_boot_guards.py` boots a real process in a subprocess with a
production-shaped configuration using dummy values, and asserts it starts. That is the
proof the **BOOT** column is complete, and it is why this checklist can be trusted to
be sufficient rather than merely plausible.

---

## 5. Backup and restore (§20)

**Requirement, not a status.** Nothing below has been done.

- Railway PostgreSQL provides automated backups. **That is not a backup policy.** A
  backup nobody has restored is an assumption.
- For a cash pilot, the loss that matters is a completed trip and its settlement row:
  the money record. RPO should be minutes, not hours.
- **The drill, before a real rider exists:** take a backup, restore it into a *separate*
  database, point a non-production backend at it, and confirm a known completed trip
  reads back with its fare, its settlement row and its receipt. `qa/query_plan_audit.py`
  already creates and migrates a throwaway database, so the mechanics of pointing a
  backend at a restored copy are established.
- Record the measured restore time. That number is the recovery objective, and without
  it there is no incident plan.

Do not mark this GREEN because the provider says backups exist.

---

## 6. Redis requirements (§21)

Redis carries four distinct jobs and they have different tolerances.

| Job | db | If it is lost |
|---|---|---|
| Celery broker | 0 | Scheduled work is lost. The accept deadline now has a database-backed backstop; other tasks rely on redelivery. |
| Cache | 1 | OTP records and the ops login guard. **Losing this invalidates in-flight OTPs**, so a rider mid-login must request a new one. |
| Channel layer | 4 | Live WebSocket fan-out. Clients reconnect; state is re-read from PostgreSQL. |
| Geo index + presence | — | Dispatch cannot find drivers until they ping again (45 s TTL). Reconstructed automatically. |

**Configuration:**

- **`maxmemory-policy` must NOT be an `allkeys-*` eviction policy.** Evicting a broker
  key loses a task silently. `noeviction` is correct: a full Redis then fails loudly,
  and every caller in this codebase now treats a Redis failure as an error rather than
  as a business answer.
- Persistence: AOF or RDB is preferable but not load-bearing — the outage matrix
  (`tests/test_redis_outage_matrix.py`) proves every ride stage survives Redis being
  entirely unreachable, and that Redis is reconstructed from PostgreSQL when it returns,
  in both directions, with no shell and no operator.
- Connection limit: the load run at 100 rides recorded `rejected_connections: 0`. A
  non-zero value in production means the limit was the ceiling, not the application.
- Monitor: `rejected_connections`, `evicted_keys` (must stay 0), memory headroom, and
  the Celery queue depth (`llen celery`).

**The property to preserve:** Redis must remain *reconstructable*. It is, and it is
tested. Do not add anything to Redis that PostgreSQL cannot rebuild.

---

## 7. S3 (§22)

- **Private buckets. No public read, ever.** The bucket holds driver KYC documents —
  licence, registration, identity — and rider receipts.
- **Separate buckets per environment.** QA and production must not share one: a QA test
  must not be able to read a real driver's licence, and a QA cleanup must not be able to
  delete one.
- **Access by presigned URL only**, short-lived. `AWS_QUERYSTRING_EXPIRE` controls the
  window; keep it minutes.
- **Least privilege.** The application credential needs `PutObject` and `GetObject` on
  its own bucket prefix and nothing else. Not `s3:*`. Not a second bucket.
- KYC and receipts should live under distinct prefixes so their access can diverge
  later without a migration.
- Server-side encryption on.
- `tests/test_presigned_uploads.py` and `tests/test_s3_endpoint_support.py` cover the
  application side. The bucket policy is the part a human must set.

**Nothing here has been verified against a real bucket.** It is the requirement.

---

## 8. Database credential (§23)

**Status: unchanged, and this remains RED until a human confirms otherwise.**

A database credential appeared in the repository's history. The standing position,
restated because it is the kind of thing that erodes: **removing a credential from HEAD
is not rotation.** Anything that has been in a git history must be treated as
compromised, because clones, forks, CI caches and local checkouts all still have it.

This cannot be resolved from here and must not be guessed at. The question for a human
with Railway access is narrow:

> Is the credential that appeared in git history still valid for any database that
> exists today — QA or production?

- **If yes:** production is RED regardless of every application readiness item in this
  package, and rotation is the first action. That is not an architectural defect; it is
  a credential operation.
- **If no** (the database was recreated, or the credential was already rotated): record
  *when*, and close the item.

Do not report this as an application concern. It is neither fixed nor fixable by code.

---

## 9. Maps credentials (§24)

**Status: RED. No work this run.**

`GOOGLE_MAPS_API_KEY` is a single variable read server-side. What is required before
either mobile app reaches a store:

| Key | Restriction |
|---|---|
| Android | Restricted to the app's package name **and** release signing certificate SHA-1. |
| iOS | Restricted to the bundle identifier. |
| Server | Restricted by IP to the backend's egress addresses, and by API to only the endpoints used (Geocoding, Directions). |

**Three keys, not one shared key.** A single unrestricted key embedded in a shipped APK
is extractable by anyone who downloads it, and the bill is the platform's.

Also set a billing quota and an alert. An unrestricted leaked key has produced
five-figure bills for other people, and a quota converts that into a broken feature —
which is recoverable.

The backend already rate-limits its own maps proxy per user
(`check_maps_rate_limit`, 100/hour), so the server-side exposure is bounded by the
application as well as by the key.

---

## 10. Cashfree (§25, §26, §27)

### The pilot position

**Online payments OFF. Automatic payouts OFF.** Cash only. That is the technical target
and nothing in this run moved it.

### The CVE chain, exactly (§27)

`pip-audit` against `requirements.txt`: **14 advisories across 2 packages.**

```
urllib3     2.0.7    PYSEC-2026-141, -1994, -1995, -1996, -1998, -1999
sentry-sdk  1.32.0   PYSEC-2026-1917
```

Both are held in place by **one pin**:

```
cashfree-pg==3.2.12
    requires urllib3<2.1.0,>=1.25.3
    requires sentry-sdk<1.33.0,>=1.32.0
```

Measured, not assumed:

- **The upgrade is otherwise clean.** With `urllib3==2.7.0` and
  `sentry-sdk[django]==1.45.1` installed, the full backend suite passes (860 at the time
  of measuring). So nothing in this codebase objects; only the SDK's declared pin does.
- **There is no fixed `urllib3` inside the Cashfree constraint.** Every advisory's fix
  version is above `2.1.0`. Pinning `urllib3==1.26.20`, which does satisfy the
  constraint, clears exactly one advisory of seven and moves onto the legacy 1.26 line:
  14 advisories become 12. That is not a fix and it was not applied.
- **The SDK surface is small.** `cashfree_pg` is imported in 7 places, all within
  `servers/payments/payment_gateways/cashfree_gateway.py` plus one availability check in
  `factory.py`.

`requirements.txt` is **unchanged**. Do not spend more time trying combinations; there
are none.

### Sandbox validation plan (§25)

The upgrade passing 860 tests proves nothing about provider behaviour, and that is the
whole point of this section. Before any SDK change, against Cashfree **sandbox**
credentials:

1. create a payout order — record the provider reference;
2. create a second with the **same `transferId`** — must be recognised as a duplicate,
   not a second transfer;
3. force a **timeout** — must become `unresolved`, never `FAILED`, and never an
   automatic refund;
4. **status lookup** on a known reference — must round-trip;
5. observe a genuine **pending** state and its transition;
6. observe a genuine **success**;
7. observe a genuine **failure**;
8. confirm the provider reference is stored and displayed for every outcome.

Only after that contract is demonstrated is an SDK upgrade a security fix rather than a
payments change wearing one as a disguise.

The application side of this is already built and tested:
`tests/test_payout_unknown_outcome.py`, `tests/test_payout_single_provider_call.py`,
`tests/test_payout_reconciliation_selection.py` — an ambiguous provider answer becomes
`unresolved`, is never presented as `FAILED`, and offers no blind Retry.

---

## 11. The order to do this in

Each step is safe to stop after.

1. **Change the production deploy trigger off `dev`** (§3). Resolve the staged delete.
   *Nothing else in this list is safe until this is done.*
2. **Answer the database credential question** (§8). If it is still live, stop and
   rotate.
3. Provision PostgreSQL 15 and Redis 7 in the production environment.
4. Set the variable set (§4), including `RAILWAY_DOCKERFILE_PATH` so it builds the
   artifact QA validates. Let the boot guards tell you what is missing — that is what
   they are for, and they report every fault at once rather than one per boot.
5. Create the `worker` and `beat` services (§2). Beat separate from the worker.
6. Set the Redis eviction policy to `noeviction` (§6).
7. Create the private S3 bucket with a least-privilege credential (§7).
8. **Take a backup and restore it** into a scratch database; confirm a trip reads back
   (§5). Record the time.
9. Restrict the Maps keys (§9). Three keys, quota, alert.
10. Deploy a **tagged** revision. Verify through `/version`, not `/healthz`.
11. `bootstrap_qa_operator`'s production sibling does not exist and must not: create the
    production operator through the Django admin, then enrol MFA with
    `manage.py enroll_operator_mfa`. Production enforces MFA with no override, so an
    operator without an enrolled factor cannot sign in — which is deliberate.

Steps 1, 2 and 8 are the ones that cannot be retrofitted after a real rider exists.
