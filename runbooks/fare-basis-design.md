# FareBasis — the minimum immutable fare snapshot

- **Status:** design only. **Nothing implemented.** No column added, no migration
  written, no billing behaviour changed.
- **Date:** 2026-09-23
- **Depends on:** [promo-fare-integrity](promo-fare-integrity.md),
  [fare-finalisation-design](fare-finalisation-design.md)
- **Requirement:** once finalised, historical fare economics must not depend on
  querying today's `RateCard`.

## The finding that shapes the whole design

**A fare snapshot already exists.** `ride.FarePricing` is created inside the same
transaction as the `Trip`, in `_create_trip`, and stores:

```
trip_id, base_fare, distance_fare, time_fare, surge_multiplier, total_fare
```

Nothing anywhere updates it — a repo-wide search finds exactly one write site
(`consumers.py:657`) and three read sites in `receipts.py`. It is already
append-only in practice, and `receipts.py` already calls it "the FarePricing
snapshot".

So this is not a new-table problem. It is a **completeness** problem: the existing
snapshot is missing the few facts needed to explain a fare without re-reading
configuration, and it is written only at booking, never at finalisation.

That matters for the recommendation below: extending a table that three code paths
already treat as the fare record is a much smaller change than introducing a
parallel one, and it cannot create a second disagreeing source of truth.

## What "explain this fare" actually requires

The arithmetic `quote_fare` performs, in order:

```
subtotal = base_fare + (per_km × distance_km) + (per_min × duration_min)
if night window and night_surge > 1:      subtotal ×= night_surge
if dynamic surge ≠ 1:                     subtotal ×= dynamic
if combined multiplier > surge_cap:       subtotal = pre_surge × surge_cap
if subtotal < min_fare:                   subtotal = min_fare
total = round(subtotal, 2)
```

To reconstruct that from stored data alone, the snapshot must answer: what were the
components, what multiplier was applied, **did the minimum-fare floor bind**, and
which rate schedule produced it.

The third one is the non-obvious requirement. When the floor binds, the components
do **not** sum to the total — so without a flag, a correct snapshot looks
internally inconsistent and a dispute cannot be answered. Same for the surge cap:
when it binds, the multiplier is not the product of night and dynamic.

## The minimum model

**Keep** on `FarePricing`: `base_fare`, `distance_fare`, `time_fare`,
`surge_multiplier`, `total_fare`.

**Add** — each with the reason it cannot be derived or looked up later:

| Field | Why it is required |
|---|---|
| `quoted_distance_km` | `distance_fare ÷ per_km` needs a rate card to invert. Also the quantity a rider disputes. |
| `quoted_duration_min` | same |
| `min_fare_applied` | when the floor binds, components do not sum to the total. Without this the snapshot reads as broken. |
| `surge_cap_applied` | when the cap binds, the multiplier is not night × dynamic. Same reason. |
| `night_surge_applied` | separates a time-of-day surcharge from demand surge. The dynamic component is then derivable as `surge_multiplier ÷ night_surge`, so the split needs only this one boolean plus the night multiplier below. |
| `night_surge_multiplier` | the rate card's value **as applied**; the card can change. |
| `rate_card_version` | identity of the schedule used. |
| `zone_code` | which city/zone priced it. A code, not an FK, so a renamed or reparented zone cannot rewrite history. |
| `vehicle_type` | as a string, for the same reason. |
| `pricing_source` | `db` or `default`. A fare quoted from the hardcoded fallback defaults **must** be identifiable; today it is invisible. |
| `gross_fare` | the fare before any discount. |
| `discount_amount` | see [promo-fare-integrity](promo-fare-integrity.md) Q10/11. |
| `rider_payable` | `gross_fare − discount_amount`. Stored, not derived, because it is what payment captured. |
| `promo_id`, `promo_code` | the id for joins, the code string as at booking because codes can be renamed. |
| `fare_basis` | `metered`, `metered_degraded`, `estimate_no_telemetry`, `estimate_legacy`. The field that makes metering auditable and the fallback rate measurable. |
| `finalized_at` | null until finalisation; non-null means immutable. Also the flag that makes the write idempotent. |

`total_fare` keeps its current meaning (the quoted total) and `rider_payable`
becomes the amount charged. On a trip with no promo they are equal.

## Deliberately NOT added, and why

Adding these "just in case" is how a snapshot becomes unreadable.

