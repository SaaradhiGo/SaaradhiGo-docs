# Phase-0 Pricing & Fee Schedule

For Hyderabad pilot launch.

- **Effective from:** {{LAUNCH_DATE}}
- **Version:** {{POLICY_VERSION}}
- **Owner:** Founder + Operations Manager
- **Reviewed by:** Compliance / CA / Legal

> **DRAFT — these numbers are recommended starting points calibrated to
> Hyderabad ride-hailing in 2026 and to SaaradhiGo's launch context. The
> founder team must finalise pricing based on:**
> - Actual driver supply willingness to operate at these splits
> - Competitive landscape (Uber, Ola, Rapido, Namma Yatri, etc.)
> - Working capital available to fund driver incentives
> - Regulatory caps (Telangana Transport Department, MVA 2020 1.5x surge cap)

---

## TL;DR — what goes into the platform

These are the values to seed into the database and the in-app schedule on
launch day. Engineering sets them via `VehicleFarePricing` rows + env vars.

| Setting | Auto | Hatchback | Sedan | SUV |
|---|---|---|---|---|
| Base fare (₹) | 30 | 45 | 60 | 100 |
| Per-km (₹/km, distance fare) | 12 | 14 | 17 | 23 |
| Per-min (₹/min, time fare) | 1.50 | 1.50 | 2.00 | 2.50 |
| Minimum fare (₹) | 40 | 70 | 100 | 150 |
| Night-surge multiplier (11pm–5am) | 1.25 | 1.25 | 1.25 | 1.25 |
| Maximum surge multiplier (any cause) | 1.50 | 1.50 | 1.50 | 1.50 |
| **Platform commission (% of gross fare)** | **18%** | **18%** | **20%** | **20%** |
| Cancellation fee (rider, post-acceptance + 60s OR post-arrival wait) | 25 | 25 | 50 | 75 |
| Driver compensation on rider no-show | 25 | 25 | 50 | 75 |
| Waiting charge after no-fare window (₹/min, post-3 min wait) | 1.50 | 1.50 | 2.00 | 2.50 |
| Toll / parking | Pass-through to rider at actual + 0% |

`PLATFORM_COMMISSION_PERCENT` env var — set to `20`. The category-level
override (`18%` for auto and hatchback) is configured in
`VehicleFarePricing.platform_commission_percent` per row.

---

## How a fare is computed

```
Distance fare    = per_km × validated_distance_km
Time fare        = per_min × validated_duration_min
Subtotal         = base_fare + distance_fare + time_fare
Surge'd subtotal = max(subtotal, min_fare) × surge_multiplier
Total fare       = surge'd subtotal + tolls + parking
GST              = applicable rate × total fare (5%/12% taxi, NIL auto)
Final fare       = total fare + GST
```

Distance and duration are **server-side** values from Google Distance
Matrix (or, if Google is unavailable, Haversine × road-factor 1.4). The
client-supplied distance is logged for fraud monitoring but does NOT
drive the fare.

---

## Worked examples (Hyderabad)

### Example 1 — Banjara Hills to Hitech City (sedan, 12 km / 30 min, no surge, day)

| Item | Value |
|---|---|
| Base fare | ₹60 |
| Distance fare (12 km × ₹17) | ₹204 |
| Time fare (30 min × ₹2) | ₹60 |
| Subtotal | ₹324 |
| Surge | 1.0× (none) |
| Subtotal after surge | ₹324 |
| Minimum-fare floor (₹100) | not applied (324 > 100) |
| Tolls (PVNR Expressway) | ₹50 |
| **Total fare (incl. taxes)** | **₹374** |
| GST (5% on ₹324, AC taxi, no ITC; tolls excluded) | ₹16.20 |
| **Rider pays** | **₹390.20** |
| SaaradhiGo commission (20% of ₹324) | ₹64.80 |
| TDS u/s 194O (1% of gross ₹324) | ₹3.24 |
| **Driver receives** | **₹305.96 (after commission, TDS) + ₹50 toll pass-through** = **₹355.96** |

### Example 2 — Madhapur to Gachibowli (auto, 6 km / 20 min, evening 1.2× surge)

| Item | Value |
|---|---|
| Base fare | ₹30 |
| Distance fare (6 km × ₹12) | ₹72 |
| Time fare (20 min × ₹1.50) | ₹30 |
| Subtotal | ₹132 |
| Surge | 1.2× |
| Subtotal after surge | ₹158.40 |
| Minimum-fare floor (₹40) | not applied |
| Tolls | nil |
| **Total fare (incl. taxes)** | **₹158.40** |
| GST (NIL on auto-rickshaw fare via aggregator) | 0 |
| **Rider pays** | **₹158.40 → ₹159 (rounded up)** |
| SaaradhiGo commission (18% of ₹158.40) | ₹28.51 |
| TDS u/s 194O (1% of gross ₹158.40) | ₹1.58 |
| **Driver receives** | **₹128.31** |

