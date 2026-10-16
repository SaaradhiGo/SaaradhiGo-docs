# Multi-country support: admin + backend readiness

Written for: the CTO/product owner, and the engineer who picks this up next.

**Built at:** backend `ac6f074` on `dev` (five commits, `7461261` → `ac6f074`),
57 files, +5,018 / −758. Mobile apps are **out of scope** for this run and are
listed at the end as what a US launch still needs.

**Scope of this run:** make the admin console and the backend genuinely support
more than one country, implement it, and validate it against real scenarios
rather than a passing test count.

---

## The one-paragraph version

India was never *wrong* in this codebase — it was **unattributed**. A global
`Asia/Kolkata`, `+91` in three phone helpers, inclusive-GST arithmetic, a
`per_km_fare` column, five Indian documents in the KYC gate, `Rs.` in a push
notification, a redaction regex that only matched Indian numbers. None of it
failed loudly when a second market appeared; it produced a US rider charged in
rupee magnitudes, a driver who could never be approved, and a receipt headed
"GST" and "Rs." The work was to attribute every one of those to a country and
resolve it from the **zone**, while leaving India byte-identical.

The market is now a property of the `ServiceZone`, which already had a
country → state → city → subzone hierarchy. One registry (`base/countries.py`)
describes each country; every money row carries its own currency; and the
things that were India-shaped are resolved per market.

---

## What is ready

| Area | State | Evidence |
|---|---|---|
| Fare quote (service → wrapper → **HTTP**) | **Ready** | 7 market fields over HTTP in both markets; additive total closes exactly |
| Receipts (issue **and reissue**) | **Ready** | Both markets' tax, currency, symbol, distance unit, tax name |
| Driver onboarding / KYC | **Ready** | A fully documented US driver can be **approved**, asserted through the POST |
| Operator console money | **Ready** | Rendered `$30.83` with no `₹` anywhere on the page |
| Payout minimums | **Ready** | At the endpoint, not just the service: $10 US / ₹500 India |
| Payout rails | **Guard-railed** | A non-INR withdrawal is refused before the UPI gateway is touched |
| Online payment | **Guard-railed** | An order the gateway cannot settle is refused, not mischarged |
| Timezones ("today", night surge) | **Ready** | Each market's own midnight; night window in the zone's clock |
| Phone numbers + log redaction | **Ready** | `+1` normalisation; redaction no longer India-only |
| Tax precision | **Ready** | 3 decimal places — 8.625% was being charged as 8.62% |
| Money aggregation | **Ready** | Every money `Sum()` is now currency-grouped or currency-scoped |
| Executive revenue report | **Ready** | Reports one market at a time, with a selector |

### The blocker that was removed

**A US driver could not be approved. There was no path, in four separate
places.** The console's KYC gate required `all()` of India's five statutory
vehicle documents; three of them (PUC, commercial permit, fitness certificate)
do not exist in the US, so the condition was permanently false. Then:

1. the approval POST counted `total_checks = 6` — India's document count,
   unreachable in the US;
2. the same POST read `item["permit"]` out of a dict keyed by the documents the
   driver's country requires, so for a US driver it would have raised
   `KeyError` — a 500, not a refusal;
3. the checklist and the five document cards beside the gate were a second and
   third hand-written copy of India's five, so an operator reviewing a US driver
   saw three permanent failures and a score that could not pass 50%;
4. the US safety inspection had **nowhere to be stored**, so it was
   required-but-unrecordable, which correctly counts as not compliant — another
   permanent block.

All four are closed. `Vehicle.inspection_doc` was added (nullable; no Indian
row changes), and `unrecordable_documents()` was kept as the mechanism that
makes the *next* market's new document type block **visibly** rather than pass
silently — the console now renders it as a named gap with "raise this with
engineering."

---

## How this was validated

Three layers, deliberately, because the first two were not enough.

### 1. The drill — `qa/multi_country_drill.py`, 34/34

Builds a fresh database by applying every migration, then prices, receipts,
onboards and pays out a **real ride in each market** inside a rolled-back
savepoint, and prints the numbers:

