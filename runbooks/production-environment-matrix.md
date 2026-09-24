# Production environment matrix

Compiled 2026-09-24 from the Railway API. **Names only — no value appears here, and
none should ever be added.**

QA backend service holds **30** application variables. Production holds **8**, all of
them injected by Railway (`RAILWAY_*`). Every row marked REQUIRED is missing from
production.

Classification:

- **REQUIRED** — the process will not boot, or a core journey cannot work.
- **SECRET** — REQUIRED and confidential. Set as a *sealed* Railway variable.
- **OPTIONAL** — a sensible default exists; set it deliberately anyway.
- **QA-ONLY** — must **never** exist in production. Boot now refuses.
- **PRODUCTION-ONLY** — has no place in QA.

## Django core

| Variable | Class | In QA | In prod | Note |
|---|---|---|---|---|
| `DJANGO_SECRET_KEY` | SECRET | yes | **no** | Boot refuses without it (guard added this run). Sessions and every signed value. |
| `ALLOWED_HOSTS` | REQUIRED | yes | **no** | **The current crash.** Comma-separated; no wildcard. |
| `DEBUG_ENV` | REQUIRED | yes | **no** | Must be `False`. Boot refuses if `True` in production. |
| `ENVIRONMENT` | REQUIRED | yes | **no** | Must be `production`. Drives the boot guards and Sentry tagging. Absent + `DEBUG=False` is treated as production. |
| `CSRF_TRUSTED_ORIGINS` | REQUIRED | yes | **no** | The ops console posts forms. |
| `CORS_ALLOWED_ORIGINS` | REQUIRED | yes | **no** | Without it the Next.js console cannot call the API. |
| `DJANGO_SECURE_SSL_REDIRECT` | REQUIRED | yes | **no** | Defaults to `False`. Set `True`. |
| `DJANGO_HSTS_SECONDS` | REQUIRED | yes | **no** | Defaults to `0`. Set a real value. |
| `PORT` | REQUIRED | yes | **no** | Railway supplies a value to bind. |

## Database

| Variable | Class | In QA | In prod | Note |
|---|---|---|---|---|
| `DB_HOST` | REQUIRED | yes | **no** | Absent selects the SQLite fallback — never in production. |
| `DB_NAME` | REQUIRED | yes | **no** | |
| `DB_USER` | REQUIRED | yes | **no** | |
| `DB_PASSWORD` | SECRET | yes | **no** | **Use a freshly rotated value.** The old one is in git history. |
| `DB_PORT` | OPTIONAL | yes | **no** | Defaults `5432`. |
| `DB_SSLMODE` | REQUIRED | yes | **no** | Default is `require`, which is correct. **Do not set `disable`.** |
| `DB_SSLROOTCERT` | OPTIONAL | no | no | Only if the CA bundle must be pinned. |

## Redis

| Variable | Class | In QA | In prod | Note |
|---|---|---|---|---|
| `REDIS_URL` | SECRET | yes | **no** | One URL serves four logical DBs: `/0` Celery broker, `/4` channel layer, plus geo/presence and streams. Use the `redis://:password@host` form; production Redis must require auth and must not be publicly reachable. |

## Celery / worker

| Variable | Class | In QA | In prod | Note |
|---|---|---|---|---|
| — | — | — | — | Celery reads `REDIS_URL`. Durability (`acks_late`, `reject_on_worker_lost`, prefetch 1, time limits) is now in code, not env. **Beat is still embedded in the worker via `-B` and must be split into its own process before a second worker exists.** |

## Object storage (S3)

| Variable | Class | In QA | In prod | Note |
|---|---|---|---|---|
| `AWS_ACCESS_KEY_ID` | SECRET | yes | **no** | Scope the IAM user to this bucket only. |
| `AWS_SECRET_ACCESS_KEY` | SECRET | yes | **no** | |
| `AWS_S3_BUCKET_NAME` | REQUIRED | yes | **no** | **Private bucket.** Holds KYC documents. |
| `AWS_S3_REGION` | REQUIRED | yes | **no** | `ap-south-1` for India. |
| `AWS_S3_ENDPOINT_URL` | OPTIONAL | yes | **no** | Only for a non-AWS S3. |

## Payments (Cashfree)

| Variable | Class | In QA | In prod | Note |
|---|---|---|---|---|
| `CASHFREE_WEBHOOK_SECRET` | SECRET | yes | **no** | `servers.payments.apps` refuses to start without it. |
| `CASHFREE_ENVIRONMENT` | REQUIRED | yes | **no** | `sandbox` or `production`. **Keep `sandbox` until the H6 suite passes.** |
| `CASHFREE_PAYOUT_APP_ID` | SECRET | no | no | Payouts are not wired for production. Leave unset to keep them off. |
| `CASHFREE_PAYOUT_SECRET_KEY` | SECRET | no | no | As above. |

## Auth / tokens

| Variable | Class | In QA | In prod | Note |
|---|---|---|---|---|
| `JWT_SIGNING_KEY` | SECRET | yes | **no** | Every rider and driver token. Distinct from `DJANGO_SECRET_KEY`. |
| `JWT_ISSUER` | OPTIONAL | no | no | Defaults `saaradhigo`. |

## SMS (OTP) and push (FCM)