### Example 3 — Late-night Shamshabad airport pickup (sedan, 28 km / 50 min, night surge 1.25×)

| Item | Value |
|---|---|
| Base fare | ₹60 |
| Distance fare (28 km × ₹17) | ₹476 |
| Time fare (50 min × ₹2) | ₹100 |
| Subtotal | ₹636 |
| Night surge | 1.25× |
| Subtotal after surge | ₹795 |
| Airport entry fee + parking (pass-through) | ₹110 |
| **Total fare (incl. taxes)** | **₹905** |
| GST (5% on ₹795) | ₹39.75 |
| **Rider pays** | **₹944.75 → ₹945 (rounded up)** |
| SaaradhiGo commission (20% of ₹795) | ₹159 |
| TDS u/s 194O (1% of gross ₹795) | ₹7.95 |
| **Driver receives** | **₹628.05 + ₹110 pass-through = ₹738.05** |

---

## Cancellation policy details

| Scenario | Cancellation fee (rider) | Driver compensation |
|---|---|---|
| Rider cancels before driver accepts | Free | None |
| Rider cancels within 60s of driver acceptance | Free | None |
| Rider cancels >60s after acceptance, before driver arrives | ₹25 (auto/hatch), ₹50 (sedan), ₹75 (SUV) | ₹25 (auto/hatch), ₹50 (sedan), ₹75 (SUV) |
| Rider doesn't show within 5 min of driver arrival | Full cancellation fee + no-show comp | Same as above |
| Driver cancels (in good standing) | Free | None (counts against acceptance rate) |
| Driver cancels (>3 in 7 days, no documented reason) | Free | Negative score event |

Cancellation fee is paid directly to the driver. SaaradhiGo does not take
commission on cancellation fees (Phase-0 — review for Phase-0.5).

## Waiting charge

After driver arrives:

- **First 3 minutes:** free.
- **Minute 4 onwards:** ₹1.50/min (auto/hatch) or ₹2/min (sedan/SUV).
- Wait charge accrues to the driver (no SaaradhiGo commission on wait).

---

## Surge pricing details

### Why surge

When supply (online drivers) drops below demand in a given area, surge
raises fares for that period to (a) discourage marginal riders, (b)
incentivise drivers to come online or move to the area. It's not a profit
mechanism — it's a market-clearing mechanism.

### Cap

Surge maximum is **1.50× base fare**, in line with MVA 2020 and consistent
with the Karnataka cap (Telangana has not published a hard cap as of 2026
mid-year but the 1.5× number is the safe operating value).

### Triggers

The server computes surge in `servers/ride/utils.py:estimate_amount` based
on demand/supply ratio in a 3km radius around the pickup:

| Ratio (demand ÷ supply) | Surge |
|---|---|
| < 0.5 (and supply ≥ 10) | 0.90× (discount) |
| 0.5 – 1.5 | 1.00× |
| 1.5 – 3 | 1.15× |
| 3 – 5 | 1.30× |
| ≥ 5 | 1.50× (cap) |

Night surge (11pm – 5am): adds a 1.25× multiplier to whatever the
demand-supply surge is, **capped at the same 1.50× ceiling.**

### Display

Surge must be clearly displayed to the rider at the time of booking
("Fares are 1.3× normal due to high demand"). Hidden surge is unethical
and a regulatory risk.

---

## Driver incentives (recommended Phase-0 acquisition spend)

The platform earns ~18-20% commission, and competitors will pay drivers
better in the short term to crowd you out. To onboard the first 100-200
drivers in 30-60 days, recommend:

| Incentive | Phase-0 default |
|---|---|
| **Sign-on bonus** | ₹500 paid after the first 10 completed trips with ≥4.0 rating |
| **Weekly target bonus** | ₹500 for 30 trips in a week with ≥95% acceptance and ≥4.2 rating |
| **Peak-hour incentive** | +₹15 flat per trip during 8-10am and 6-9pm weekdays |
| **Referral bonus (driver-to-driver)** | ₹1,000 to the referrer when the referee completes 50 trips |
| **First-month commission discount** | 0% commission for the first 30 days of operation (after that, standard 18%/20%) |

**Budget:** at 100 drivers × ~₹2,500 average bonus per driver in month 1 =
₹2.5 lakh allocation. Plan for ₹10-15 lakh acquisition spend for the
first 60-90 days. Treat this as a launch line item, not a recurring
cost; ramp it down once supply stabilises.

---

## Rider promotions (recommended Phase-0)

| Promotion | Default | Funded by |
|---|---|---|
| **First ride free up to ₹150** | New rider, one-time | SaaradhiGo (driver paid in full) |
| **Refer-a-friend** | ₹100 SaaradhiGo Wallet credit per side after referee's first paid trip | SaaradhiGo |
| **Office corridor discount** | 10% off Hitech City ⇄ Banjara Hills corridor during 9am-10:30am and 6:30pm-8:30pm weekdays for the first 60 days | SaaradhiGo |
| **Airport drop-off promo** | Flat ₹50 off airport trips for first 30 days | SaaradhiGo |