```
--- INDIA ---
  tax: GST 5.000% mode=inclusive amount=9.05
  total charged: ₹190.00     base ₹30.00  distance ₹120.00  time ₹40.00
  [PASS] decomposition closes exactly        190.00 + 0.00 == 190.00
  [PASS] the receipt extracted tax at the card rate   expected ~9.05, got 9.05

--- UNITED STATES ---
  tax: Sales tax 8.625% mode=additive amount=2.33
  total charged: $30.83      base $2.50  distance $17.50  time $7.00
                             tax $2.33   per_trip_fee $1.50 (City access fee)
  [PASS] decomposition closes exactly        27.00 + 3.83 == 30.83
  [PASS] the receipt taxes at the rate the quote charged   quote 2.33 vs receipt 2.33
```

**The drill is what found the defects the tests missed.** On its first run the
quote charged US sales tax at 8.625% while the receipt recomputed it at 5% —
India's rate — because the receipt's rate-card fallback was a literal
`Decimal('5.00')`, and because it re-derived the zone from pickup coordinates
instead of using the zone already stamped on the trip (so a boundary change
would retroactively alter an old receipt's tax). Every individual assertion had
passed. Nothing compared the two numbers a rider compares.

### 2. The boundary tests — 118 tests, all passing

`tests/test_multi_country.py` (78), `test_multi_country_api.py` (20),
`test_multi_country_console.py` (20). The last two exist because of a mistake
worth keeping visible, described next.

### 3. The full suite — 1,174 passing

The 9 failures are the known Redis-dependent locals (7 `test_durable_dispatch`,
2 `test_executive_revenue`); they fail identically on `8a284fa` without any of
these changes, and pass in CI where Redis is up. `ruff` clean, no pending
migrations.

### 4. Deployed QA, against real PostgreSQL

All seven migrations applied cleanly to QA's live Postgres, from the Railway
deploy log (deployment `a6da2d83`, status SUCCESS):

```
07:46  Applying driver.0023_multi_country_market_fields...    OK
07:46  Applying payments.0012_multi_country_market_fields...  OK
07:46  Applying pricing.0007_multi_country_market_fields...   OK   (+ backfill)
07:46  Applying ride.0017_multi_country_market_fields...      OK
07:46  Applying rider.0012_multi_country_market_fields...     OK
08:24  Applying driver.0024_vehicle_inspection_document...    OK
08:24  Applying pricing.0008_widen_tax_rate_precision...      OK
```

No `Traceback`, `ImportError`, `CRITICAL` or 500 in the deploy stream since.
And the public zone list on deployed QA, read anonymously, resolves every
market field for all four Indian zones:

```json
{"code":"IN-TG-HYD","country":"IN","currency":"INR","timezone":"Asia/Kolkata",
 "distance_unit":"km","tax_mode":"inclusive","tax_label":"GST"}
```

`currency` was blank in that column before the backfill, so this is also
evidence the data migration did its work on real rows.

---

## The mistake worth keeping: verify at the boundary, not the layer

After the foundation commits — 77 unit tests green, drill 33/33 — a live quote
against deployed QA came back:

```
estimated_fare   '135.22'
currency         None        tax_mode      None       tax_percent    None
tax_label        None        tax_amount    None       per_trip_fee   None
distance_unit    None
```

Every market field null over HTTP, while the service layer had been producing
all of them correctly the whole time. **Three** places rebuild the fare payload
field by field — `quote_fare`, the `estimate_amount` compatibility wrapper, and
the view's own response dict — and tests against the service could only see the
first. This is the second time this exact shape of defect has shipped here; the
first was `fare_adjustments`, and `tests/test_fare_api_decomposition.py` records
it.

That is why the new tests go through the API, the rendered console page and the
approval POST. Working outward from that one null field found, at the
boundaries, worse things than the layers had:

- three copies of India's ₹500 payout minimum — the endpoint's own
  `amount < 500`, the serializer's `min_value: 500`, and the market-aware
  function that was **imported at the top of the view and never called**;
- `currency='INR'` as a literal at all three `create_order` sites, so a $30.83
  fare would have gone to a rupee-only gateway as ₹30.83 — about a ninetieth of
  what is owed, silently, with the Payment row recording a currency the rider
  never paid in;
- `"Final fare: Rs.{fare}"` in the ride-completed notification **and** the push,
  which is the copy a rider sees on a lock screen and cannot correct;
- `reissue_receipt` carrying its own copy of the inclusive-GST arithmetic with
  India's 5% as the fallback, so a reissued US receipt stated a tax that was
  never charged, at the wrong rate, in no currency;
- `_is_night_now` reading `settings.TIME_ZONE`, so a San Francisco rider booking
  at **11am** was inside the 23:00–05:00 night window because it was 23:30 in
  Kolkata — and a genuine 1am ride was not. A fare charged at the wrong time of
  day, with nothing erroring.

Two more surfaced in the aggregation sweep, and neither needed a second market
to be wrong:

- `check_monthly_upi_limit` computed its window as
  `timezone.now().replace(day=1, hour=0, ...)` — **UTC** midnight, which is
  05:30 IST. A withdrawal taken in the first five and a half hours of an Indian
  month counted against the **previous** month's NPCI ceiling. Wrong today, in
  the live market.
- the driver-loyalty page labelled `Sum(actual_distance_km)` as `mi`. Indian
  kilometres presented as miles.

---

## One decision worth recording: the executive report scopes, it does not convert

The executive revenue report is the one screen where a single number is the
whole point — GBV, platform revenue, the take rate, the vehicle-class split,
the monthly series, month-over-month growth. Every one of those is built by
adding `final_fare` across completed trips.

There were three options:

1. **Add them anyway.** ₹190 + $30.83 = 220.83, which is not an amount of
   money in either currency, and the template renders it with a leading symbol
   — so the page would assert it was rupees. Growth percentages computed from
   such a number are worse than wrong: they move when the currency mix moves,
   not when the business does. A board figure that drifts with volume mix.
2. **Convert at a rate.** There is no FX source in this system. Adding one
   makes a board figure depend on a number nobody in this repo owns, and a
   historical report would change value every time the rate moved unless rates
   were snapshotted per trip — which is a project, not a patch.
3. **Report one market at a time.** Every figure stays an amount of money, the
   growth rates stay meaningful, and nothing is invented.

It does (3). The report defaults to the only currency with data — so the India
pilot's page is unchanged, with no selector and no banner — and shows a market
selector plus a one-line scope statement once a second market has rides. That
behaviour is asserted in four tests, including the single-market control.

If a genuinely consolidated figure is wanted later, it needs an FX policy
decided by the business: which rate, as of when, and snapshotted where. That is
a product decision, and it is listed here rather than guessed at.

**The daily ops dashboard keeps one day boundary**, the deployment's, and that
is also now a stated convention rather than the timezone setting showing
through. These are platform-wide tiles and the ops team reading them sits in
one place, so a single cut-off is what makes them comparable day to day — at
the cost that Indian midnight splits a US afternoon. The money tiles there are
already per currency (`gmv_by_currency`, `withdrawals.completed_today_by_currency`),
and the flat `gmv` is `null` rather than a sum when the day spans markets.

---

## India did not move, and that is asserted

The entire risk of this change is a refactor that silently reprices a live
market. The guards:

- the pinned arithmetic, 30 + 12×10 + 2×20 = **190.00**, at the service layer
  **and** through the HTTP API;
- no `tax` and no `per_trip_fee` adjustment line in India — the guard against
  India gaining an additive tax is the **absence of the line**, not a zero
  amount, because the quote now truthfully reports the ₹9.05 GST already inside
  the fare (showing a rider "GST 0.00" beside a 5% inclusive rate was a lie, and
  reporting it does not change the total);
- each of India's five documents, individually, still blocking approval — five
  parametrised cases through the POST;
- the night window unmoved: 23:30 IST is night, 12:00 IST is not;
- the fallback road factors byte-identical (1.4 detour, 25 km/h);
- `5.00` and `5.000` are the same number, so the tax-precision widening is a
  no-op for every Indian rate card.

---

## What is NOT done

Stated plainly, because a partial that is known is safer than one that is
assumed closed.

### Out of scope for this run, by instruction

**The mobile apps.** Rider and driver Flutter apps are untouched. The backend
now *sends* `currency`, `distance_unit`, `tax_mode`, `tax_label`, `tax_amount`
and `per_trip_fee` — in the quote response, the public zone list, the dispatch
offer and `trip_created` — and every one is **additive**, so current builds are
unaffected. But an app that does not read them will render a dollar fare with a
rupee sign and a mileage trip in kilometres. That is the single largest
remaining item for a US launch.

### Not attempted, and why

- **No US payment gateway and no US payout rail.** Cashfree is INR-only and UPI
  is an Indian rail. Both paths now *refuse* a non-INR amount instead of
  mischarging it, which is the correct behaviour in the absence of a provider —
  but it means a US ride can only be taken in cash today. Selecting a US
  provider is a commercial decision, not a code change.
- **No US tax determination.** `default_tax_percent` for the US is deliberately
  `0.00`: there is no national rate, and sales tax is state and local. A US zone
  **must** carry a rate card declaring its own rate; if it does not, the receipt
  logs a loud ERROR and taxes at zero rather than inventing a jurisdiction's
  rate. Per-jurisdiction tax lookup (origin vs destination sourcing, district
  rates) is not built.
- **The US fallback road factors are starting points, not measurements.** 1.25
  detour and 40 km/h, used only when the Google Distance Matrix call fails.
  Duration is a priced component, so these should be re-tuned against real trip
  data before a US launch. They replaced Hyderabad's 25 km/h, which inflated a
  US fallback quote's time fare by roughly 60%.
- **Phone validation is still length-based.** `base/countries.py` documents the
  trigger for moving to `phonenumbers`: more than five countries, or a country
  with variable-length national numbers. India (10) and the US (10) are both
  fixed-length, so the library is not yet worth the dependency.
- **India's UPI/NPCI limits remain India-only constants** (₹100,000 daily,
  ₹200,000 monthly, >₹50,000 reporting) in `driver/services.py`. That is
  correct — they are NPCI's figures for the only market that has a payout rail —
  and they are now explicitly INR-scoped rather than implicitly so, and
  unreachable from a non-INR withdrawal. A second payout rail will need its own
  limits, not a translation of these.

