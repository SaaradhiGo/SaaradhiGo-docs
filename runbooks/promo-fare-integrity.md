# Promo fare integrity — investigation

- **Status:** investigation only. **Nothing implemented.** No code changed.
- **Date:** 2026-09-23
- **Branch:** `fix/promo-fare-integrity` (this document). The backend branch of the
  same name will be created when a funding model is approved.
- **Related:** [fare-finalisation-design](fare-finalisation-design.md)

## Correction to the earlier report

The overnight report said a rider "can enter a promo code, be shown a discount,
and then be charged the full fare." That overstated the live exposure. Tracing the
client confirms the rider **cannot enter a code at all** in the shipped app:

- `SaaradhiGo-mobile/lib/widgets/promo_code_field.dart` exists and is complete, but
  `PromoCodeField` is **not mounted in any screen** — a repo-wide search for its
  use outside its own file returns nothing.
- The booking payload in `websocket_service.dart` carries pickup, destination,
  addresses, distance, duration, vehicle type and payment method. **No promo
  field.**
- The fare breakdown widget renders a "Promo Discount" line only when the response
  contains `fareBreakdown['discount']`. `quote_fare` never returns that key, so
  the line never appears.

So the real finding is not an active overcharge. It is that **the feature looks
finished on both sides and is wired to nothing** — which is more dangerous in a
specific way: the remaining work looks like "mount the widget and add a field to
the payload," and doing only that would ship silent full-fare charging.

Live today, and worth fixing regardless of the funding decision: ops **can**
create and advertise promo codes from the admin console (`promo-codes/`), and the
executive dashboard reports `total_redemptions`, which is structurally always 0 —
the same failure mode as `cancellation_fee`.

## The lifecycle as it actually exists

```
discovery/display      NOWHERE. No campaign list endpoint, no rider-facing list.
eligibility            PromoCode: zone, validity window, min_fare, per-user cap,
                       global cap. All modelled.
selection              PromoCodeField widget exists but is not mounted.
validation             POST /api/v1/ride/promo/apply/ -> promos.apply_promo
                       Preview only. Explicitly does not redeem.
redeem_promo           IMPLEMENTED AND CORRECT. ZERO CALLERS.
Trip creation          no promo field on the frame, no discount column on Trip
FarePricing            base/distance/time/surge/total. No discount column.
payment                reads trip.final_fare or estimated_fare
completion             same
settlement             commission on that same amount
receipt                no discount line; GST extracted from the total
reports                admin trip detail reads PromoRedemption (always empty);
                       executive reports total_redemptions (always 0)
```

## The 20 questions

**1. Where is the promo code currently supplied by the client?**
Only to `POST /api/v1/ride/promo/apply/`, as `{code, fare, pickup_lat, pickup_long}`,
and only from a widget that is not mounted. It is **never** supplied at booking.

**2. What amount does the rider UI display before booking?**
The server's `quote_fare` total. `apply_promo` returns a `final_fare` alongside the
discount, but nothing in the shipped UI consumes it, so no discounted amount is
displayed anywhere today.

**3. Is the discount calculated anywhere?**
Yes — `promos._compute_discount`, and it is good: flat is `min(value, fare)`,
percent is `fare × value/100` capped by `max_discount_amount` and by the fare
itself, all `ROUND_HALF_UP` on `Decimal`. It is **only** ever called for a preview.
The result is never persisted and never reaches a charge.

**4. Is promo validation authoritative server-side?**
Partly, and this is the most important defect to fix before wiring anything.

The *code rules* are fully server-side: existence, active flag, validity window,
zone, min fare, per-user cap, global cap. Good.

The *fare the discount is computed against is supplied by the client.*
`apply_promo_endpoint` reads `fare` from the request body. For a preview that is
harmless. The moment redemption is wired to that number, a client can post an
inflated fare to obtain a larger percent discount than their real fare warrants —
bounded by `max_discount_amount` where set, unbounded where it is not.

**The rule this implies:** the server must recompute the quote itself — from
pickup, destination, vehicle type and its own validated distance — and apply the
discount to *that*. The client may send a code; it must never send an amount.

**5. Is redemption intended at booking or completion?**
**At booking**, and the code says so in three places: `PromoRedemption.trip` is
nullable specifically for "validate at booking time, attach to trip once trip is
created"; `redeem_promo` takes a `trip`; and the docstring describes reserving at
apply time and attaching the trip id when the booking lands. The caps are also
booking-shaped — a global cap has to be claimed when the rider commits, not
minutes later at completion.

This matters for question 11: if redemption is at booking, the discount is part of
the quoted fare the rider accepts, not an adjustment discovered later.

