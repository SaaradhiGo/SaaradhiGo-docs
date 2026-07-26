# Phase-0 launch-readiness checklist

A short, opinionated punch-list of what must be true on launch day for
the Phase-0 cities (Hyderabad + Vijayawada + Warangal + Visakhapatnam),
what's already shipped, and what still needs founder action. Updated
against the merged code in dev.

## Shipped engineering (no further action)

### Launch-hardening batch (ADR-0007)
- [x] **Trip transaction holds no network I/O.** Receipt render + S3 +
  email moved to `ride.issue_receipt_for_trip` (Celery); pushes and Redis
  writes go through `transaction.on_commit`; the Cashfree order is no
  longer created inside the completion transaction.
- [x] **Trip-group IDOR closed.** Only the rider and the assigned driver
  join `trip_<id>`; candidate drivers can accept but never subscribe, and
  losers get an explicit `trip_taken` + socket close.
- [x] **Dispatch rebuilt.** Per-vehicle-type Redis geo keys (a sedan
  request no longer misses sedans behind 50 nearer bikes), rolling
  1500/3000/5000m waves, 90s accept timeout (was 600s), presence
  heartbeats + a 60s ghost-driver sweeper, and no Postgres read per
  location ping.
- [x] **Commission priced per zone** from `RateCard.commission_percent`
  as of the trip's request time. The old path read a global env var that
  shipped defaulting to `0`.
- [x] **Driver settlement split from rider credits** — `Wallet.scope`
  with a unique `(user, scope)` constraint + backfill migration.
- [x] **Cancellation attribution** (`cancelled_by`, `cancellation_reason`,
  `cancellation_fee`) written by all three cancel paths and auto-cancel.
- [x] **`/ride/trip/<id>/details/` authorised.** It previously served
  driver name, phone and number plate to any authenticated caller who
  guessed a trip id.
- [x] **Maps key off the device.** New `/ride/maps/place-details` and
  `/ride/maps/reverse-geocode` proxies; the app no longer calls Google
  directly, no longer relays through `corsproxy.io`, and no longer routes
  via the public OSRM demo server.
- [x] **Webhook parity** — trip payments now re-verify `order_status` and
  amount with Cashfree before settling, as wallet top-ups already did.
- [x] **CI gates**: ruff + bandit + pip-audit + a missing-migration check,
  a Redis service for the suite, and a deploy that runs migrations as a
  discrete step with a health gate and automatic rollback.
- [x] **Redis is no longer world-reachable**: loopback binding,
  `requirepass`, AOF persistence, `noeviction`.

### MVA 2020 compliance
- [x] **Service-area enforcement.** Pickup + drop must sit inside an
  active `ServiceZone` polygon. See
  [ADR-0002](../adr/0002-multi-city-pricing.md).
- [x] **Surge cap 1.5×.** Enforced centrally per zone in
  `pricing.services.quote_fare` via
  `RateCard.surge_cap_multiplier`.
- [x] **Driver fatigue cap (12h / 24h).** `DriverSession` ledger +
  `Driver.fatigue_lockout_until` + accept-time gate in
  `consumers._accept_trip`. 3 cancels in 24h triggers a 1h online
  lockout via the same column.
- [x] **Vehicle credential expiries.** Insurance / permit / fitness /
  PUC expiry fields on `Vehicle`; daily Celery job blocks any driver
  whose active vehicle has an expired credential.
- [x] **Driver KYC gate.** `Driver.approved` re-checked inside the
  trip-accept lock; admin endpoint refuses to approve until all
  required documents + expiry dates are present.
- [x] **SOS / panic button.** `servers/sos/` is wired with
  `SOSEvent` model + `/sos/` POST + `/sos/<id>/acknowledge/` +
  `/sos/<id>/resolve/` + `/sos/<id>/false-alarm/`. FCM push to ops
  (is_staff=True) + SMS to the rider's emergency contact.

### DPDP Act 2023 compliance
- [x] **Privacy Policy** drafted at
  [legal/privacy-policy.md](../legal/privacy-policy.md).
- [x] **/me/export** + **/me/delete** endpoints
  (data portability + anonymisation).
- [x] **Notification preferences.** `NotificationPreference` model;
  marketing + promo default OFF; safety/transactional/payout locked
  on. Endpoints under `/api/v1/rider/notifications/preferences/`.
- [x] **Aggregator data localisation.** Primary DB + S3 in
  `ap-south-1` (Mumbai).

### GST + tax
- [x] **CGST §31 invoicing.** `Receipt` model + HTML email + **PDF
  attachment** on trip-complete + resend endpoint
  (`/api/v1/ride/trip/<id>/receipt/resend/`) + PDF URL endpoint
  (`/api/v1/ride/trip/<id>/receipt/pdf/`). GST captured per
  RateCard at issue time.
