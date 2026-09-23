# Fare finalisation — design, with the 18 questions answered

- **Status:** design. **Nothing monetary implemented.** No rider charge, driver
  settlement, commission or receipt changes on any branch.
- **Date:** 2026-09-23
- **Builds on:** [fare-finalisation-design-report](fare-finalisation-design-report.md)
  (the original investigation), [ADR-0010](../adr/0010-DRAFT-trip-gps-trail.md)
- **Now available that was not before:** a durable GPS trail, observe-only
  `actual_distance_km` / `actual_duration_min`, and a fare shadow that records
  what metering *would* have charged.

Every answer below is what the code does **today**, followed by what it should
do. Where an answer is a commercial decision rather than an engineering one it is
marked **DECISION NEEDED** and nothing has been built for it.

## Read this first: three things the investigation found

**1. Promo codes are never applied.** `ride/promos.py` defines `apply_promo`
(preview) and `redeem_promo` (commit). `apply_promo` is called from one
fare-estimate endpoint. **`redeem_promo` has no callers anywhere.** There is no
discount column on `Trip`, `FarePricing` or `Payment`. So a rider can enter a
promo code, be shown a discount, and then be charged the full fare. This is a
live customer-facing defect independent of fare finalisation, and it is the
single most likely source of a "you charged me the wrong amount" complaint today.

**2. The Cashfree order is created after completion, not before.** `POST
/payments/create-order/` is called by the rider's app from the payment screen,
requires `status == 'completed'`, and reads `final_fare or estimated_fare` at that
moment. That makes question 14 far simpler than it looks: if finalisation happens
inside the completion transaction, no order can exist yet.

**3. Settlement records neither the fare nor the rate it charged.**
`WalletTransaction` stores one net-or-commission amount, a parsed `TRIP_<id>`
reference string and no trip FK. `TransactionHistory` stores an amount and no
commission. So the commission charged on a trip is only recoverable by
re-deriving it from a rate card that may since have changed. Any fare
finalisation must fix this at the same time, or the platform will have an
immutable fare and a mutable record of what it took from the driver.

---

## 1. When does `final_fare` become immutable?

**At the commit of the completion transaction.** It is written once, inside that
transaction, before anything reads a fare.

The reason is ordering, not preference. Inside the same transaction
`_create_payment_on_complete` creates the `Payment` row, and `credit_driver_wallet`
fixes the commission basis behind the idempotency key `TRIP_{id}_EARNING` — the
first call wins permanently. `credit_driver_wallet` has **seven** callers
(completion, cash confirmation, three payment-webhook paths, the payment
reconciler, retry). If any of them can fire before `final_fare` is written, the
driver is settled on the estimate and no later correction reaches them.

After commit, `final_fare` is never updated. A correction is a new adjustment or
refund row plus a new `Receipt` version — `Receipt.version` and
`issue_receipt_version` already exist for exactly this. Mutating `final_fare`
after a CGST §31 tax document has been issued is not an option.

## 2. What happens if GPS points are sparse?

Today: nothing, because nothing reads the trail. With the observe-only metrics
now landed, sparsity is *measured* rather than guessed: every computation returns
`coverage_ratio`, `max_gap_seconds` and `rejected_segments`, and a straight line
across a gap under-reads a real route rather than over-reading it.

**DECISION NEEDED** — the banding. The original report proposed:

| Trail quality | Rule |
|---|---|
| coverage ≥ 80%, no gap > 2 min | bill the metered fare |
| coverage 40–80%, or gaps | bill the **greater** of estimate and metered, flag for ops |
| coverage < 40% | bill the estimate, record the reason, count it |

That banding is still the recommendation, with one change: **decide it from the
shadow data, not from these round numbers.** The shadow now records coverage
alongside every comparison, so the thresholds can be chosen by looking at where
metered and estimated fares actually diverge in this city, on this driver
population. Choosing them before the data exists is guessing with someone's money.

## 3. What happens if GPS is completely missing?

The metrics module returns `distance_km = None`, deliberately distinct from
`0.00`, and writes nothing. `actual_duration_min` is still written, because
duration comes from `started_at`/`completed_at`, which are authoritative
server-side facts and always present.

So a trip with no telemetry can still be billed on time but not on distance. It
must be billed on the estimate and **marked as such** (see question 4). A rising
rate of these is an app or worker outage, not a pricing event, and needs to be
visible as a metric so it is investigated rather than absorbed into revenue.

