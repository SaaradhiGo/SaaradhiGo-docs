# Promo — platform-funded MVP implementation design

- **Status:** design. **Nothing implemented.** No migration written, no billing changed.
- **Date:** 2026-09-23
- **Decision applied:** platform-funded promo (approved). Configurable/shared
  funding is **not** implemented.
- **Depends on:** [promo-fare-integrity](promo-fare-integrity.md) (the investigation),
  [fare-basis-design](fare-basis-design.md)

## The approved economics, stated as arithmetic

```
gross_fare        = canonical quote_fare(...)          <- commission basis, driver economics
discount_amount   = promo applied to gross_fare        <- funded by the platform
rider_payable     = gross_fare - discount_amount       <- what we capture

commission_amount = gross_fare x commission_percent    <- NOT reduced by the discount
driver_net        = gross_fare - commission_amount     <- unchanged by the promo
platform_take     = commission_amount - discount_amount  (can be negative; that is
                                                          the marketing spend)
```

The driver's economics are computed from `gross_fare` and are **identical** to a
non-promo trip of the same distance. That is the whole point of platform funding,
and it is one line in `credit_driver_wallet`: keep passing the gross.

## One blocker you need to know about before this can ship

**Tax is unavoidably touched, and tax is not authorized.**

`receipts.py` extracts GST *inclusive* from the total it is given:
`gst = total x rate / (100 + rate)`. Today that total is
`final_fare or estimated_fare`. The moment a rider pays `rider_payable` instead of
`gross_fare`, the receipt either:

- keeps showing `gross_fare` — issuing a tax document for an amount the rider did
  not pay, or
- shows `rider_payable` — which changes the GST figure, i.e. a tax behaviour change.

There is no third option. So **promo cannot go live before the tax review**, and
"keep tax behaviour unchanged" is not achievable alongside a real discount. This
is not a reason to delay the schema work below (which is inert), only the
go-live.

My recommendation for that review, for what it is worth: tax follows what the
customer was charged, so `rider_payable`; the platform's discount is marketing
spend, not rider consideration. But it is your call and it is not implemented.

## Architecting for future funding without implementing it

The hook is **one stamped field, not a switch**:

`funding_model` on the promo application record, written as the constant
`'platform'` today. No branching on it, no configuration, no `funded_by` column on
`PromoCode`.

Why this is the right amount of future-proofing: when a second model is
introduced, every historical row already says which economics produced it, so old
trips remain interpretable and no backfill is needed. Without the stamp, the day
model B arrives every historical promo trip becomes ambiguous. Adding a *switch*
now would mean shipping and testing branches nobody has approved.

## Data model

### 1. `PromoRedemption` — constraints, not conventions

Add:

```python
constraints = [
    models.UniqueConstraint(
        fields=['promo', 'trip'],
        name='promo_one_redemption_per_trip',
    ),
    models.UniqueConstraint(
        fields=['promo', 'user'],
        condition=models.Q(trip__isnull=True),
        name='promo_one_open_reservation_per_user',
    ),
    models.CheckConstraint(
        check=models.Q(discount_amount__gte=0),
        name='promo_discount_non_negative',
    ),
]
```

Plus `funding_model = models.CharField(max_length=16, default='platform')`.

The first constraint is the one that matters: **one redemption per (promo, trip),
enforced by PostgreSQL.** Trip creation has retries and reconnects, so a retry is
otherwise indistinguishable from a legitimate second redemption. Application-level
checking cannot close this; two concurrent requests both read "not redeemed yet"
and both insert.

The second only matters if the trip-less reservation flow described in the model's
docstring is kept. **Recommendation: drop that flow.** It has no sweeper (the
cleanup task in the docstring does not exist), it creates rows that count against
caps without a ride, and redeeming at booking makes it unnecessary. If it is
dropped, the second constraint goes with it.

### 2. Fare snapshot — three amounts and the promo identity

Per [fare-basis-design](fare-basis-design.md), on `FarePricing`:
`gross_fare`, `discount_amount`, `rider_payable`, `promo_id`, `promo_code`
(the code string as at booking, because codes can be renamed).

