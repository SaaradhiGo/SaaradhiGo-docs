# India + USA go-live — code review and requirements

Written for: the CTO/product owner and the engineering team who will clear these items.

**Reviewed at:** backend `b6be1e8`, rider `69598ae`, driver `bae0a3e`, web `fcf56fc`,
all unchanged since 2026-09-24. Backend: 63,499 lines of Python across 5 repos.

This is a fresh code review, not a restatement of the pilot reports. Where a prior
report covers something, it is cited rather than repeated — and in one place below, this
review **corrects** a prior recommendation.

---

## 0. Three corrections to the existing record

### Backend CI has been red for three weeks, and not for the reason it looks like

Every push since 2026-09-25 shows a failed workflow. The test jobs are **green**:

```
success  lint
success  postgres-concurrency
success  test
failure  deploy      ← EC2_HOST_KEY secret not set — ssh-keyscan against EC2_HOST
                       produced nothing, exit 1
```

So the code gates pass and have passed throughout. What fails is a deploy job on a
missing secret against an unreachable host. The practical harm is a red badge that
trains everyone to ignore CI — which is how a real failure gets missed.

### There are THREE deployment targets, and two are broken

| Target | Mechanism | State |
|---|---|---|
| Railway QA | `backend` service, branch `dev` | **Works.** Serving `b6be1e8`. |
| Railway production | `SaaradhiGo-backend` service, branch `dev` | Empty shell, marked deleted, zero variables, no domain. 12 builds, all failed. |
| **EC2** | `.github/workflows/deploy.yml` → SSH → `docker-compose.prod.yml` | **Complete design, currently failing** on the host key. |

Nothing in the repo says which is authoritative. That is a decision somebody has to
make, and it is the first item on the production list.

### The prior production package recommended the wrong shape

`india-pilot-production-package.md` specified a Railway topology. It was written without
knowing `docker-compose.prod.yml` and the EC2 deploy job existed — and that path is
**further along** than the Railway one:

```yaml
django:   bound to 127.0.0.1:8000 only (nginx terminates TLS on the host)
          migrations NOT in the container command — run as a one-shot deploy step
redis:    --requirepass, --appendonly yes, --appendfsync everysec,
          --maxmemory-policy noeviction, persistent volume, loopback only
celery:   -B embedded, with a comment that beat must stay at one replica
deploy:   build → one-shot migrate → up -d → 30×10s health gate →
          automatic rollback to the previous SHA on failure
```

Three things my package listed as *requirements* are already **implemented** there:
`noeviction`, AOF persistence, and loopback binding. The health-gate-with-rollback is
better than anything I specified.

Two gaps in it: **there is no PostgreSQL service** (an external database is assumed, and
which one is undocumented), and the nginx/TLS config lives on the host rather than in the
repo, so it is not reviewable or reproducible.

**Recommendation: adopt the EC2/compose path as the production target and retire the
Railway production environment**, unless there is a business reason for Railway. It is
the more complete design, and maintaining two is how they both rot.

---

## 1. What is already market-portable (the good news)

Reviewing for USA readiness, four pieces of the architecture turned out to be built for
it already. These are not accidents and they save months.

**`ServiceZone` is a genuine four-level geographic hierarchy** —
`servers/pricing/models.py:104-165`:

```python
country    = CharField(max_length=2, default='IN')      # ISO 3166-1 alpha-2
state_code = CharField(...)                             # ISO 3166-2, e.g. "TG", "CA"
city       = CharField(...)
zone_type  = country | state | city | subzone
```

A US zone is a row, not a schema change. The index
`(country, state_code, city)` is already there.

**Pricing is per-zone, not global** — `RateCard` carries `base_fare`, `per_km_fare`,
`per_min_fare`, `min_fare`, `commission_percent` and `gst_percent` **per zone and vehicle
type**. Different economics per market is already expressible.

**Payments are behind an abstraction** —
`servers/payments/payment_gateways/` has `BasePaymentGateway`, a `CashfreeGateway`, and a
`factory.get_payment_gateway()` keyed on `settings.PAYMENT_GATEWAY`. Adding Stripe for
the US is **a new class and one factory branch**, not a rebuild.

**Driver hours-of-service is enforced, and the cap is already US-shaped** —
`servers/driver/fatigue.py`:

```python
FATIGUE_CAP_SECONDS     = 12 * 3600   # 12h active in any rolling 24h
FATIGUE_WINDOW_SECONDS  = 24 * 3600
FATIGUE_LOCKOUT_SECONDS =  8 * 3600   # then 8h off
```