**6. Is redemption idempotent?**
**No.** `redeem_promo` inserts a `PromoRedemption` row unconditionally inside its
transaction and increments the counter. Calling it twice for the same
`(code, user, trip)` creates two rows, two counter increments, and two discounts.
There is no unique constraint and no idempotency key.

This is the single most important gap to close before wiring, because the trip
creation path has retries and reconnects. The fix is a unique constraint on
`(promo, trip)` (and a separate guard for the trip-less reservation case), not
application-level checking.

**7. Can the same user redeem concurrently?**
The global cap is safe: `select_for_update` on the `PromoCode` row serialises
concurrent redemptions, and the count is checked inside that lock. Good design.

The **per-user cap is not safe**. `user_count` is a `COUNT(*)` over
`PromoRedemption` for that user, taken inside the promo lock — which does serialise
against other redemptions *of the same code*. So concurrent redemptions of the same
code by the same user are in fact serialised, and the per-user cap holds. But a
user redeeming **two different codes** concurrently is not serialised at all — that
is fine today because caps are per code, and would only matter if a
"one promo per rider per day" style rule is ever added.

The real concurrency hole is question 6: without a unique constraint, a retry is
indistinguishable from a legitimate second redemption.

**8. Is usage limited globally / per user?**
Yes, both are modelled and enforced: `max_total_redemptions` (NULL = unlimited)
with `redemption_count`, and `max_per_user_redemptions` (default 1). The global
check is correctly done under a row lock.

**9. Where should applied promo identity be persisted?**
`PromoRedemption` already is the join and should remain the authority — it has
`promo`, `user`, `trip` and `discount_amount`. What is missing is a **denormalised
pointer on the money record** so that explaining a fare does not require a reverse
lookup through a table that can be swept. That belongs on the fare snapshot
proposed in [fare-basis-design](fare-basis-design.md), as a promo id plus the
code string as it was at booking, because codes can be renamed.

**10. Where should the discount amount be persisted?**
Twice, deliberately, and they mean different things:

- `PromoRedemption.discount_amount` — the campaign's record, for "what did this
  campaign cost us".
- the fare snapshot's `discount_amount` — the trip's record, for "why is this
  rider paying this". This one must be immutable.

Not on `PromoCode` (that is configuration) and not derived at read time from
`discount_type`/`discount_value` (that is the recomputation-from-mutable-config
problem this whole workstream exists to remove).

**11. Should the quoted fare be gross-before-discount or rider-payable?**
**Both, stored separately.** Every downstream consumer needs a different one:

| Consumer | Needs |
|---|---|
| payment capture | rider payable |
| GST receipt | rider payable (tax follows what was charged) |
| driver settlement | depends on funding model (question 12/13) |
| GMV | gross, or GMV drops when marketing spends more |
| campaign cost | the discount |

A single number cannot serve those. `gross_fare`, `discount_amount` and
`rider_payable` as three stored fields, with `rider_payable = gross − discount`
enforced, is the minimum.

**12. Which amount should commission use?**
**This is the funding decision and is not ours to make.** See below.

**13. Which amount should driver earnings use?**
Same decision. Note the constraint the current code imposes: `credit_driver_wallet`
reads one amount and applies one percentage, and it is keyed on
`TRIP_{id}_EARNING` so the first call fixes the basis permanently. Whatever is
chosen must be decided *before* settlement runs, not reconciled afterwards.

**14. Which amount should GST / tax use?**
**Rider payable.** Tax follows what the customer was actually charged; charging GST
on an amount the rider did not pay is not defensible to them or to a tax audit.
`receipts.py` already extracts GST inclusive of the total it is given, so feeding
it `rider_payable` is correct with no change to the extraction.

If a funding model makes the platform pay the driver on the gross, the difference
is platform marketing spend, not rider consideration, and is not part of the
rider's taxable amount.

**15. Which amount should appear on the receipt?**
All three lines: gross fare, the discount with its code, and the rider-payable
total from which GST is extracted. The mobile fare-breakdown widget already has a
"Promo Discount" line waiting for a `discount` key, so the client is ready.

**16. What happens on cancellation?**
Undefined today, because nothing redeems. Recommended, and it is an engineering
call rather than a product one: a cancelled trip **releases** the redemption —
delete the `PromoRedemption` row and decrement the counter in one transaction, so
the rider keeps their entitlement and the campaign is not charged for a ride that
did not happen. That requires the decrement to be as carefully locked as the
increment. If a cancellation fee is ever charged, the promo does not apply to it.

