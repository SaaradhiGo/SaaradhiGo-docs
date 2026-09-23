# Driver earnings — root cause and fix

- **Status:** root cause confirmed; the read-only fix is implemented on
  `fix/driver-earnings-visibility`. The ledger changes described at the end are
  **not** implemented.
- **Date:** 2026-09-23
- **Reported as:** cash-only drivers see ₹0 earnings.

## The trace, end to end

```
trip completion            consumers._update_trip_status -> _create_payment_on_complete
  -> settlement            driver.utils.credit_driver_wallet
       -> wallet           rider.models.Wallet (scope=driver), balance updated
       -> ledger           rider.models.WalletTransaction  (the settlement row)
       -> history          payments.models.TransactionHistory (a display row)
  -> earnings API          driver.views.driver_earnings / driver_earnings_summary
  -> driver UI             SaaradhiGo-driver  features/earnings
```

## The defect

`credit_driver_wallet` writes **one** `WalletTransaction` per completed trip, and
its direction depends on who is holding the money:

| Payment method | Who holds the fare | Row written | Amount |
|---|---|---|---|
| online / wallet | the platform | **credit**, `purpose='trip_earnings'` | fare − commission (**net**) |
| cash | the driver | **debit**, `purpose='trip_commission'` | the commission only |

Both endpoints filtered `txn_type='credit'`:

```python
TransactionHistory.objects.filter(driver_id=driver, txn_type='credit')
```

A cash trip never produces a credit row, so a cash-only driver's earnings history
was **empty** and both `total_earned` and `total_trips` were 0 — while their
settlement balance went *negative* by the commission they owed. That is the
reported ₹0, and it is a read bug: settlement itself was correct.

### Two more defects in the same twelve lines

**Commission deducted twice for online drivers.** The credit row holds the *net*.
The summary summed those credits into `total_earned` and then subtracted 20% of
that total as commission. The driver app then computes `netEarned = total_earned −
total_commission` (its own model does this, see
`SaaradhiGo-driver/lib/features/earnings/domain/earnings.dart`), so an online
driver saw net-minus-commission-again. `total_earned` has to be the **gross fare**
for the app's subtraction to be right.

**Hardcoded rates.** `driver_earnings_summary` had `commission_percent = 20`, and
`driver_earnings` returned `'commission': 0.0` on every row with the comment
"Commission not tracked in TransactionHistory". It is tracked — in the row's own
amount.

## Every hardcoded commission percentage in the codebase

Searched for commission literals across `servers/` and `base/`:

| Location | Value | Status |
|---|---|---|
| `driver/views.py` `driver_earnings_summary` | `commission_percent = 20` | **fixed** — now derived from the ledger |
| `driver/views.py` `driver_earnings` | `'commission': 0.0` | **fixed** — now the real amount |
| `driver/utils.py` `credit_driver_wallet` | none — calls `commission_percent_for_trip` | correct |
| `admin_dashboard/views.py` executive revenue | none — calls `commission_percent_for_trip` | correct, but see below |
| `admin_dashboard/views.py` ride detail | none — calls `commission_percent_for_trip` | correct, but see below |
| `pricing/services.py` `commission_percent_for_trip` | final fallback `Decimal('18')` | correct: this **is** the canonical resolver |

The canonical source is `pricing.commission_percent_for_trip`, which resolves in
order: the trip's zone+vehicle `RateCard.commission_percent`, then the
`PLATFORM_COMMISSION_PERCENT` platform setting, then the Django setting, then 18%.
That is the right design and is already used by settlement and by the admin
reports. `test_executive_revenue.py` pins it with a sentinel 11% rate card, so
reintroducing any literal (15, 18 or 20) fails CI.

Also hardcoded nearby, not commission but worth listing: the withdrawal platform
fee (`fee_percent = 2.0`), the minimum withdrawal (₹500) and the payout retry
limit (3) are all literals in `driver/views.py` and `driver/services.py`. They
belong in `PlatformSettings` alongside the commission percentage.

## The fix that shipped

Read-only, on `fix/driver-earnings-visibility`:

* Filter on **`purpose`**, not on `txn_type`. A driver's wallet also holds
  withdrawals, payouts and withdrawal refunds; matching debits would have turned
  every withdrawal into a phantom cash ride. A test covers that.
* Report **gross, commission and net** for every trip, with one definition of net
  for cash and online, so a mixed-method driver can read one list.
* Derive commission **from the ledger row**, never by re-deriving from today's
  rate card — a rate card edited in March must not rewrite what a driver was told
  they earned in January.
* `commission_percent` becomes the effective rate actually charged.
* Trips whose fare cannot be resolved are counted as `unresolved_trips` rather
  than dropped.
* Logic lives in `servers/driver/earnings.py`; the views are thin adapters.

Every existing response key is preserved for the shipped driver app;
`total_net`, `today_net`, `cash_collected` and `unresolved_trips` are added.
13 tests, including one that snapshots the wallet balance, every ledger row and
the trips' fare columns across a full read and asserts nothing changed.

### Why it is safe to ship without review of settlement

It performs no writes. Settlement history, wallet balances and commission are
untouched, so if the presentation is still wrong it can be corrected again with
no data migration and no adjustment entries.

## What is still wrong, and is NOT fixed here

These are ledger changes. They alter or extend settlement records, so they are
listed rather than done.

1. **`WalletTransaction` has no trip FK.** The only link from a settlement row to
   its trip is the string `TRIP_<id>` in `reference_id`, which the fix parses. A
   real foreign key is the correct fix and needs a migration plus a backfill.
2. **Settlement stores neither the gross fare nor the commission rate.** What the
   platform took on a trip is recoverable only by re-deriving it. An immutable
   fare with a recomputable commission is not auditable — see
   [fare-finalisation-design](fare-finalisation-design.md) question 10.
3. **`final_fare` is never written**, so the "fare" every earnings figure rests on
   is the up-front estimate. Same document.
4. **Admin revenue re-derives commission at render time**, so editing a rate card
   silently changes last month's reported platform revenue. Stamping the
   commission (item 2) fixes this as a side effect.
5. **`TransactionHistory` is a display table doing double duty.** It carries
   trip transactions, driver credits, refunds and withdrawal rows in one shape,
   with no `purpose`. That is why the original query could not distinguish a cash
   commission debit from a refund reversal debit — both are `txn_type='debit'`
   with a trip. The settlement ledger and the display log should be separate
   concerns.

Recommended order: 1 and 2 together in one migration with a backfill, because
both are needed before fare finalisation and both touch the same rows.
