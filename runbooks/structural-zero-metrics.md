# Structural-zero financial metrics — findings and recommendation

- **Status:** investigation + recommendation. **Nothing changed.**
- **Date:** 2026-09-23
- **Subject:** `cancellation_fees` and `total_redemptions` on the admin dashboards.

Both are presented as financial/operational figures. Both are **structurally
incapable of being non-zero**. Neither is a bug in arithmetic; in each case the
thing being counted is never written.

## 1. `cancellation_fees`

**Correction to my earlier report.** I previously said the dashboard aggregates
`Trip.cancellation_fee`. It does not. The actual query is:

```python
cancellation_history = TransactionHistory.objects.filter(
    txn_type="payment",
    status__in=["cancelled", "canceled", "completed", "success"],
    method__icontains="cancel",
)
```

Every `TransactionHistory.method` ever written is one of `'wallet'`, `'online'`,
or `trip.payment_method` (`'cash'`, `'online'`, `'wallet'`, `'deferred'`).
**Nothing writes a method containing "cancel"**, so the queryset is empty by
construction — the same shape of defect as the payout reconciler's selection.

Consequences today:
- the `cancellation_fees` total on the payment dashboard is always `0`;
- the "Cancellation Fee" option in the transaction-type filter always yields an
  empty list;
- `Trip.cancellation_fee` exists as a column and is **never written by any code**,
  so even a corrected query would find nothing.

ADR-0007 states that `cancelled_by`, `cancellation_reason` and `cancellation_fee`
"are set" by the cancel paths. The first two are. The third is not. That ADR line
is wrong and should be corrected regardless of what is decided below.

**Recommendation: C — remove from the dashboard until implemented.**

Not B (label it "unavailable"), because this is a *revenue* line on a payments
dashboard: a persistent ₹0 next to real revenue numbers reads as "we charge no
cancellation fees", which is a business statement nobody has made, and a
greyed-out "not implemented" still occupies a revenue row. Not A (implement now),
because cancellation fees need a real policy — who cancelled, after how long, was
the driver en route, and the MVA-2020 constraints — and that is product work with
no design yet.

Remove the total, the filter option and the transaction-type entry. Keep the
`Trip.cancellation_fee` column: it costs nothing and is where the value will go.
The removal is presentation-only and touches no money path.

## 2. `total_redemptions`

```python
"total_redemptions": PromoRedemption.objects.count(),
```

The count is correct. `PromoRedemption` rows are only created by
`promos.redeem_promo`, which **has no callers** — so the count is always 0, and
will stay 0 until promos are wired end to end.

The promo admin page also shows `total_promos` and `active_promos`, which are
genuine: ops really can create codes, and those counts reflect reality. So this
page is not uniformly fictional — one number on it is.

**Recommendation: B — show it explicitly as not yet available, on this page only.**

Different from `cancellation_fees` for two reasons. First, it is not a revenue
figure on a payments dashboard; it is an operational counter on the promo admin
page, next to counts that *are* real. Second, and more important: ops can create
and advertise promo codes today, and a "0 redemptions" figure next to "3 active
promos" invites the conclusion that the campaign is unpopular, when the truth is
that redemption is not implemented. That is a worse failure than showing nothing.

Replace the number with an explicit marker — "Redemption not yet enabled" — until
[promo-implementation-design](promo-implementation-design.md) step 5 lands, at
which point it becomes a real metric with no further change.

**Also worth doing at the same time, and arguably more urgent than the label:**
ops can currently create and advertise a promo code that no rider can use. Either
disable code creation in the admin until redemption is wired, or put the same
warning on the creation form.

## 3. Related, found while looking

`platform_fees` on the same dashboard carries the comment *"No authoritative
platform-fee ledger currently exists."* It is computed, not structurally zero, so
it is out of scope here — but it is the third figure on that page whose
provenance is weaker than its presentation suggests, and
[settlement-snapshot-design](settlement-snapshot-design.md) is what fixes it
properly.

## Summary

| Metric | Verdict | Action |
|---|---|---|
| `cancellation_fees` total | **C — remove** | Drop the total, the filter option and the type entry. Keep the column. |
| "Cancellation Fee" transaction type | **C — remove** | Always empty. |
| `Trip.cancellation_fee` column | keep | Unused but harmless; it is the destination when policy exists. |
| `total_redemptions` | **B — explicit marker** | "Redemption not yet enabled" until promos are wired. |
| promo code creation in admin | flag | Codes can be created and advertised but never redeemed. |

None of this invents a value, and none of it touches a money path. All of it is
presentation.