`rider_payable` is **stored, not derived**, because it is the amount payment
captured and it must not change if the discount is ever recomputed.

### 3. Nothing else

No new table. No column on `PromoCode`. No Redis key.

## Concurrency and idempotency, case by case

`redeem_promo` already row-locks the `PromoCode` with `select_for_update` and
checks both caps inside that lock. That part is correct and is kept. What each
hazard needs:

| Hazard | What stops it |
|---|---|
| **Booking retry** (same trip, request replayed) | `UniqueConstraint(promo, trip)`. The second insert raises `IntegrityError`; the caller treats it as "already redeemed" and proceeds. |
| **Double click** (two bookings, two trips) | The per-user cap under the promo row lock. With `max_per_user_redemptions=1` the second is refused. **With a cap above 1 it is not**, and that is a *trip* idempotency gap, not a promo one — a rider can create two trips. Named here rather than papered over. |
| **Concurrent redemption of the same code** | `select_for_update` on the `PromoCode` row serialises them; the count is read inside the lock. |
| **Global usage-limit race** | Same lock, plus `F('redemption_count') + 1`. Belt and braces: a `CheckConstraint` that `redemption_count <= max_total_redemptions` would make an over-count impossible at the database, not merely unlikely. |
| **Per-user limit race** | Serialised by the same lock for the same code. Two *different* codes concurrently are not serialised — harmless while caps are per code, and it must be revisited if a "one promo per rider per day" rule is ever added. |
| **Expired promo** | Re-validated inside `redeem_promo`, not trusted from the preview. |
| **Disabled promo** | Same; `is_active` is re-read under the lock. |

**No Redis anywhere in this path.** Locking is PostgreSQL row locks and uniqueness
is database constraints. A cache is not a concurrency primitive, and promo caps
are money.

## Server-side authority

The client may submit **`promo_code`** and nothing else about pricing.

```
POST /ride/promo/apply/   (preview)       {code, pickup, destination, vehicle_type}
booking frame             (redeem)        {..., promo_code}
```

The server:

1. computes the canonical quote itself from pickup, destination, vehicle type and
   its own validated distance — **never from a client-supplied `fare`**;
2. validates eligibility against that quote;
3. computes the discount server-side;
4. persists the redemption and the three amounts in the **same transaction as the
   Trip**.

This replaces the current preview endpoint's `fare` body parameter, which is the
exploitable part today: a client can inflate `fare` to obtain a larger percentage
discount, bounded only by `max_discount_amount` where one is set.

## Lifecycle answers under platform funding

| Event | Behaviour |
|---|---|
| Cancellation before `started_at` | Release: delete the redemption and decrement the counter in one transaction. The rider keeps the entitlement; the campaign is not charged for a ride that did not happen. |
| Payment failure after the ride | Redemption **stands**. The ride was consumed; the unpaid amount is `rider_payable` and goes down the dues path. Releasing here would let a rider farm promos by failing payment. |
| Ride never starts | Same as cancellation — release. The boundary is `started_at`, the same one the actual-metrics work uses. |
| Promo expires after booking | The stored discount stands. This is why the amount is stored rather than recomputed. |
| Admin edits or deactivates the promo | The stored discount stands. Deactivation stops new redemptions only. `on_delete=PROTECT` already prevents deleting a code with history. |

## Sequencing

1. `PromoRedemption` constraints + `funding_model`, and drop the reservation flow.
   **Inert** — nothing redeems yet.
2. Fare snapshot columns (part of the FarePricing work). **Inert.**
3. Server-side quoting in the promo endpoint; remove the `fare` body parameter.
   Preview-only, so no money moves.
4. **Tax review.** Blocking for everything below.
5. Wire `redeem_promo` into trip creation, writing the three amounts.
6. Client: add `promo_code` to the booking payload and mount `PromoCodeField`.

Steps 1–3 are safe now. Step 5 is the first one that changes what a rider pays,
and it must not precede step 4.
