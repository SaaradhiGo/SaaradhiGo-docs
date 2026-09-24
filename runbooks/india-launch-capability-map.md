# SaaradhiGo India launch capability map

- **Date:** 2026-09-24
- **Purpose:** what EXISTS, across all repositories, before anything new is built.
- **Method:** source inspection of all five repositories plus the deployed QA stack.
  Where a capability could not be executed (no Flutter SDK in this environment), that
  is stated rather than guessed.

## Why this document exists

The single most expensive mistake available here is building something that already
exists in another repository. The rider app is named `vahango` with the description
"A new Flutter project", which reads like a scaffold — and contains a wallet, trip
chat, promo service, favourite locations, service-area checks, ongoing-ride
notifications and ride-state recovery. Judging it by its pubspec would have been
wrong.

## Repositories

| Repo | Stack | Branch / HEAD | CI | Tests | Deploy target |
|---|---|---|---|---|---|
| `SaaradhiGo-backend` | Django 5.2.17, DRF, Channels 4.2, Daphne 4.2.3 | `dev` / `b3444d7` | `deploy.yml`: lint, SQLite suite, postgres-per-file, EC2 deploy | **695 collected** (598 default + 88 postgres-marked + 9 skipped) | Railway QA (live); EC2 workflow present, owner undetermined |
| `SaaradhiGo-driver` | Flutter, Riverpod | `feat/driver-app-rebuild` / `ca6fbe9` | `flutter.yml`: analyze + test | **74 passing** (verified on GitHub Actions) | Android; iOS unconfirmed |
| `SaaradhiGo-mobile` (rider) | Flutter, Provider | `dev` / `7d5846d` | **none** | ~48 across 8 files, never run in CI | Android + some web support |
| `SaaradhiGo-web` | Next.js 15.5.26 / React 19, monorepo | `develop` / `a4ec60b` | **none** | **none** (no `test` or `typecheck` script) | undetermined — no vercel/netlify/railway config |
| `SaaradhiGo-docs` | Markdown | `fix/promo-fare-integrity` / `44278de` | n/a | n/a | n/a |

### Duplicate / stub applications found

**Two operations consoles exist, both actively developed.** This is a product
decision that needs a human, not a unilateral engineering choice:

| | Backend Django console | Next.js ops console |
|---|---|---|
| Location | `SaaradhiGo-backend/templates/admin_pages/` | `SaaradhiGo-web/apps/web` |
| Size | 19 templates, 6,619-line `views.py`, 21 routes | 14 screens |
| Auth | Django session login | `/login` against the API |
| Coverage | fleet monitor, rides, ride detail, riders, driver onboarding, driver profile, payments, transactions, disputes, emergency, promo codes, fare/surge, global search, notifications, executive revenue, driver loyalty, predictive heatmaps | dashboard, drivers, driver detail, trips, trip detail, SOS, support, support detail, withdrawals, zones |
| Recency | grown over time | newest commits in the org |
| Tests/CI | covered by backend suite | none |

The Django console is broader (and contains clearly aspirational screens —
`predictive_heatmaps`, `driver_loyalty`, `executive_revenue`). The Next.js console is
narrower but is where recent operations work landed (SOS triage, withdrawal approval,
support replies). **Running two is an operational hazard during a pilot**: an
operator who approves a payout in one and looks for it in the other will conclude the
platform is broken.

`SaaradhiGo-web/apps/android` and `apps/ios` contain only a `README.md` — stubs, not
abandoned applications.

## Backend API surface

120 routes across nine Django apps. Not a thin layer.

| App | Routes | Notable |
|---|---|---|
| `driver` | 27 | KYC, documents, vehicle, earnings, withdrawals |
| `ride` | 22 | estimate-fare, active, trip detail, cancel, driver-cancel, receipt PDF + resend, chat, rate-trip, promo/apply, maps proxy (geocode / reverse-geocode / place-details / directions) |
| `admin_dashboard` | 21 | trips, live locations, payments, transactions, dashboard |
| `rider` | 12 | wallet balance/transactions/payment, saved locations, notifications + preferences |
| `auth_user` | 11 | OTP, login, refresh, profile, DPDP data export/delete |
| `payments` | 11 | create-order, webhook, refund, retry, switch, payout-webhook |
| `support` | 8 | rider tickets + messages + close; admin list/assign/reply |
| `sos` | 5 | raise, admin list, acknowledge, resolve, false-alarm |
| `pricing` | 3 | admin rate cards |

## What is already implemented (do not rebuild)

Capabilities confirmed present in source, many verified end-to-end in QA during
previous runs:

- **Ride lifecycle** with a strict transition table under `SELECT FOR UPDATE`, and
  correlated application-level `command_ack` (committed / already_done / rejected).
- **Booking idempotency** on a per-rider `client_request_id` with a PostgreSQL
  partial unique index.