## 4. Is estimate the fallback?

Yes — and that is exactly today's silent bug, which is why the fallback has to
become explicit. `final_fare` is NULL on every trip and every consumer reads
`trip.final_fare or trip.estimated_fare`, so the estimate is already the fallback
everywhere, with nothing recording that fact.

**Required before any metering:** a `fare_basis` column on `Trip`, with values
like `metered`, `metered_degraded`, `estimate_no_telemetry`, `estimate_legacy`.
Without it nobody can audit the change, measure the fallback rate, or answer a
driver asking why two similar trips paid differently. This field is cheap, is not
itself a monetary change, and should land **before** the first metered fare.

## 5. How are waiting/time charges handled today?

There is **per-minute time fare** (`RateCard.per_min_fare`), applied to the
quoted duration at booking. There is **no concept of waiting time** anywhere in
the codebase — no field, no rate-card entry, no timer. A grep for waiting-time
handling in `pricing/` and `ride/` returns nothing.

Note what the per-minute fare currently means: it is charged on the *estimated*
duration, so a trip that takes 40 minutes instead of 20 is billed for 20. If
metering lands, the per-minute fare starts following real elapsed time, and that
alone will change fares for every congested trip. It is the largest single effect
of finalisation and must be quantified from the shadow before it ships.

**DECISION NEEDED** — whether the driver's wait at pickup (`reached_at →
started_at`) is billable at all, and if so above what free threshold. The
lifecycle already records both timestamps, so the data exists; the policy does
not.

## 6. How is surge preserved?

Today: `Trip.surge_multiplier` is stamped at creation, and `FarePricing` stores
the multiplier and the full breakdown for the trip. That is the right shape and
should be kept.

The trap is that `quote_fare` re-evaluates surge **at call time**: it reads live
Redis demand through `compute_surge_multiplier` and the wall clock through
`_is_night_now`. Calling it again at completion would therefore reprice the trip
at completion-time surge — the rider would be charged for demand that arose after
they were quoted, which is indefensible.

**So finalisation must not re-quote surge.** It re-quotes distance and duration at
the *stamped* multiplier: take the pre-surge subtotal from the new metrics, then
apply `Trip.surge_multiplier`, the stamped rate card version, and the night flag
recorded at booking. The fare shadow already demonstrates this hazard: it quotes
the estimate and the actual in the same instant precisely so that surge drift
lands in a separate, labelled number instead of contaminating the comparison.

A consequence: `quote_fare` needs a way to be called with an explicit multiplier
and an explicit rate card, rather than resolving them live. That is a small,
non-monetary refactor of the canonical function and should land before
finalisation.

## 7. How are promos applied?

**They are not.** See finding 1 above. `redeem_promo` has no callers, and no
model has a discount column.

This must be fixed before or with finalisation, because a discount applied after
a fare is finalised has no place to live. The design:

* `Trip.discount_amount` and a `PromoRedemption` row created **in the same
  transaction as the trip**, since `redeem_promo` is where the global and
  per-user caps are enforced atomically.
* The discount applies to the **gross fare**, and `final_fare` is the discounted
  amount the rider pays.
* **DECISION NEEDED — who funds the discount.** If the platform funds it, the
  driver's settlement is computed on the *undiscounted* fare and the platform
  absorbs the difference. If the driver co-funds it, they earn less on promo
  trips. This changes driver economics and cannot be inferred from the code.
  Until it is answered, do not implement promos.

## 8. How are cancellation fees handled?

`Trip.cancellation_fee` exists as a column and is **never written**. Worse, the
ops dashboard *aggregates* it (`"cancellation_fees": cancellation_total`) and
offers it as a revenue line type, so a structural zero is being reported as a
business metric. ADR-0007 states these fields "are set" by the cancel paths;
`cancelled_by` and `cancellation_reason` are, `cancellation_fee` is not. That ADR
line is wrong and should be corrected regardless of what happens next.

Cancellation fees are **out of scope for fare finalisation** — a cancelled trip
has no journey to meter. Two independent actions:

1. Immediately: stop reporting `cancellation_fees` as a metric, or label it
   explicitly as not implemented. Reporting a structural zero is worse than
   reporting nothing.
2. Separately: implement the MVA-2020 cancellation policy, which needs its own
   design (who cancelled, after how long, was the driver en route). The
   `cancelled_by` enum landed earlier makes this possible.

## 9. Which amount goes to payment capture?

`final_fare` once it exists, and it is already reached correctly: both
`_create_payment_on_complete` and `POST /payments/create-order/` read
`trip.final_fare or trip.estimated_fare`. Writing `final_fare` inside the
completion transaction makes every one of those paths capture the final amount
with no change to them.

For a promo trip, capture is the **discounted** amount.

## 10. Which amount drives commission?

`credit_driver_wallet` reads `trip.final_fare or trip.estimated_fare` and
multiplies by `commission_percent_for_trip(trip)`. So commission follows the fare
automatically — with the ordering requirement from question 1, and one addition
that matters more than it looks:

**Settlement must stamp the gross fare, the commission rate and the commission
amount on the row it writes.** Today it stores one net-or-commission amount and
nothing else, so what the platform took is only recoverable by re-deriving it
from a rate card that may have changed. An immutable fare with a recomputable
commission is not an auditable system. This is a schema addition to the
settlement ledger and should land with finalisation, not after.

For a promo trip, **DECISION NEEDED** per question 7: commission on the gross or
on the discounted amount.

## 11. Which amount drives driver settlement?

The same `final_fare`, minus the commission computed in question 10. The
definition is already consistent for cash and online — cash records a commission
debit because the driver holds the fare, online records a net credit because the
platform does — and the driver-earnings fix now reports both against one
definition of net.

Unchanged by finalisation: the driver wallet is debited for withdrawals at
request time, not at settlement, so nothing about payouts depends on this.

## 12. Which amount appears on the GST receipt?

`ride/receipts.py` reads `trip.final_fare or trip.estimated_fare` and extracts
GST **inclusive** of the total: `gst = total * rate / (100 + rate)`, using the
rate card at issue time. That is correct and needs no change — but it means a
wrong total silently produces a wrong tax document, which is why a receipt must
never be issued before `final_fare` is settled.

Receipt issuance is already a Celery task fired on commit, so it necessarily runs
after `final_fare` is written. Corrections use `issue_receipt_version`, which
already exists: a new version, never an edit.

## 13. Which amount appears in GMV?

`admin_dashboard/views.py` sums `item["fare"]`, derived from `final_fare or
estimated_fare`, for completed trips. So GMV follows finalisation automatically.

Two cautions. First, **GMV will step when metering starts**, and the step is a
measurement change, not growth — the executive report needs to be able to
separate metered from estimate-billed trips, which is another reason `fare_basis`
must land first. Second, platform revenue in the same report re-derives commission
from `commission_percent_for_trip` at *render* time, so editing a rate card today
silently changes last month's reported revenue. Stamping commission (question 10)
fixes the report as a side effect.

## 14. How are already-created Cashfree orders handled if the final fare changes?

They cannot exist. The order is created by the rider's app **after** completion
via `POST /payments/create-order/`, which requires `status == 'completed'`, and
`final_fare` is written inside the completion transaction. By the time an order
can be created, the fare is already immutable.

Two edge cases remain, and both are already handled or easy:

* **A rider on the payment screen when nothing has changed** — the endpoint
  refuses with `ALREADY_PAID` if a completed payment exists, and re-uses the
  latest `Payment` row otherwise.
* **A post-hoc correction** (dispute, ops adjustment) — this must never reprice
  an existing order. It is a refund or an additional charge with its own row and
  a new receipt version.

The one rule to preserve: **no gateway call inside the completion transaction.**
The current code comments record why — a Cashfree timeout used to fail trip
completion, and a rollback left orphaned orders at Cashfree. Finalisation must
not reintroduce that.

## 15. What happens on cash rides?

Completion creates a `Payment(method='cash', status='pending')` and immediately
calls `credit_driver_wallet`, which writes a **commission debit** because the
driver is holding the fare. The driver's later "confirm cash" tap calls
`credit_driver_wallet` again, which is a no-op through the idempotency key.

So a cash ride settles on whatever fare exists at completion — which makes
finalisation *more* important for cash, not less: the driver physically collects
the number the app shows them. If the app shows the estimate and the platform
settles on a metered fare, the driver is out of pocket by the difference on every
trip.

**Therefore: metering must not ship for cash rides until the driver app shows the
metered amount at completion.** That is a client change, and it gates the cash
path specifically. Rolling out metering online-first, cash-second, is the
defensible sequence.

## 16. What happens if completion is retried?

Completion is a WebSocket action guarded by a state machine inside
`select_for_update`. The transition table only allows `completed` from
`in_progress`, so a second completion for the same trip fails the transition and
changes nothing. That guard is already correct and is the primary protection.

Behind it, the side effects are individually idempotent:
`_create_payment_on_complete` returns early if `trip.payments.exists()`;
`credit_driver_wallet` is keyed on `TRIP_{id}_EARNING`; `issue_receipt` returns
the existing receipt rather than creating a version.

Finalisation must join that set, which is question 17.

## 17. How is fare finalisation made idempotent?

By making the write conditional on the column being unset, inside the same locked
transaction:

```
trip = Trip.objects.select_for_update().get(id=...)     # already held at completion
if trip.final_fare is None:
    trip.final_fare = computed
    trip.fare_basis = basis
    trip.save(update_fields=['final_fare', 'fare_basis'])