**Budget:** ~₹5-7 lakh over the first 60 days.

---

## Payout schedule (drivers)

| Aspect | Phase-0 setting |
|---|---|
| Cadence | Weekly — Monday for the prior Mon–Sun |
| Minimum balance to trigger an on-demand withdrawal | ₹500 |
| Maximum on-demand withdrawals per 7-day window | 2 |
| Maximum on-demand amount per request | Available balance, daily cap ₹25,000 |
| Method | UPI VPA (preferred) → IMPS/NEFT bank fallback |
| Cooling-off after a failed payout | 24 hours; driver must update VPA/bank |
| TDS withholding | 1% on gross fare, deducted before payout |

These are configurable per-driver in case operations need a different
cadence for specific cases.

---

## What we are NOT doing in Phase-0

These get deferred to Phase-0.5 or Phase-1; do not promise to drivers or
riders:

- **Shared rides / pool** — adds matching complexity
- **Scheduled rides** — supply uncertainty makes it unreliable at this scale
- **Multi-stop trips** — adds dispute surface
- **Tipping** — adds payment complexity for cash trips; consider for
  Phase-0.5
- **Subscription / pass plans** — premature
- **Inter-city / outstation** — different permit class
- **Female-only driver flag** — important but not at MVP
- **Quiet ride preference, ambient music control** — Phase-1 nicety
- **Loyalty tiers / membership** — premature; lacks data to design
- **Multi-currency** — N/A
- **Crypto / non-INR payment** — N/A

---

## Numbers we'll watch in the first 30 days

These tell you whether the pricing is calibrated correctly. Engineering
should expose these on the admin dashboard (or in CloudWatch).

| KPI | Healthy range | Action if outside |
|---|---|---|
| Trip completion rate | > 85% | Below: investigate cancellations |
| Driver cancellation rate | < 10% | Above: pricing or matching issue |
| Rider cancellation rate (post-acceptance) | < 10% | Above: ETA too long or fare surprise |
| Average wait at pickup | < 5 min | Above: supply too low or matching too wide |
| Fare estimate vs final fare deviation | < 10% | Above: distance estimation issue |
| Driver weekly earnings (mid-50%) | ₹4,500 – ₹9,000 | Below: incentives + commission re-look |
| Driver "online but idle" rate | < 30% | Above: matching not finding them, or supply > demand |
| Surge prevalence (% of trips at >1.0×) | 5–25% | Outside: surge thresholds need tuning |
| Daily complaints per 1000 trips | < 3 | Above: ops / quality issue |
| SOS triggers (genuine + false alarms) | observe & log | Treat every one as P0 |

Bake these into a once-daily slack/email report from a Celery beat task.

---

## Pricing change governance

Once the pricing is live, change it via:

1. Proposal in a ticket — what changes, why, expected effect
2. Founder + Ops Manager + CA review (CA verifies GST implications)
3. Driver-side announcement: ≥30 days notice for any commission or payout
   schedule change (per the Driver Agreement)
4. Rider-side announcement: in-app banner ≥7 days before
5. Engineering deploys to staging → smoke test → production
6. KPI watch for 14 days

Don't change pricing more than once a quarter unless an obvious problem
emerges.

---

## What to seed into the database

For engineering reference — minimum row set for the `VehicleFarePricing`
table:

```sql
INSERT INTO ride_vehiclefarepricing (vehicle_type_id_id, base_fare, per_km_fare, per_min_fare, min_fare, night_surge_multiplier, source_label)
VALUES
  ((SELECT id FROM driver_vehicletype WHERE type='auto'),     30, 12.00, 1.50,  40, 1.25, 'phase0'),
  ((SELECT id FROM driver_vehicletype WHERE type='hatchback'), 45, 14.00, 1.50,  70, 1.25, 'phase0'),
  ((SELECT id FROM driver_vehicletype WHERE type='sedan'),     60, 17.00, 2.00, 100, 1.25, 'phase0'),
  ((SELECT id FROM driver_vehicletype WHERE type='suv'),      100, 23.00, 2.50, 150, 1.25, 'phase0');
```

(Field names + an exact migration helper come from a follow-up
engineering PR. Ops can ask engineering to ship a one-time data
migration for these once you confirm the numbers.)

Set `PLATFORM_COMMISSION_PERCENT=20` in `.env.prod`. Per-vehicle-type
overrides for auto/hatchback go into the model itself if/when the
schema gains a per-type commission field; for Phase-0 the simplest
operational fix is to set a single 18% commission and accept the higher
sedan/SUV margin slips a bit.

---

*Last updated: {{LAUNCH_DATE}} — Version {{POLICY_VERSION}}*