- **Reconnect recovery**: the trip socket's greeting carries durable `trip_status`
  read from PostgreSQL, proven against a deliberately corrupted Redis cache.
- **GPS**: Redis stream → sampled durable `TripLocationPoint`, bounded at 12
  points/minute and capped per trip; raw trails reachable by no API.
- **Fare engine**: single `quote_fare`, zone + RateCard resolution, surge, night
  surge, min fare, surge cap. RateCard pricing immutable once created.
- **Fare shadow**: observe-only comparison, reads no money path.
- **Payments**: Cashfree order creation, webhook, refund, retry, method switch; cash
  and wallet paths.
- **Wallet ledger**: `WalletTransaction`, idempotent on `TRIP_<id>_EARNING`.
- **Receipts**: versioned, PDF, S3, resend endpoint.
- **SOS**: durable-first, fan-out `on_commit` to Celery, repeat collapsing that fails
  open, admin acknowledge/resolve/false-alarm.
- **Support tickets**: rider-facing create/list/message/close, admin assign/reply.
- **Notifications**: FCM push, in-app notification list, per-user preferences.
- **Maps**: server-side proxy for geocode, reverse-geocode, place details, directions
  — so the API key is not solely in the client.
- **DPDP**: data export and deletion endpoints for a user's own data.
- **Rider app**: login/OTP, splash, home with map, precise-pickup screen, route map,
  history, wallet + add money, payment status/success, notifications, trip chat,
  profile + privacy/security, help & support, cancelled overlay, payment-pending
  banner, sync overlay, ride-state recovery service, service-area service.
- **Driver app**: login/OTP, duty (online/offline) controller, dispatch offer
  controller, trip controller with the full lifecycle, earnings, withdrawals,
  profile/KYC, history, `CommandDispatcher` with ack + bounded retry, self-healing
  `WsClient` with app-level keepalive.

## Gaps found during inventory

Recorded here; acted on in priority order elsewhere in this run.

| # | Gap | Severity for a pilot |
|---|---|---|
| 1 | **Both mobile apps default to the QA API.** Rider: `dotenv.env['BASE_URL'] ?? 'https://dev.api.saaradhigo.in/...'`. Driver: `String.fromEnvironment(..., defaultValue: 'https://dev.api...')`. A production build that omits the variable ships silently pointed at QA. | **Launch-blocking if unfixed** |
| 2 | **Rider app has a hardcoded Google Maps API key committed** in `android/app/src/main/AndroidManifest.xml`. The driver app already does this correctly with `${MAPS_API_KEY}`. | High (billing abuse, and the right pattern already exists next door) |
| 3 | Two operations consoles, both live-developed. | High (operational confusion during an incident) |
| 4 | **Rider app has no CI.** ~48 tests exist and have never been run automatically. | High |
| 5 | **Ops console has no CI, no tests, and no typecheck script.** | High |
| 6 | Settlement economics still recomputed at render time; `TripSettlement` not implemented. | High (disputes) |
| 7 | `feat/fare-snapshot-schema` unmerged, so historical fares are not permanently explainable. | High |
| 8 | Bundle IDs inconsistent: `com.deeptrics.saaradhigo` (rider) vs `com.saaradhigo.driver`. | Medium (store/org hygiene) |
| 9 | Rider app last touched 2026-09-10; driver app 2026-09-23. The rider app is the less-maintained half of a two-sided product. | Medium |
| 10 | 8 of 88 postgres tests fail in a single process; CI works around it per-file. | Medium (unexplained) |
| 11 | Celery Beat still embedded (`-B`); per-task durability audit not done. | Medium |
| 12 | Cashfree SDK three majors behind and its pins block 15 security advisories. | Blocked (needs sandbox) |

## Environments

| | Backend | Rider | Driver | Ops console |
|---|---|---|---|---|
| local | SQLite fallback / local PG | `assets/.env` (gitignored) | `--dart-define` | `localhost:8000` default |
| QA | Railway `backend-qa-811d.up.railway.app` | default fallback | default fallback | — |
| production | not deployed | **same fallback as QA** | **same fallback as QA** | not deployed |

The rider app bundles `assets/.env` as a Flutter asset, so whatever `.env` sits on
the build machine is what ships. That is a build-hygiene dependency, not a code
guarantee.

## What could not be executed in this environment

Stated plainly so nothing below is over-claimed:

- **No Flutter/Dart SDK is installed**, so neither mobile app could be built or run.
  Mobile findings in this run are source-level, plus whatever GitHub Actions can
  execute. The driver app has CI and is therefore genuinely verified; the rider app
  gets CI in this run for the same reason.
- **No Node/npm run of the ops console** was attempted beyond static inspection.
- **No Cashfree sandbox credentials**, so provider-contract behaviour remains
  unverified.