Enforced, not merely displayed: `servers/consumers.py:435-450` refuses the driver with
`{'ok': False, 'fatigue': {...}}`. 12-on/8-off matches California's TNC rule and is
within NY's and Massachusetts's. One global constant rather than per-jurisdiction, but
the mechanism is there.

**Also portable:** marketing notifications default **OFF** with explicit opt-in
(`servers/rider/models.py:165-170`) — the correct posture for DPDP *and* TCPA; the DPDP
export/anonymise-rather-than-delete shape maps cleanly onto CCPA; and money is `Decimal`
throughout with no floats.

---

## 2. India go-live — what is still open

The pilot work is substantially done. See `india-pilot-release-gate.md` for the 51-item
scorecard. Condensed, what remains:

| | Item | Owner |
|---|---|---|
| 1 | **Rotate the database credential**, or confirm it is dead. It appeared in git history; removing it from HEAD is not rotation. Holds the gate RED on its own. | Human |
| 2 | **Run `bootstrap_qa_operator`.** No operator account exists, so not one operator workflow has been performed by a person. Unblocks four scorecard items. | Human |
| 3 | **Decide the production target** (§0) and fix its deploy path. | Human + eng |
| 4 | **Restrict the Maps keys** — three keys (Android by package + SHA-1, iOS by bundle id, server by IP), plus a billing quota and alert. | Human |
| 5 | **Rehearse a database restore** before a real rider exists. | Human |
| 6 | **Watch a real handset receive a backgrounded ride offer.** Server path tested, no device observed. | Human |
| 7 | **14 CVEs** in `urllib3` and `sentry-sdk`, both pinned by `cashfree-pg==3.2.12`. No fixed `urllib3` exists inside that constraint. Needs a Cashfree sandbox-validated SDK upgrade. | Human + eng |
| 8 | **FCM token rotation** — no `onTokenRefresh` listener; a long-signed-in driver can hold a token the server does not have. | Eng |
| 9 | **Wire Sentry.** `SENTRY_DSN` unset in every environment; a production exception is visible only in container logs. | Eng |
| 10 | **Split Celery beat out of the worker** for production. `-B` is correct at one replica and double-fires every scheduled task at two. | Eng |

India-specific compliance that is **outside code** and unchanged: GST registration and
GSTIN on receipts, the legal entity name on the receipt
(`PLATFORM_LEGAL_ENTITY` currently defaults to "SaaradhiGo Mobility" and nobody has
confirmed it), Motor Vehicles Aggregator Guidelines 2020 compliance, state aggregator
licences, and a monitored grievance-officer mailbox (`PLATFORM_CONTACT_DOMAIN` defaults
to `saaradhigo.in` — confirm the mailboxes exist).

---

## 3. USA go-live — code changes required

Ten items. Grouped by whether they are a data-model change, a correctness bug, or a
localisation gap.

### 3.1 BLOCKER — there is no currency field on any money model

The most consequential finding for a two-market platform.

`Trip.estimated_fare`, `Trip.final_fare`, `WalletTransaction.amount`, `Receipt`, and
`FarePricing` are all bare `DecimalField`s. Currency exists **only** as a function
parameter that defaults to India:

```python
# servers/payments/payment_gateways/base_gateway.py:107
def create_payout(..., currency: str = "INR", ...)
# servers/driver/services.py:1577
currency="INR",
```

So the database cannot distinguish a ₹450 trip from a $450 trip. Consequences:

- **Reporting and reconciliation break silently.** `SUM(estimated_fare)` across markets
  is a meaningless number, and `admin_dashboard`'s revenue views compute exactly that.
- **A misconfigured rate card is undetectable.** A US zone seeded with Indian magnitudes
  charges $30 base instead of $3 — or the reverse — and nothing in the schema objects.
- Driver wallet balances, the ₹500 withdrawal minimum, and the NPCI daily limits all
  compare numbers with no unit.

**Required:** a `currency` column (ISO 4217) on `ServiceZone` as the source of truth, and
denormalised onto `Trip`, `WalletTransaction` and `Receipt` at write time, with every
aggregate grouped by it. This is a migration plus an audit of every `Sum()` on a money
field. It is the single largest piece of USA work and it should be done **before** any US
zone exists, because backfilling currency onto mixed historical rows is guesswork.

### 3.2 BLOCKER — tax arithmetic is India's convention, on the receipt

`servers/ride/receipts.py:426` extracts tax **inclusively**:

```python
gst_amount = total * gst_rate / (100 + gst_rate)
```

That is correct for India, where the quoted fare includes GST. **US sales tax is
additive** — the fare is quoted, tax is added on top, and the receipt shows both lines
and a larger total.

The field is `gst_percent` and it is per-zone, so the *storage* is fine. The *arithmetic*
is not, and it is on the document the rider receives. A US receipt produced by this code
would understate the fare and misstate the tax.

**Required:** a `tax_mode` on the zone (`inclusive` | `additive`) and two code paths in
receipt generation, plus renaming `gst_percent` to `tax_percent` so nobody configures a
US zone believing the label. Several US jurisdictions also levy **per-trip** charges
rather than percentages (NYC congestion surcharge, Chicago ground transportation tax), so
the model needs a flat per-trip fee alongside the percentage.

### 3.3 DEFECT — US phone numbers are not redacted from logs

`base/logging_filters.py:34`:

```python
(re.compile(r'(\+91)(\d{2})\d{8}'), r'\1\2********'),
```

The redaction filter matches `+91` only. **A US number written to a log line goes out in
clear.** This is a live privacy defect the moment a US user exists, and under state
privacy law it is an exposure, not a cosmetic issue.

**Required:** redact on E.164 shape generally — keep the country code and a couple of
digits, mask the rest — rather than enumerating country codes.

### 3.4 DEFECT — `+91` is hardcoded in three phone-normalisation paths

```
servers/admin_dashboard/views.py:102     phone_number = f"+91{phone_number}"
servers/auth_user/admin_views.py:103     phone_number = f'+91{phone_number}'
servers/auth_user/dpdp_views.py:230      placeholder_phone = f"+91-deleted-{user.id}"
```

A US operator entering `5551234567` becomes `+915551234567` and cannot sign in. The
validation regex itself is fine — `PHONE_REGEX = r'^\+?1?\d{9,15}$'` accepts E.164
generally — so only the normalisation helpers need a default-region parameter.

**Required:** one normalisation helper taking a default region, used everywhere. Consider
`phonenumbers` (libphonenumber) rather than hand-rolling; it is the library every other
platform uses for exactly this.

### 3.5 Distance is kilometres everywhere, including the rate card

`per_km_fare` on `RateCard`; `estimated_distance_km`, `actual_distance_km`,
`quoted_distance_km` on `Trip`. US riders expect miles, and US pricing is per-mile.

Storing metric internally and converting for display is the right design — but the
**rate card is per-km**, so whoever configures a US zone must convert a per-mile rate to
per-km by hand. That is an invisible 1.609× trap sitting on the price a rider pays.

**Required:** keep storage metric; add a `distance_unit` on the zone that drives display
**and** rate-card entry, so a US zone is configured in the units its operator thinks in.

### 3.6 Payout rails are India-only

`Driver.upi_id`, `DriverUPIContact`, `payout_method` choices of `upi`/`bank`, NPCI limits
(`₹100,000` daily, `₹200,000` monthly at `servers/driver/services.py:136,159,589`), and
AML reporting at the Indian `₹50,000` threshold.

**Required for US:** ACH or Stripe Connect / instant-to-debit-card payouts, US limits,
and FinCEN rather than Indian AML thresholds. The `BasePaymentGateway` abstraction makes
the gateway swap tractable; the `upi_id` field and UPI-shaped validation on the driver
model do not generalise and need a payout-method abstraction.

### 3.7 `₹` is hardcoded in 30 places across the clients, with no localisation

```
rider app   20 occurrences in  9 files      driver app   6 in 4 files
ops web      4 occurrences in  1 file       backend      user-facing strings:
                                              "Minimum withdrawal amount is ₹500"
                                              "Insufficient balance. Available: ₹{x}"
```

Neither Flutter app has `flutter_localizations`, ARB files, or any `Locale` handling. The
`intl` package is imported but used as
`NumberFormat.currency(symbol: '₹', decimalDigits: 2)` — the symbol is a literal.

**Required:** `flutter_localizations` + ARB files in both apps; currency formatting driven
by the trip's currency, not a constant; and the backend's user-facing money strings moved
behind formatting that takes a currency. US English only is a legitimate scope — but the
machinery has to exist before a second currency does.

### 3.8 Vehicle documents are the Indian set