- [x] **TDS u/s 194O guide** at
  [legal/gst-tds-registration-guide.md](../legal/gst-tds-registration-guide.md).

### Payments
- [x] **Cashfree integration + webhook fail-closed + replay guards.**
- [x] **Closed-loop wallet posture.** External top-ups disabled by
  default; refund-to-credit mode available; see
  [ADR-0003](../adr/0003-closed-loop-wallet.md).
- [x] **Driver payouts via Cashfree Payouts.** Withdrawal recon
  Celery job sweeps stuck states every 10 min.

### Ops
- [x] **Driver payout approval screen** at `/withdrawals` in the ops
  console. Shows payee name, phone, UPI handle and KYC state before
  release; Approve is disabled for a driver whose KYC is not approved.
  Backed by the existing maker-checker
  `/driver/admin/withdrawals/<id>/approve|reject/` endpoints, which had
  no UI until now.
- [x] **SOS triage queue** at `/sos` in the ops console, backed by a new
  `GET /api/v1/sos/admin/`. Sorts unacknowledged-first, polls every 15s,
  flags any open event older than the 5-minute response target, and
  one-taps to the caller's phone and map location. Acknowledge / resolve /
  false-alarm write immutable `SOSEventUpdate` rows as before.
- [x] **/healthz** probe.
- [x] **JSON logs.**
- [x] **Sentry** wired (Django + Celery + Redis).
- [x] **Ops admin dashboard.** `/api/v1/ride/admin/dashboard/` returns
  daily trips by status, GMV, drivers online now/24h, riders active
  24h, driver cancels 24h, withdrawals pending/completed, receipts
  issued / failures.
- [x] **Support tickets.** `SupportTicket` + `SupportMessage` thread
  model with rider + admin endpoints under `/api/v1/support/`.
- [x] **CI test gate** before EC2 deploy; 114 tests + 9 skipped on
  dev today.

### Mobile
- [x] **Phone-call privacy.** Rider app uses OS dialer (`tel:` deep
  link) — driver phone never displayed in the UI.
- [x] **Foreground service** for trip-in-progress (India OEM
  compatibility).
- [x] **VahanGo Credits UI** with server-flag-driven Add Money button.
- [x] **In-trip chat** (`lib/screens/home/trip_chat_screen.dart` +
  `chat_service.dart`). See [ADR-0004](../adr/0004-in-trip-chat.md).
- [x] **Promo code entry** widget (`lib/widgets/promo_code_field.dart`).