**17. What happens if payment fails?**
The ride happened, so the entitlement was consumed: the redemption **stands** and
the unpaid amount is `rider_payable`, which is the dues path. Releasing the
redemption on payment failure would let a rider farm a promo by failing payment.

**18. What happens if a ride never starts?**
Same as cancellation — release. The distinguishing line is `started_at`: no rider
aboard means no ride consumed, which is the same boundary the actual-metrics work
already uses.

**19. What happens if the promo expires after booking?**
The rider booked while it was valid, so **the booking keeps the discount.** This is
why the discount must be *stored* at booking rather than recomputed: a validity
check at completion would revoke a discount the rider was promised. Storing it
makes the right behaviour the automatic one.

**20. What happens if an admin edits or deactivates the promo after booking?**
Identically: the stored discount stands. Deactivation stops *new* redemptions and
must not touch existing ones. `PromoCode` is referenced with `on_delete=PROTECT`
from both `PromoRedemption` and the zone FK, so a code with history cannot be
deleted — that part is already right. Editing `discount_value` after bookings
exist is safe *only* because the amount will be stored; today, with nothing
stored, it would silently rewrite history.

## The funding decision — yours, not ours

What the existing code *implies*, stated without choosing:

- `credit_driver_wallet` computes commission from one fare amount, and `quote_fare`
  has no notion of a discount. So the current structure has **no opinion**; it
  would apply commission to whatever single number it is handed.
- `PromoCode` has no funding, sponsor or cost-centre field, and no
  `platform_funded` flag. There is no partial evidence of an intended model.
- The admin console's promo page and the executive report's `total_redemptions`
  treat promos as a marketing metric rather than a driver-economics lever, which is
  weak evidence in favour of platform funding — but it is weak, and it is a UI
  placement, not a decision.

The three models, analysed and **not** selected:

**A. Platform-funded.** Driver settles on `gross_fare`; commission on gross; the
platform absorbs `discount_amount`. Driver economics are unaffected by marketing,
which makes driver supply predictable and promos safe to run aggressively.
Platform revenue per promo trip falls by the full discount, and can go negative on
a large discount plus a small fare. Requires no change to driver-facing
communication.

**B. Driver-funded / shared.** Driver settles on `rider_payable`; commission on
`rider_payable`. The platform's take is protected, but the driver earns less on
promo trips without having agreed to the campaign. This needs driver consent in
terms and visible per-trip disclosure in the driver app, or it becomes a trust and
possibly a regulatory problem. It also makes promo trips less attractive exactly
when the platform most wants them accepted.

**C. Configurable per campaign.** A funding field on `PromoCode`, e.g.
`funded_by ∈ {platform, driver, shared}` with a share percentage. Most flexible,
and the reporting and settlement code must then handle every mode — which is the
real cost, since each mode needs its own tests and its own driver disclosure.

**What is needed from you:** one of A, B or C. Nothing else on this page is
blocked; everything else is engineering.

## The smallest correct data model

Assuming redemption at booking (question 5), and no funding model chosen:

**1. `PromoRedemption` gains a uniqueness guarantee.** One redemption per
`(promo, trip)`, enforced by the database, plus an idempotency key for the
trip-less reservation window. This is the fix for question 6 and it is needed under
every funding model.

**2. Three amounts on the fare snapshot**, not on `Trip` directly — see
[fare-basis-design](fare-basis-design.md): `gross_fare`, `discount_amount`,
`rider_payable`, with the promo's id and its code string as at booking.

**3. Nothing on `PromoCode`** unless model C is chosen, in which case exactly one
field: `funded_by`.

**4. No new table.** `PromoRedemption` plus the fare snapshot cover it. A separate
"promo ledger" would be a second financial history, which
[settlement-snapshot-design](settlement-snapshot-design.md) argues against.

Also needed, and independent of all of the above:

- **the server must quote the fare itself** when applying a promo (question 4);
- **the null-trip redemption sweep** described in `PromoRedemption`'s docstring
  does not exist. Either implement it or remove the reservation flow;
- **stop reporting `total_redemptions`** as a live metric, or label it as not
  implemented, exactly as with `cancellation_fee`.

## Recommended sequencing

1. Merge nothing. This document first.
2. Funding decision (A/B/C).
3. `PromoRedemption` uniqueness + the sweep, or removal of the reservation flow.
   Independent of funding and safe to do first.
4. Fare snapshot with the three amounts (part of the FareBasis work).
5. Server-side quoting in the promo endpoint; client sends a code, never an amount.
6. Only then: wire redemption into trip creation, mount the widget, and add the
   code to the booking payload — in that order, so the server is ready before the
   client can send anything.
