# Settlement snapshot — the minimum immutable settlement record

- **Status:** design only. **Nothing implemented.** No model added, no migration
  written, no settlement behaviour changed.
- **Date:** 2026-09-23
- **Depends on:** [fare-basis-design](fare-basis-design.md)
- **Related:** [driver-earnings-root-cause](driver-earnings-root-cause.md)
- **Requirement:** settlement must stop being recomputable from mutable rate cards,
  and there must remain **one** authoritative financial history.

## What exists today

`driver.utils.credit_driver_wallet` writes, per completed trip:

**1. `rider.WalletTransaction`** — the row that moves money.

```
user_id, amount, txn_type, status, purpose, reference_id='TRIP_<id>',
idempotency_key='TRIP_<id>_EARNING'
```

The driver's balance is updated in the same transaction. `idempotency_key` is
unique, which is what makes the seven callers of `credit_driver_wallet` safe. This
is the **authoritative ledger**: it is the only thing whose sum equals a balance.

**2. `payments.TransactionHistory`** — a display row.

```
trip_id, user_id, driver_id, amount, method, status, txn_type, user_name, gateway_*
```

What neither of them records: the **gross fare**, the **commission rate** and the
**commission amount**. Cash stores the commission as the amount; online stores the
net. So the platform's take on a trip is recoverable only by re-deriving it from a
`RateCard` that may have changed since — which is exactly the property to remove.

Also missing: `WalletTransaction` has **no trip FK**. The only link is the string
`TRIP_<id>` in `reference_id`, which the driver-earnings fix has to parse.

## Can `TransactionHistory` carry this safely? No.

It is tempting because it already has `trip_id`, `driver_id` and an amount. Three
reasons not to promote it:

1. **It has no `purpose`, and that has already caused a bug.** A cash commission
   debit and a refund reversal debit are both `txn_type='debit'` with a trip and an
   amount. The driver-earnings work had to filter on `WalletTransaction.purpose`
   precisely because `TransactionHistory` cannot distinguish them. Adding
   settlement economics to a table that cannot identify its own row types makes the
   ambiguity more expensive, not less.
2. **It is written from at least four places** with different meanings — trip
   payment, driver credit, refund reversal, withdrawal — and none of them is
   idempotent. Settlement economics must be written exactly once.
3. **It is a display log, and it does not drive balances.** Making it authoritative
   would create a second financial history competing with `WalletTransaction`,
   which is the specific outcome to avoid.

Its honest role is what it already is: a human-readable transaction feed. It should
stay that, and ideally gain a `purpose` field so it stops being ambiguous.

## Recommendation: one snapshot, no new ledger

**`WalletTransaction` remains the single authoritative financial history.** Nothing
in this design writes a second balance-affecting record, and no amount is duplicated
as a second source of truth for money movement.

Add **`TripSettlement`** — one row per trip, recording the *economics* of a
settlement rather than the *movement* of money:

| Field | Purpose |
|---|---|
| `trip` (OneToOne) | the real foreign key that `WalletTransaction` lacks. One settlement per trip. |
| `driver` (FK) | who was settled. Denormalised deliberately: a trip's driver assignment is mutable, the settlement's is not. |
| `wallet_transaction` (FK) | the ledger row that actually moved the money. This is what makes the snapshot an *explanation* rather than a duplicate. |
| `gross_fare` | the fare the settlement was computed on, copied from the fare snapshot. |
| `commission_percent` | the rate **as applied**, not a pointer to a card. |
| `commission_amount` | the money taken. Stored, not derived, because rounding matters. |
| `driver_net` | `gross_fare − commission_amount`. Stored so the identity is asserted at write time. |
| `payment_method` | `cash` / `online` / `wallet` — decides the ledger's direction and who holds the fare. |
| `settled_at` | when the economics were fixed. |
| `version` | for corrections. A correction is a new row, never an edit. |

That is **ten fields and one table**, and it is the whole answer to "what did the
platform take from this ride and why".

### Why a separate table rather than columns on `WalletTransaction`

`WalletTransaction` is generic: rider wallet top-ups, refunds, promo credits,
support credits, withdrawals and payouts all use it. Putting `gross_fare`,
`commission_percent` and `driver_net` on it would leave those columns null on the
large majority of rows and would place trip-specific economics on a rider's wallet
top-up. The FK in the other direction keeps each model honest: the ledger says what
moved, the snapshot says why.

### Tax

Not on this record. GST is a rider-side tax on the fare and already lives on the
versioned `Receipt`. The platform's commission has its own tax treatment which is
an accounting matter, not a per-trip settlement field, and inventing a column for
it now would be guessing at a policy nobody has stated. If commission GST is ever
required per trip, it is one field added then, from a stated rule.

## Immutability and idempotency

The settlement snapshot must be written in the **same transaction** as the
`WalletTransaction` it explains, or the two can disagree after a crash. Concretely,
inside `credit_driver_wallet`'s existing atomic block:

1. The `WalletTransaction` insert already provides the idempotency guard: a unique
   `idempotency_key` of `TRIP_<id>_EARNING`, and the existing code returns early on
   `IntegrityError`. The `TripSettlement` insert goes **after** it, inside the same
   block, so it is created exactly once for the same reason.
2. `trip` being OneToOne gives a database-level guarantee of one settlement per
   trip, independent of that logic.
3 No update path. Corrections insert `version + 1` and leave the prior row intact.

This is a small change to a function with seven callers, and its safety rests
entirely on the idempotency key that already exists — worth stating plainly,
because it is what makes the change reviewable.

## The backfill question

Existing settled trips have a `WalletTransaction` and no snapshot. Two honest
options:

- **Backfill what is knowable, and mark it as reconstructed.** Gross comes from the
  trip's fare, commission from the ledger row (cash: the amount; online:
  `gross − amount`), which is exactly what the driver-earnings report now does. The
  `commission_percent` is then a derived figure, not the applied one — so it must be
  recorded as reconstructed, with a `source` value distinguishing it from a
  natively-written row.
- **Backfill nothing.** Historical trips keep being explained by the earnings
  report's derivation, and only new trips get snapshots.

**Recommendation: backfill, marked.** A financial table with a silent behaviour
change at a cutover date is worse than one that says which rows were reconstructed.
This is an engineering call and needs no product decision.

## What this fixes downstream, for free

- **Admin revenue stops drifting.** `admin_dashboard` currently calls
  `commission_percent_for_trip` at *render* time, so editing a rate card today
  silently changes last month's reported platform revenue. Reading
  `TripSettlement.commission_amount` ends that.
- **Driver earnings stop being derived.** The read-only earnings fix has to infer
  commission from the ledger's direction. With a snapshot it reads three columns.
- **Disputes become answerable**: gross, rate, commission, net, method and
  timestamp, from one row, with a link to the ledger entry that moved the money.

## Sequencing

1. `TripSettlement` model and migration. Nothing reads it. **Non-monetary.**
2. Write it inside `credit_driver_wallet`'s existing transaction. Still
   non-monetary: no amount changes, one row is added alongside the existing two.
3. Backfill historical trips, marked as reconstructed.
4. Point the driver-earnings report and the admin revenue report at it, and delete
   the derivation code. This is where the drift actually stops.
5. Add `purpose` to `TransactionHistory` so the display log stops being ambiguous.
   Independent of the above and worth doing whenever convenient.

Steps 1–3 are additive and safe to do before any fare policy is settled. Step 4 is
the one that changes numbers on a dashboard — not because any money moved, but
because the numbers stop moving on their own.