### Web ops console (Phase-0 MVP)
- [x] **Next.js 14 + Tailwind app** at `apps/web/` in
  [SaaradhiGo-web](https://github.com/SaaradhiGo/SaaradhiGo-web).
- [x] OTP login + Dashboard + Trips + Drivers (with KYC approve) +
  Support tickets + Zones pages.
- [x] **Trip detail page** at `/trips/<id>`. Backend aggregate at
  `GET /api/v1/ride/admin/trips/<id>/` returns rider + driver +
  vehicle + fare breakdown + payments + ratings + receipts (with
  PDF links) + chat thread + driver cancellations + SOS events +
  promo redemption in one round-trip. Header has refund (to-credits
  or to-original) + resend-receipt actions.
- [x] **Driver detail page** at `/drivers/<id>`. Backend aggregate
  at `GET /api/v1/driver/admin/<id>/full/` returns user + KYC +
  fatigue status + all vehicles (with credential expiries colour-
  coded) + sessions + cancellation counters + withdrawals + recent
  trips + earnings. Header has KYC approve / revoke actions.
- [x] **Support ticket detail + reply** at `/support/<id>`. Renders
  the thread with role-aware styling, posts replies via
  `/support/admin/tickets/<id>/reply/` with optional state change,
  assigns staff via `/support/admin/tickets/<id>/assign/`.
- [ ] Phase-1: CSV export, refresh-token rotation, live map,
  two-factor for admin login, audit log read view. See
  [ADR-0005](../adr/0005-ops-web-console.md).

### Multi-city
- [x] **Hyderabad** seeded.
- [x] **Vijayawada (IN-AP-VJA)**, **Warangal (IN-TG-WGL)**, and
  **Visakhapatnam (IN-AP-VTZ)** seeded with regional rate cards.
  See [ADR-0006](../adr/0006-multi-city-expansion-vja-wgl-vtz.md).

### Promos + rider rating
- [x] **PromoCode + PromoRedemption** models + apply endpoint
  (`POST /api/v1/ride/promo/apply/`) + admin CRUD via Django admin.
  Supports percent + flat discounts, min-fare gate, zone scope,
  per-user + global redemption caps.
- [x] **Rider rating decay** via
  `servers.rider.rating_decay.apply_rider_rating`. Below 3.0 ->
  `flagged_for_review`; below 2.5 -> soft-block hint surfaces in the
  driver-side payload so drivers can decline without penalty.

---

## Required founder actions before public launch

These cannot be done by engineering. Listed by hard deadline.

### Legal
- [ ] **30-minute fintech-lawyer consult** on the closed-loop credits
  framing in [legal/credits-policy.md](../legal/credits-policy.md).
  ~₹25-50K. Required by ADR-0003 before \`WALLET_TOPUPS_ENABLED\` is
  ever flipped on.
- [ ] **Review all `legal/*.md` drafts with Indian-qualified counsel**
  (consumer / IT / MV / data protection). Each file lists the
  required placeholders.
- [ ] **Designate a Grievance Officer** per IT Rules 2021 (name,
  email, response SLA). Add to Privacy Policy + Terms of Service.

### Government registrations
- [ ] **Company CIN** confirmed.
- [ ] **GSTIN** issued (the application is a multi-step flow; see
  [legal/gst-tds-registration-guide.md](../legal/gst-tds-registration-guide.md)).
- [ ] **TAN** issued for TDS u/s 194O.
- [ ] **Professional Tax** registration.
- [ ] **Shops & Establishments** registration.
- [ ] **GHMC Trade Licence** (Hyderabad municipal).
- [ ] **Telangana RTO aggregator licence** application per MVA 2020.

### Operational
- [ ] **Phone-masking proxy** signed with Exotel (recommended) or
  Knowlarity. ~₹0.40-0.80 per minute. Phase-1.
- [ ] **Cashfree production** keys + webhook secret + escrow account.
- [ ] **Comprehensive passenger insurance** product chosen (Tata AIG /
  ICICI Lombard) — MVA 2020 expects per-trip cover.
- [ ] **Background check vendor** chosen for drivers (CrimeCheck /
  AuthBridge / SignzyOnboard).
- [ ] **Police verification** workflow for each driver (state-RTO
  channel).
- [ ] **First 50 drivers** sourced + onboarded through the SOP at
  [legal/mva-2020-driver-verification-sop.md](../legal/mva-2020-driver-verification-sop.md).
- [ ] **App Store / Play Store** developer accounts active +
  privacy disclosure tables filled in.
- [ ] **Customer support staffing**: at least one person on
  WhatsApp / phone during operating hours.
- [ ] **24×7 SOS responder** designated (per MVA 2020).

### Infrastructure
- [ ] **Daily automated database backups** + a tested restore.
- [ ] **Rotate the leaked Postgres password** (`PossibleMe2025`, previously
  hardcoded in `docker-compose.override.yml`) on RDS and scrub it from git
  history. The compose file now reads it from `.env.local`, but the value
  is still in every clone's history.
- [ ] **Rotate the Google Maps API key** shipped in past app builds and
  restrict the new one to the backend's server IPs. Any key that has been
  inside a released APK is public.
- [ ] **Set `REDIS_PASSWORD` and the `redis://:PASSWORD@host` form of
  `REDIS_URL`** in `.env.prod`, plus `JWT_SIGNING_KEY` and the
  `EC2_HOST_KEY` GitHub secret (from `ssh-keyscan`).
- [ ] **CloudWatch / Grafana alerts** for: error-rate spike, payment
  webhook failures, driver oncall queue depth, SOS event creation.
- [ ] **AWS Secrets Manager** for production secrets (currently
  env-driven).
- [ ] **Production environment promotion**: switch
  `DEBUG_ENV=False`, set `ALLOWED_HOSTS`,
  `CORS_ALLOWED_ORIGINS`, `DJANGO_SECURE_SSL_REDIRECT=True`
  (after confirming nginx forwards `X-Forwarded-Proto`).

---

## Out of scope for Phase-0 (Phase-1 backlog)

* Phone-masking proxy (Exotel/Knowlarity) integration.
* Co-branded PPI / own RBI PPI licence (for real wallet top-ups).
* Multi-city expansion beyond the 4 Phase-0 cities (Bangalore +
  Chennai + Mumbai are the obvious Phase-1 candidates).
* ML-based surge pricing (Phase-3, ≥6 months of trip data).
* Ops console refresh-token rotation + CSV export + live map +
  two-factor for admin login + audit log read view.
* Sustained-load + chaos testing.

Track these in the project tracker; do not block Phase-0 on them.
