# Fare finalisation — design report

- **Status:** investigation only. Nothing implemented. Depends on the GPS trail.
- **Date:** 2026-09-22
- **Depends on:** [ADR-0010 DRAFT](../adr/0010-DRAFT-trip-gps-trail.md)

## The finding, stated plainly

An AST scan of the entire backend for writes to `Trip.final_fare`,
`actual_distance_km`, `actual_duration_min` and `cancellation_fee` returns
**exactly one production write site**:

```
servers/consumers.py:640   Trip.objects.create(estimated_fare=...)
```

That is all. In production:

| Field | Written? |
|---|---|
| `estimated_fare` | once, at trip creation |
| `final_fare` | **never** |
| `actual_distance_km` | **never** |
| `actual_duration_min` | **never** |
| `cancellation_fee` | **never** |

Every consumer resolves `trip.final_fare or trip.estimated_fare`, so with
`final_fare` permanently NULL **the entire money chain is computed from the
up-front estimate**:

* rider charge — `payments/views.py`
* driver settlement and platform commission — `driver/utils.credit_driver_wallet`
* GST receipt total — `ride/receipts.py`
* GMV and executive revenue — `admin_dashboard/views.py`

`cancellation_fee` is worse than unused: the ops dashboard *aggregates* it
(`"cancellation_fees": cancellation_total`), so a real number is being reported
that is structurally always zero. ADR-0007 line 117 states that
`cancelled_by`, `cancellation_reason` and `cancellation_fee` "are set" by the
cancel paths. The first two are; **the third is not.** That doc line should be
corrected regardless of what happens to fares.

## Canonical fare source

`servers/pricing/services.quote_fare()` is unambiguously the single engine, and
it is already good. It resolves the zone by point-in-polygon, picks the
`RateCard` in force **at the trip's request time**, applies per-km and per-min
rates, the night-surge window, the MVA-2020 surge cap, and the minimum fare, and
returns a full breakdown including `zone_code` and `rate_card_version`.

It is used by `/estimate-fare/` and by trip creation, which is why the estimate
itself is trustworthy. `FarePricing` then stores the breakdown as a row per
trip.

What `quote_fare` does **not** model, and would need for a true final fare:

* **waiting time** — no concept of it anywhere in the codebase
* **tolls** — no field, no rate-card entry
* **night surge on the *actual* trip window** — currently evaluated once, at
  request time, so a trip that crosses 23:00 is priced as if it did not
* **rounding to rupees** — `round(x, 2)` gives paise; Indian ride-hailing
  normally presents whole rupees

GST is handled separately and correctly at receipt time: extracted *inclusive*
from the total (`gst = total * rate / (100 + rate)`) using the rate card at
issue. So GST follows whatever total it is given — it does not need changing,
but it does mean a wrong total silently produces a wrong tax document.

## Where finalisation has to happen, and the ordering trap

Both of these run **inside** the completion transaction, and both read the fare:

```python
elif status_code == 'completed':
    trip.completed_at = timezone.now()
    self._create_payment_on_complete(trip)          # reads final_fare or estimated_fare
    ...
# and, later in the same flow:
credit_driver_wallet(trip)                          # reads final_fare or estimated_fare
```

So `final_fare` must be written **before** both, in the same transaction.
Otherwise the rider is charged the final fare while the driver's commission is
taken on the estimate, or vice versa — a money split that would be very hard to
reconcile after the fact.

`credit_driver_wallet` is called from **seven** places (completion, cash
confirmation, three payment-webhook paths, the payment reconciler, and retry).
It is idempotent on `TRIP_{id}_EARNING`, so the *first* call wins and fixes the
commission basis permanently. That makes the ordering requirement absolute: if
any of those seven can fire before finalisation, the driver is settled on the
estimate and the later final fare never reaches them.

## When the fare must become immutable

**At the moment the completion transaction commits.** After that point:

* a `Payment` row exists and may already be settled at the gateway;
* `credit_driver_wallet` has fixed the commission basis via its idempotency key;
* `issue_receipt` will emit a **CGST §31 tax document**, which must not change
  after issue.

So finalisation is a one-shot write inside the completion transaction, and any
later correction must be an explicit adjustment or refund with its own audit
trail — never a mutation of `final_fare`. The existing `Receipt.version` field
suggests re-issue was anticipated; that is the right mechanism for corrections.

## Deriving actual distance and duration from the trail

Once the GPS trail writer lands (ADR-0010):

* **duration** = `completed_at - started_at`. Already available and needs no
  trail. Note `started_at` is set at OTP verification, so it correctly excludes
  the driver's approach to the pickup.
* **distance** = sum of haversine between consecutive stored points with
  `recorded_at` between `started_at` and `completed_at`. The sampler's 25 m
  threshold means the sum slightly under-reads a curved route; a fixed
  correction factor is *not* recommended — prefer lowering the sampling
  threshold if accuracy proves insufficient, because a fudge factor in a fare is
  indefensible to a driver or a regulator.

The trail must be filtered to the in-progress window specifically. Points from
`accepted`/`reached` are the driver's approach and must not be billed.

## Required fallback policy — the decision that needs a human

Telemetry will sometimes be missing: the driver app was killed, the device
denied location, or the worker draining the stream was down. A fare cannot be
left undefined, and silently falling back to the estimate is exactly today's
bug.

Proposed policy, **needs sign-off because it is commercial, not technical**:

| Trail quality | Rule |
|---|---|
| Good (points spanning ≥80% of the trip duration, no gap > 2 min) | bill the recomputed final fare |
| Degraded (gaps, or coverage 40–80%) | bill the **greater** of estimate and recomputed, flag the trip for ops review, and record why |
| Absent (< 40% coverage) | bill the estimate, mark `fare_basis = 'estimate_no_telemetry'`, and count it — a rising rate is an app or worker outage signal, not a pricing event |

Whatever is chosen, the trip must record **which basis was used**. Today there
is no way to tell an estimate-billed trip from a metered one, and without that
field no one can audit the change or measure the fallback rate.

## Recommended sequencing

1. GPS trail **writer** (the deferred half of ADR-0010). Nothing below is
   possible without it.
2. Populate `actual_distance_km` / `actual_duration_min` at completion.
   Observe-only at first: write them, change no billing, and compare against
   the estimate across real trips for a week. **This is the cheapest way to
   learn what finalisation would actually do to revenue and driver payouts
   before it does it.**
3. Add `fare_basis` and the fallback policy above.
4. Only then compute `final_fare` through `quote_fare`, inside the completion
   transaction, before `_create_payment_on_complete` and
   `credit_driver_wallet`.
5. Separately: implement `cancellation_fee`, or remove it and stop aggregating
   it in the ops dashboard. Reporting a structural zero as a metric is worse
   than reporting nothing.

Step 2 is the important one and is worth doing on its own. It is non-breaking,
it turns an argument into data, and it makes step 4 a measured change rather
than a leap.

## Explicitly out of scope here

Waiting-time charges, tolls, rounding-to-rupees and re-evaluating night surge
over the actual trip window are all **product pricing decisions**, not
engineering gaps. They should be decided before step 4, because adding them
afterwards means changing fares twice.