| Variable | Class | In QA | In prod | Note |
|---|---|---|---|---|
| `AWS_SNS_SENDER_ID` | REQUIRED | — | **no** | Shown on the OTP SMS. In India a sender ID must be registered with DLT. |
| FCM credentials | REQUIRED | — | **no** | The backend uses `firebase-admin`. **Neither mobile app has a working FCM setup**: the driver app has no `google-services.json` at all, so it registers no token and receives no background ride offer. |

## Maps

| Variable | Class | In QA | In prod | Note |
|---|---|---|---|---|
| server-side Maps key | SECRET | — | **no** | Restrict by server IP. Distinct from the keys compiled into the apps. |
| app Maps key | — | — | — | **Compiled into the APK, not an env var.** One key is currently shared by Android and iOS, so it cannot be platform-restricted on both — rotate and split. |

## Email

| Variable | Class | In QA | In prod | Note |
|---|---|---|---|---|
| `DEFAULT_FROM_EMAIL` | REQUIRED | no | **no** | Falls back to `no-reply@PLATFORM_CONTACT_DOMAIN`. Receipts are sent from it, so the domain must have SPF/DKIM or receipts land in spam. |
| SMTP host / user / password | SECRET | no | **no** | Not currently configured anywhere. Receipt email is untested end to end. |

## Brand and legal

| Variable | Class | In QA | In prod | Note |
|---|---|---|---|---|
| `PLATFORM_BRAND_NAME` | OPTIONAL | no | no | Defaults `SaaradhiGo`. Set explicitly in production. |
| `PLATFORM_LEGAL_ENTITY` | REQUIRED | no | **no** | Appears on the GST receipt beside the MVA-2020 statement. **Its correctness is an unanswered business question.** |
| `PLATFORM_CONTACT_DOMAIN` | REQUIRED | no | **no** | Builds the grievance and fraud addresses given to drivers. Those mailboxes must exist and be monitored. |

## Logging and observability

| Variable | Class | In QA | In prod | Note |
|---|---|---|---|---|
| `DJANGO_LOG_FORMAT` | REQUIRED | yes | **no** | `json` in production so structured events are queryable. |
| `LOG_LEVEL` | REQUIRED | yes | **no** | `INFO`. `DEBUG` in production is a PII risk. |
| `SENTRY_DSN` | SECRET | no | **no** | Optional but strongly wanted; without it a crash is invisible. |
| `SENTRY_TRACES_SAMPLE_RATE` | OPTIONAL | no | no | Defaults `0.1`. |
| `DJANGO_RELEASE` | OPTIONAL | yes | no | **No longer the primary revision source.** `/version` prefers `RAILWAY_GIT_COMMIT_SHA`, which updates itself. |
| `BUILD_TIME` | OPTIONAL | no | no | Surfaced by `/version` if set at build time. |

## URLs

| Variable | Class | In QA | In prod | Note |
|---|---|---|---|---|
| `BACKEND_URL` | REQUIRED | yes | **no** | Absolute links in receipts and notifications. **Boot refuses if it points at QA, staging or localhost.** |
| `FRONTEND_URL` | REQUIRED | yes | **no** | Same guard. |

## Feature flags — decide, do not inherit

| Variable | Class | In QA | In prod | Note |
|---|---|---|---|---|
| `GPS_TRAIL_ENABLED` | REQUIRED | yes | **no** | Defaults `False`, so **no durable route evidence is stored**. Decide whether the pilot needs it for disputes, SOS investigation and insurance. |
| `FARE_SHADOW_ENABLED` | OPTIONAL | yes | **no** | Observation only; charges nobody. |
| `WALLET_TOPUPS_ENABLED` | REQUIRED | no | **no** | Must stay `False`. Closed-loop credit only. |
| `RIDER_CREDIT_BALANCE_CAP` | OPTIONAL | no | no | Defaults `2000.00`. |
| `DISPATCH_WAVE_SECONDS` / `DISPATCH_RADIUS_WAVES_M` | OPTIONAL | no | no | Tune per city after observing the pilot. |

## MUST NEVER EXIST IN PRODUCTION

Boot now refuses if any of these are set while `ENVIRONMENT=production`. QA has all
four, and production is meant to be configured by copying QA — which is exactly why
the guard exists rather than a note asking someone to remember.

| Variable | Class | Why |
|---|---|---|
| `TEST_PHONE_NUMBERS` | QA-ONLY | Each entry logs in with a **fixed OTP**, skipping SMS delivery *and* throttling. An authentication bypass. |
| `QA_ADMIN_BOOTSTRAP_PHONE` | QA-ONLY | Creates an admin account. |
| `QA_ADMIN_BOOTSTRAP_CODE` | QA-ONLY | As above. |
| `QA_ADMIN_BOOTSTRAP_PASSWORD` | QA-ONLY | A back door into payout approval and KYC. |

## How to apply this safely

1. **Repoint production off the `dev` branch first.** Populating variables while the
   service still auto-deploys `dev` turns a dormant misconfiguration into a live one.
2. Rotate `DB_PASSWORD` before setting it anywhere.
3. Create every SECRET row as a **sealed** Railway variable. Sealing afterwards does
   not un-expose a value that has already been readable, and
   `sealedVariableNames` is currently empty in both environments.
4. Set the REQUIRED rows, then attempt a boot. The guards report **all** faults at
   once rather than one per attempt.
5. Do not relax a guard to make a boot succeed. A guard with an override is a comment.
