# Production environment: what is actually there

Gathered 2026-09-24 against the live Railway project and GitHub. Every claim
below is from a tool response, not from reading configuration files.

No secret values appear in this document. Variable and secret **names** are
listed because the gap is which names exist, not what they hold.

## The headline

**There is no working production environment.** It exists as a Railway
environment with one service, it has never been configured, and it crash-loops on
startup.

```
django.core.exceptions.ImproperlyConfigured:
    ALLOWED_HOSTS must be set (comma-separated) when DEBUG=False.
  File "/app/base/settings.py", line 28, in <module>
```

That is from the deploy logs of production's most recent **successful build**
(deployment `795b8186`, 2026-09-23T15:08:32Z). The build succeeded; the process
could not start.

Environment status for `production` (`169a60b3-2de2-4568-9f3a-8a5ffcf44732`):

```
service            SaaradhiGo-backend (2d8a3755-…)
replicaStatus      crashed: 1, running: 0, total: 1
recentFailures     8
warnings           30
last 3 deploys     FAILED, FAILED, FAILED
staged patch       EnvironmentPatch pending since 2026-09-22T20:35:47Z
```

The boot guard in `base/settings.py` is behaving **correctly**. It refuses to
serve rather than falling back to `ALLOWED_HOSTS = ['*']`. This is a
configuration gap, not a code defect, and the guard is the reason it is loud
instead of silent. Do not weaken it to get production up.

## Production has zero application variables

Railway reports these variable names for the production backend service:

```
RAILWAY_ENVIRONMENT      RAILWAY_PROJECT_ID
RAILWAY_ENVIRONMENT_ID   RAILWAY_PROJECT_NAME
RAILWAY_ENVIRONMENT_NAME RAILWAY_SERVICE_ID
RAILWAY_PRIVATE_DOMAIN   RAILWAY_SERVICE_NAME
```

All eight are injected by Railway. **Not one application variable is set.** No
database, no Redis, no signing keys, no payment configuration.

For comparison, QA's backend service has 30 application variables. The exact set
production is missing:

| Variable | Why production cannot start or cannot work without it |
|---|---|
| `ALLOWED_HOSTS` | The current crash. Django refuses to boot with `DEBUG=False`. |
| `DJANGO_SECRET_KEY` | Session and signing integrity. Required by the boot guard. |
| `DB_HOST` `DB_NAME` `DB_USER` `DB_PASSWORD` `DB_PORT` `DB_SSLMODE` | No database. `DB_SSLMODE` defaults to `require`, which is right for production — do not set it to `disable`. |
| `REDIS_URL` | Dispatch, driver presence, the channel layer and Celery all depend on it. |
| `JWT_SIGNING_KEY` | Every rider and driver token. |
| `CASHFREE_WEBHOOK_SECRET` | `servers.payments.apps` refuses to start without it. |
| `CASHFREE_ENVIRONMENT` | Decides sandbox vs live money. |
| `AWS_ACCESS_KEY_ID` `AWS_SECRET_ACCESS_KEY` `AWS_S3_BUCKET_NAME` `AWS_S3_REGION` `AWS_S3_ENDPOINT_URL` | KYC documents and receipts. Without these, driver documents have nowhere to go. |
| `CORS_ALLOWED_ORIGINS` `CSRF_TRUSTED_ORIGINS` | The operations console cannot talk to the API. |
| `DEBUG_ENV` | Must be `False`. |
| `ENVIRONMENT` | Sentry environment tagging. |
| `DJANGO_SECURE_SSL_REDIRECT` `DJANGO_HSTS_SECONDS` | Transport security. Both are permissive by default. |
| `DJANGO_LOG_FORMAT` `LOG_LEVEL` `DJANGO_RELEASE` | Structured logs and release correlation. |
| `BACKEND_URL` `FRONTEND_URL` `PORT` | Absolute links in receipts and notifications. |
| `GPS_TRAIL_ENABLED` `FARE_SHADOW_ENABLED` | Feature flags; decide deliberately rather than inheriting a default. |

### Two names that must NEVER be set in production

- **`TEST_PHONE_NUMBERS`** — QA has it. It exists so QA logins can bypass real
  OTP delivery. In production it is an authentication bypass. A launch checklist
  must assert this is unset, not merely assume it.
- **`QA_ADMIN_BOOTSTRAP_PHONE` / `QA_ADMIN_BOOTSTRAP_CODE` /
  `QA_ADMIN_BOOTSTRAP_PASSWORD`** — QA has all three. They create an admin
  account. In production that is a back door.

## Production deploys from the `dev` branch

The production Railway service is wired to `branch: dev`. Its failed deployment
`b5450d1a` carries `commitHash 670cf5a` — a commit pushed to `dev` minutes
earlier during ordinary development.

So **every push to a development branch targets production.** There is no release
branch, no staging gate, and no human step between a developer's push and a
production deploy attempt. The only reason this has not caused an incident is
that production has never started successfully.