```

That is sufficient because the row lock is already held for the whole completion
transaction, so no second writer can interleave. It is also the property that
makes the fare immutable: the *only* write site refuses to overwrite.

The observe-only actual metrics already work this way (`record_actuals` skips an
already-populated trip unless explicitly forced), and the fare shadow keeps one
row per trip, so at-least-once Celery redelivery cannot double-count. Those are
the same discipline applied one layer earlier.

A backfill or correction path must be a **separate, audited operation** — not a
`force=True` flag on the completion path.

## 18. What evidence is retained for a dispute?

After the GPS trail and the actuals work, considerably more than before:

| Evidence | Where | Status |
|---|---|---|
| The route actually driven | `TripLocationPoint`, one row per sampled fix, with `recorded_at`, `sequence` and `source_event_id` | **now exists** |
| Trail quality behind a distance | `coverage_ratio`, `max_gap_seconds`, `rejected_segments` | **now computed** |
| Measured distance and duration | `Trip.actual_distance_km` / `actual_duration_min` | **now written, observe-only** |
| Journey boundary | `accepted_at`, `reached_at`, `started_at` (OTP-verified), `completed_at` | already existed |
| The quote the rider accepted | `Trip.estimated_fare`, `FarePricing` breakdown, `surge_multiplier` | already existed |
| What metering would have charged | `TripFareShadow`, with both deltas attributed | **now exists** |
| The tax document | `Receipt`, versioned | already existed |
| Which basis was billed | `fare_basis` | **MISSING — must land before metering** |
| The commission actually charged | nowhere | **MISSING — see question 10** |

Two gaps block a defensible dispute process, and both are recorded above:
`fare_basis` and a stamped commission. Neither is a pricing decision; both are
small schema additions.

Privacy constraint that must survive all of this: the trail is location data
about real people. It is already excluded from logs — the structured logger
redacts coordinates and the shadow logs ids and amounts only — and any
dispute-review UI must be admin-authorised and access-logged. A retention period
for `TripLocationPoint` is **DECISION NEEDED**; indefinite retention of every
driver's movements is not a defensible default.

---

## What must land before the first metered fare, in order

1. **`fare_basis` on `Trip`.** Not monetary. Without it the change is unauditable.
2. **Stamped settlement** — gross fare, commission rate, commission amount, and a
   real trip FK on the settlement row. Not monetary; makes the existing ledger
   auditable.
3. **`quote_fare` callable with an explicit multiplier and rate card**, so
   finalisation cannot reprice surge at completion. Not monetary.
4. **Shadow data from real traffic**, with the fallback bands chosen from it.
5. **The promo decision** (question 7), because a discount has nowhere to live
   once a fare is immutable.
6. **The waiting-time and per-minute decisions** (question 5), because adding
   them after metering means changing fares twice.
7. Only then: write `final_fare` in the completion transaction, online rides
   first, cash rides only after the driver app shows the metered amount.

## Unresolved business decisions, collected

1. Fallback bands for degraded and absent telemetry (question 2, 3).
2. Whether pickup waiting time is billable, and the free threshold (question 5).
3. Whether the per-minute fare follows real elapsed time (question 5) — this is
   the largest single fare effect of metering.
4. Who funds a promo discount: platform or driver (question 7).
5. Cancellation fee policy, or removing the metric (question 8).
6. Commission basis on discounted trips (question 10).
7. Rounding: paise today, whole rupees is the Indian norm (original report).
8. `TripLocationPoint` retention period (question 18).

Nothing on this list can be resolved by reading more code.