- **`per_km_fare`, `per_min_fare`, `min_fare`, `surge_cap_multiplier`** — the
  per-unit rates. `distance_fare` with `quoted_distance_km` gives the effective
  per-km rate by division; the flags above cover the cases where the floor or cap
  bound. Storing the rates as well is redundant for explanation and gives two
  places to disagree. **Revisit only if** a rate card is ever edited in place
  rather than versioned — see the open risk below.
- **`tax` / `gst_amount` / `gst_rate`** — `Receipt` already stores `total_fare` and
  `gst_amount`, is versioned, and is the legal document. Duplicating tax onto the
  fare snapshot creates two tax numbers for one ride. Tax stays on `Receipt`.
- **`currency`** — the platform is single-currency; `quote_fare` has no currency
  concept and every caller assumes INR. A column that is always `'INR'` is not
  auditability, it is a column. Add it with the second currency, not before.
- **`waiting_component`** — there is no waiting-time charge anywhere in the
  pricing model. A column for a charge that does not exist invites someone to
  populate it without a policy.
- **`actual_distance_km` / `actual_duration_min`** — already on `Trip`, already
  populated observe-only. Copy them into the snapshot **only at finalisation**, and
  only if a metered fare was billed from them; before that they are measurements,
  not fare inputs.

## Where it lives, and the alternative rejected

**Recommendation: extend `FarePricing`, and add a `version` column.**

It is already the fare snapshot, already append-only, and already read by receipts.
The FK (rather than OneToOne) from `FarePricing` to `Trip` is fortunate here: it
allows a second row at finalisation without destroying the booking row, which is
exactly the audit trail wanted — "quoted this, finalised that". `receipts.py`
already reads `.order_by('-id').first()`, so it would pick up the finalised row
with no change; adding `version` makes that ordering explicit rather than
incidental.

**Rejected: a new `TripFareBasis` table.** It would duplicate five columns that
`FarePricing` already holds and would leave three read sites pointing at the old
one. Two tables describing one fare is the problem this work exists to remove.

**Rejected: columns on `Trip`.** `Trip` is the lifecycle entity and is mutated
throughout a ride. Money that must be immutable does not belong on a row that is
saved a dozen times per trip; the narrow-`update_fields` discipline the actuals
work needed is evidence of how easily that goes wrong.

## Immutability, concretely

1. The finalisation row is written **once**, inside the completion transaction,
   while the `Trip` row lock is already held, and only when no finalised row
   exists (`finalized_at IS NULL` check inside the lock). That is the same
   idempotency shape the actual-metrics task already uses.
2. `finalized_at` non-null means no further writes. Enforced in the one service
   that writes it, and worth a database-level guard: a partial unique index on
   `(trip_id)` where `finalized_at IS NOT NULL`, so a second finalisation is
   refused by PostgreSQL rather than by a code path.
3. Corrections are a **new** `FarePricing` version plus a new `Receipt` version,
   never an edit. Both models already support versioning.
4. Every downstream consumer reads the snapshot, never a rate card:
   payment capture, commission, settlement, receipt, GMV.

## The open risk this design cannot fix by itself

`RateCard` rows are versioned and effective-dated, but **nothing prevents editing
one in place.** `rate_card_version` is only a trustworthy pointer if cards are
immutable once used. Two options, both cheap, and this is an engineering decision
rather than a product one:

- make `RateCard` append-only for any card that has priced a trip (a save guard
  plus an admin restriction), or
- store the per-unit rates in the snapshot after all, accepting the redundancy.

The first is better: it keeps the snapshot lean and fixes the underlying problem
rather than working around it. The second is a fallback if in-place edits are
operationally necessary.

## Sequencing

1. Add the columns and the `version` column. **Non-monetary**: nothing reads them
   yet.
2. Populate them at **booking** for every new trip, from the quote that is already
   being computed. Still non-monetary — the booking snapshot simply becomes
   complete, and this is what starts building the audit trail before any behaviour
   changes.
3. Add `fare_basis` + `finalized_at` and write them at completion with
   `fare_basis='estimate_legacy'`, still copying the quoted total. **Still
   non-monetary**, and it makes the eventual switch a one-line change of which
   amount is written.
4. Only then, and only after the promo funding decision and QA shadow evidence:
   write a metered `rider_payable`.

Steps 1–3 are safe to do now and are worth doing regardless of what is decided
about metering: they end the recomputation of historical money from mutable
configuration, which is the actual requirement.