### Checked and found to need nothing

- **SOS.** No hardcoded emergency service number. The fan-out goes to ops and
  to the user's own emergency contact, which is market-agnostic.
- **SMS.** AWS SNS, which publishes to E.164 numbers in both countries.
  India's DLT sender-ID registration and the US's 10DLC/toll-free registration
  are account-level prerequisites, not code.
- **Service-area enforcement.** Already zone-table driven, not a hardcoded
  Hyderabad polygon — only the operator-facing message still said "Hyderabad",
  and that copy is fixed.
- **Per-zone revenue aggregation.** Safe by construction: a zone has exactly
  one currency.

### A dead-code trap worth knowing about

`templates/admin_pages/riders.html` has two large `{% comment %}` blocks
(lines 3–65 and 1086–1461) that contain a full rider-detail panel, wallet
balance and wallet-transaction rows included. None of it renders.

Three currency fixes were applied there before this was noticed, and then
reverted: editing dead markup adds diff noise and implies the page does
something it does not. If that panel is ever uncommented it needs
`|money:row.currency` on each amount, and the test file records that. Worth
checking for the same trap before "fixing" any other console template.

### Verification I could not complete

**The authenticated live quote against deployed QA was not re-run.** It needs
the QA test phone/OTP pair, which is not in this environment, and I did not go
and fetch credentials to get it. What *was* verified live, anonymously, is the
public zone list on deployed QA — the one market-bearing endpoint that needs no
login. The authenticated payload is asserted at the HTTP boundary by
`tests/test_multi_country_api.py`, which is the boundary a client reads, but it
is Django's test client rather than the deployed service. **Re-running the
live quote is a five-minute check for whoever has the credentials, and it is
the one piece of evidence this report is missing.**

---

## Where the pieces live

| File | What it owns |
|---|---|
| `base/countries.py` | The registry. One `Country` per market: currency, symbol, calling code, distance unit, tax mode and label, default timezone and tax rate, required documents, road fallbacks. |
| `base/money.py` | Decimal money, `ROUND_HALF_UP`, `sum_by_currency`, `format_totals`. No single-number total, on purpose. |
| `base/currency_stamp.py` | `CurrencyStampedModel` — stamps currency at `save()`, so a new call site inherits it by existing. Widens `update_fields` **only** when the value actually changed. |
| `base/market_time.py` | Which "today" a number belongs to. Zone → IANA timezone → that market's local midnight. |
| `servers/driver/compliance.py` | Which documents a driver must supply, in the country they drive in. |
| `servers/pricing/models.py` | `ServiceZone` market columns, with country-default inheritance at `save()`. |
| `qa/multi_country_drill.py` | The drill. Run it after any pricing or receipt change. |

**If you change pricing, receipts, onboarding or payouts, run the drill.** It is
the thing that catches what unit tests structurally cannot: two numbers that
each pass their own assertion and disagree with each other.