`rc_doc` (Registration Certificate), `permit_doc` (commercial permit), `fitness_doc`
(Fitness Certificate), `puc_doc` (Pollution Under Control). Of these, only insurance has a
US analogue; US needs **vehicle registration**, **proof of personal insurance**, and in
some states an **annual inspection** certificate — and no commercial permit or PUC.

**Required:** a document-requirement set per country rather than fixed columns, or the KYC
queue will ask US drivers for a Pollution Under Control certificate.

### 3.9 Timezone is a single global constant

`base/settings.py:843  TIME_ZONE = 'Asia/Kolkata'` and
`:360  CELERY_TIMEZONE = 'Asia/Kolkata'`.

The US spans six timezones. Consequences:

- The beat job commented "02:00 in TIME_ZONE (Asia/Kolkata) — quiet hours" for expiring
  driver licences fires at **16:30 Eastern**.
- Receipts, trip timestamps and the operator console render in IST.
- Daily and weekly earnings aggregates roll over at Indian midnight, so a US driver's
  "today" ends mid-afternoon.

**Required:** keep `USE_TZ=True` and UTC storage; derive display timezone from the trip's
zone; and make any schedule that means "local quiet hours" per-zone rather than global.
The earnings-rollover boundary is the one that will generate support tickets.

### 3.10 Dead code that fabricates success on a money path

`servers/driver/services.py:1760-1796`, `initiate_bank_payout()`:

```python
# For now, we'll log and simulate success
logger.info(f"Bank payout would be initiated via Cashfree for withdrawal ...")
return True  # Simulating success for now
```

It moves no money and returns `True`. **It is not currently reachable** — the caller at
`:1246` explicitly blocks `payout_method == "bank"`, marks the withdrawal failed, refunds
the driver and logs an error. So this is not a live defect.

It is, however, exactly the function someone will wire up when bank payouts are needed,
and it will report success for a payout that never happened. **Delete it, or make it
raise `NotImplementedError`.** A stub that returns success on a money path is a loaded
gun regardless of today's call graph.

---

## 4. USA go-live — regulatory items with no code at all

These are not gaps in an implementation; there is no implementation. Each is required by
TNC statute in most or all US states, and each needs a product decision before it needs
code.

| | Requirement | What exists today |
|---|---|---|
| 1 | **Criminal background check** — vendor (Checkr, Sterling), initial + annual re-check, adjudication workflow, adverse-action notice under FCRA | Nothing. `Driver.approved` is a boolean an operator flips. |
| 2 | **MVR (driving record) check** — initial + periodic, with disqualifying-offence rules | Nothing. |
| 3 | **Insurance by trip period** — P1 (app on, unmatched), P2 (en route to rider), P3 (rider aboard), each with different statutory limits, and the ability to evidence which period a trip was in for a claim | An `insurance_doc` **file upload** on the vehicle. No policy model, no period concept. The trip lifecycle *does* record `accepted_at` / `started_at` / `completed_at`, so the periods are derivable — but nothing derives them. |
| 4 | **1099 tax reporting** — W-9/TIN collection at onboarding, TIN matching, 1099-NEC/1099-K generation and e-filing | Nothing. Earnings are tracked precisely; the tax artifact does not exist. |
| 5 | **Wheelchair-accessible vehicle (WAV)** support — vehicle attribute, rider filter, dispatch preference, and in some jurisdictions a per-trip accessibility fee | Nothing. No accessibility concept anywhere in the codebase. |
| 6 | **Per-jurisdiction tax and per-trip fees** — state + local sales tax, additive, plus flat per-trip charges in specific cities | One inclusive `gst_percent` per zone (§3.2). |
| 7 | **10DLC / A2P SMS registration** — US carriers require brand + campaign registration for application-to-person SMS; unregistered traffic is filtered or blocked outright | SMS goes via AWS SNS (`base/utils.py`). No evidence of 10DLC registration, and OTP delivery failing silently in the US is an inability to sign anyone up. |
| 8 | **Zero-tolerance drug/alcohol policy** — published policy, in-app rider reporting, driver suspension on complaint | Nothing. The support ticket system could carry it, but there is no policy workflow. |
| 9 | **Trade dress** — a visible, illuminated emblem while the driver is on-app, with design and placement rules per city | Nothing. Product/ops, not code, but it gates launch. |
| 10 | **CCPA/CPRA** — 45-day response window, "Do Not Sell or Share My Personal Information" if data is shared for cross-context advertising, and a Californian legal basis for retention | DPDP endpoints exist and the *shape* is right (export + anonymise). The retention justification cites "7 years under Income Tax + GST"; the US basis differs, and the response-window tracking is absent. |
| 11 | **Airport permits** — most major US airports require a separate permit, geofence and per-pickup fee | Nothing. The zone model could express an airport subzone; no permit or fee concept. |
| 12 | **Worker classification** — 1099 contractor vs employee, and California's Prop 22 benefits regime (earnings guarantee, healthcare stipend, occupational accident insurance) if operating there | Nothing. This is a legal and business decision with substantial code consequences if Prop 22-style guarantees apply. |
| 13 | **PCI-DSS** — if card payments are enabled. Stripe Elements / hosted fields keep this at SAQ-A | Not applicable while cash-only, but it must be a deliberate choice at the point online payments are enabled. |