This must be changed before production is configured, not after. Configuring the
variables while the service still tracks `dev` converts a dormant misconfiguration
into a live one.

## The GitHub Actions deploy job has never worked

`.github/workflows/deploy.yml` has a `deploy` job targeting EC2. It fails on every
push, at the `Setup SSH` step:

```
##[error]known_hosts is empty. Populate the EC2_HOST_KEY secret or
ensure EC2_HOST is reachable.
```

Cause, confirmed by listing the repository's secret **names**:

```
EC2_HOST      2026-05-07
EC2_SSH_KEY   2026-05-07
EC2_USER      2026-05-07
```

`EC2_HOST_KEY` **does not exist**. The workflow references
`secrets.EC2_HOST_KEY`, an unset secret evaluates to an empty string, so the step
falls through to `ssh-keyscan -H "$EC2_HOST"` — which produced no output, meaning
**the EC2 host did not answer on port 22 from the runner**. The guard then
correctly refuses to continue rather than disabling host verification.

Two separate problems, and the second is the serious one:

1. The `EC2_HOST_KEY` secret was never created.
2. The EC2 host appears unreachable. Either the instance is stopped or gone, or
   its security group does not admit GitHub runners.

This also means the three green test gates (`test`, `lint`,
`postgres-concurrency`) have been passing while the overall workflow reported
**failure** on every push since at least `dc3546c`. A red pipeline that is always
red teaches people to ignore it, and it would hide a real test failure.

Decide which deployment path is real — Railway or EC2 — and delete the other.
Having both, with one permanently broken, is worse than having one.

## Secrets are not sealed

Railway reports `sealedVariableNames: []` for **both** QA and production. Every
secret QA holds — `DJANGO_SECRET_KEY`, `DB_PASSWORD`, `AWS_SECRET_ACCESS_KEY`,
`CASHFREE_WEBHOOK_SECRET`, `JWT_SIGNING_KEY`, `QA_ADMIN_BOOTSTRAP_PASSWORD` — is
an ordinary variable, readable by anyone with project access.

Production should use sealed variables from the start. Sealing after the fact
does not un-expose a value that has already been readable.

## SECURITY INCIDENT: a database password is in git history

Verified, not inferred:

```
commit  9af854c  ("first")  2026-05-07
file    docker-compose.override.yml, line 34
key     POSTGRES_PASSWORD
```

The value is a literal, not a `${VAR}` reference. The value is **not reproduced
here**.

The commit is reachable from `dev` and from at least five other branches, so it is
present in every clone and on GitHub. Exposure window: **2026-05-07 to now, about
four and a half months.**

The current `docker-compose.override.yml` has already been fixed to read
`${DB_PASSWORD}`, and its comment states this literal is also the production
database password. **That fix does not end the exposure.** Anyone who has ever
cloned this repository still has the value.

### Required, in this order

1. **Rotate the credential** on every system where it is valid — the RDS
   instance, the Railway Postgres service, and any local or CI configuration.
   Rotation is what ends the exposure. Nothing else does.
2. Update the consuming configuration to the new value, sealed where Railway
   supports it.
3. Only then consider history rewriting, and treat it as tidying rather than
   remediation. Rewriting history **before** rotating would leave a live
   credential in every existing clone while creating the impression the problem
   was handled.
4. Check whether the exposed credential was ever used from outside the VPC. If
   the database was ever publicly reachable, the rotation is a containment step
   in an incident rather than hygiene.

Do not mark this resolved because the current tree is clean. The current tree
being clean is what makes it easy to forget.

## Ordered remediation

Nothing here needs new code. It needs decisions and configuration.

1. **Rotate the leaked database password.** Everything else can wait; this
   cannot.
2. **Repoint the production Railway service off `dev`** to a release branch or a
   manual trigger. Do this before step 3.
3. **Populate production's variables**, sealed, from the QA list above, with
   production values — and with `TEST_PHONE_NUMBERS` and every
   `QA_ADMIN_BOOTSTRAP_*` name deliberately absent.
4. **Resolve the staged `EnvironmentPatch`** that has been pending since
   2026-09-22, so the environment's intended state and actual state agree.
5. **Pick one deploy path.** Either create `EC2_HOST_KEY` and restore the host's
   reachability, or delete the `deploy` job and let Railway own deployment. A
   permanently failing job is not a deploy path.
6. **Confirm database backups exist and have been restored from at least once.**
   Not covered by this pass, and an untested backup is a hope, not a backup.
7. Only then attempt a production boot, and expect the boot guards to tell you
   what is still missing — that is what they are for.

## What this document does not cover

- Whether database backups are configured, and whether a restore has ever been
  rehearsed. Unverified.
- The `alluring-happiness` service in this project, which has an auto-generated
  Railway name and an unexamined purpose.
- Why production's most recent **builds** fail at `BUILD_IMAGE`. The startup
  crash above is from the last build that succeeded; the build failures are a
  second, separate question.
- Whether the EC2 host still exists at all.
- Any load or capacity testing. None was performed.