**Items 1, 2, 3, 4 and 7 are hard launch blockers** — you cannot legally operate a US TNC
without background checks, MVR checks and period-based insurance, you cannot pay
contractors without tax reporting, and you cannot onboard anyone if your OTP SMS is
carrier-filtered.

---

## 5. Shared quality findings from this review

**`servers/admin_dashboard/views.py` is 6,852 lines** with 35 broad `except Exception`
handlers. It is the operator console — the thing a brand-new operator depends on, and the
one surface in the product that has never been used by a person. A file that size is
where the next defect hides, and it cannot be reviewed meaningfully as a unit. It should
be split by domain (drivers, trips, finance, support) before more is added to it.

Broad exception handlers by file, for scale: `admin_dashboard/views.py` 35,
`consumers.py` 22, `ride/views.py` 13, `driver/services.py` 10, `payments/views.py` 5.
Prior runs found **seven** instances of a broad handler turning an infrastructure failure
into a confident business answer; these are where an eighth would live.

**Nine tests fail locally without Redis** (`test_durable_dispatch.py`,
`test_executive_revenue.py`) and pass in CI, which provides the service. Worth a `redis`
pytest marker so a developer without Docker sees a skip rather than a failure and learns
to distrust the suite.

**No PostgreSQL in `docker-compose.prod.yml`** and no documentation of which external
database production uses. For a go-live that is a gap in the runbook, not the code.

---

## 6. Recommended sequencing

**India first, USA second, and do not interleave them.** India is ~10 items from launch,
six of which are human actions. USA is a quarter of engineering work plus a legal
programme. Running both at once means the currency migration lands in the middle of the
India pilot.

**Phase 1 — close India (weeks, mostly not engineering)**
Items 1-10 in §2. Start with the credential and the operator account; they unblock the
most and need no code.

**Phase 2 — the market-portability foundation (before any US zone exists)**
§3.1 currency, §3.3 log redaction, §3.4 phone normalisation, §3.9 timezone. These four are
cheap now and expensive later — currency especially, because backfilling it onto mixed
historical rows is guesswork. §3.3 should arguably be done in Phase 1 regardless; it is a
one-line privacy fix.

**Phase 3 — USA regulatory integrations (the long pole)**
Background check and MVR vendor, the insurance period model, W-9/1099. These are
integrations with external providers and legal review attached, and they are what
actually determines the US launch date. Start the vendor conversations before the code.

**Phase 4 — USA product surface**
§3.2 tax mode and per-trip fees, §3.5 distance units, §3.6 payout rails, §3.7
localisation, §3.8 per-country documents, WAV support.

**Not in any phase:** worker classification and Prop 22. That is a legal and business
decision that should be made before Phase 3 starts, because the answer changes what gets
built.

---

## 7. Honest summary

The platform's *mechanics* are in good shape, and the pilot reports document that in
detail: a ride completes, the money reconciles to the paisa, a crashed driver recovers
themselves, a killed Celery worker loses no work, Redis can disappear at any stage
without corrupting state or money, 100 concurrent rides complete with every financial
invariant intact, and nobody can promote themselves to operator.

**For India, it is roughly ten items from launch and six of them are not engineering.**

**For the USA, the honest read is that this is an India product with good multi-market
bones, not a multi-market product.** The zone hierarchy, the per-zone pricing, the gateway
abstraction and the fatigue rules are genuinely portable and they save real time. But
there is no currency in the data model, the tax arithmetic is the wrong convention, US
phone numbers leak into logs, and the entire US regulatory surface — background checks,
MVR, period-based insurance, 1099s, 10DLC — does not exist in any form.

None of that is a criticism of how it was built. It was built for one market, carefully.
It is a statement of what the second market costs: roughly a quarter of engineering work
on the code, and a legal and vendor programme that will take longer than the code and
should start first.
