# TripSettlement — final schema proposal

- **Status:** proposal for review. **Not implemented, not merged.** No financial
  schema has been created.
- **Date:** 2026-09-23
- **Supersedes the schema section of:** [settlement-snapshot-design](settlement-snapshot-design.md)
  (that document holds the reasoning; this one holds the final shape)

## Position in the chain

```
Trip
  └─ TripSettlement          the ECONOMICS: why this much was taken
       └─ WalletTransaction  the MONEY MOVEMENT: the authoritative ledger
```

`WalletTransaction` remains the single authoritative financial history. Nothing
here writes a second balance-affecting record, and no amount in `TripSettlement`
is ever summed to produce a balance. It explains one ledger entry; it does not
duplicate it.

## Why not a branch yet

The `FarePricing` change was prepared as a branch because it extends an existing
table and is genuinely inert — adding nullable columns changes nothing. A new
`TripSettlement` table is different: an empty table whose write path is not
authorized invites someone to start writing to it out of band. So this stays a
proposal until the write path is approved, and then lands with it.

## The model

```python
class TripSettlement(models.Model):
    """Immutable record of the economics behind one settlement.

    Written in the SAME transaction as the WalletTransaction it explains, so the
    two can never disagree after a crash. Never updated: a correction is a new
    version row.
    """

    SOURCE_NATIVE = 'native'              # written at settlement time
    SOURCE_RECONSTRUCTED = 'reconstructed' # backfilled from the ledger + trip
    SOURCE_CHOICES = [
        (SOURCE_NATIVE, 'Written at settlement'),
        (SOURCE_RECONSTRUCTED, 'Reconstructed by backfill'),
    ]

    trip = models.ForeignKey(
        'ride.Trip', on_delete=models.PROTECT, related_name='settlements',
    )
    # Denormalised deliberately: a trip's driver assignment is mutable, a
    # settlement's is not.
    driver = models.ForeignKey(
        'driver.Driver', on_delete=models.PROTECT, related_name='settlements',
    )
    # The ledger row that actually moved the money. This FK is what makes this
    # record an explanation rather than a second ledger. Nullable only for
    # reconstructed rows whose ledger entry cannot be matched.
    wallet_transaction = models.ForeignKey(
        'rider.WalletTransaction', on_delete=models.PROTECT,
        null=True, blank=True, related_name='settlement',
    )

    # --- the economics ---------------------------------------------------
    gross_fare = models.DecimalField(max_digits=10, decimal_places=2)
    commission_percent = models.DecimalField(max_digits=5, decimal_places=2)
    commission_amount = models.DecimalField(max_digits=10, decimal_places=2)
    driver_net = models.DecimalField(max_digits=10, decimal_places=2)

    payment_method = models.CharField(max_length=20)   # cash | online | wallet

    # --- provenance ------------------------------------------------------
    version = models.PositiveIntegerField(default=1)
    source = models.CharField(max_length=16, choices=SOURCE_CHOICES,
                              default=SOURCE_NATIVE)
    settled_at = models.DateTimeField()
    created_at = models.DateTimeField(auto_now_add=True)

    class Meta:
        db_table = 'trip_settlement'
        constraints = [
            # One CURRENT settlement per trip. Corrections add a version, so the
            # uniqueness is on (trip, version) and the latest version wins.
            models.UniqueConstraint(
                fields=['trip', 'version'],
                name='tripsettlement_one_per_trip_version',
            ),
            # A settlement can only ever explain one ledger row, and a ledger row
            # can only ever be explained once.
            models.UniqueConstraint(
                fields=['wallet_transaction'],
                condition=models.Q(wallet_transaction__isnull=False),
                name='tripsettlement_one_per_ledger_row',
            ),
            models.CheckConstraint(
                condition=models.Q(commission_amount__gte=0),
                name='tripsettlement_commission_non_negative',
            ),
            models.CheckConstraint(
                condition=models.Q(commission_percent__gte=0)
                & models.Q(commission_percent__lte=100),
                name='tripsettlement_commission_percent_sane',
            ),
        ]
        indexes = [
            models.Index(fields=['driver', '-settled_at'],
                         name='tripsettle_driver_time_idx'),
            models.Index(fields=['-settled_at'], name='tripsettle_time_idx'),
        ]
```

Eleven fields plus provenance. Every one maps to something the current system
cannot answer.

## Field-by-field justification

| Field | Why it must be stored |
|---|---|
| `trip` | `WalletTransaction` has **no trip FK** — the only link today is the string `TRIP_<id>` in `reference_id`, which the earnings report has to parse. This is the real foreign key. |
| `driver` | A trip's `driver_id` can be reassigned; the settlement's cannot. |
| `wallet_transaction` | Makes this an explanation of a specific ledger movement, and lets the ledger be audited from either direction. |
| `gross_fare` | Not recorded anywhere today for cash trips: the ledger holds only the commission. |
| `commission_percent` | The rate **as applied**. This is the field that ends recomputation: no consumer ever needs `commission_percent_for_trip` for a historical trip again. |
| `commission_amount` | Stored rather than derived because rounding is `ROUND_HALF_UP` on a quantized Decimal and must not be re-derived differently later. |
| `driver_net` | `gross_fare − commission_amount`, stored so the identity is asserted at write time rather than assumed at read time. |
| `payment_method` | Decides the ledger's direction and who physically holds the fare. Without it, a cash commission debit and an online net credit are indistinguishable — the exact defect behind the cash-driver ₹0 bug. |
| `version` | Corrections are new rows. There is no update path. |
| `source` | `reconstructed` marks rows whose `commission_percent` was derived by backfill rather than recorded when charged. A financial table with a silent behaviour change at a cutover date is worse than one that says which rows were derived. |
| `settled_at` | When the economics were fixed. Distinct from `created_at`, which is when the row was written — they differ for backfilled rows. |

## Not included, and why

- **Tax.** GST is a rider-side tax on the fare and already lives on the versioned
  `Receipt`. Commission's own tax treatment is an accounting question nobody has
  stated a rule for; inventing a column would be guessing. One field, added later,
  from a stated rule.
- **`funding_model`.** Belongs on `PromoRedemption`
  ([promo-implementation-design](promo-implementation-design.md)); the promo FK on
  the fare snapshot is the join. Duplicating it here would give two answers.
- **A balance column.** That is `Wallet`. This table must never become a ledger.
- **`rider_payable`.** It is on the fare snapshot. Settlement is about the driver
  side, and under platform-funded promos the driver's economics are computed from
  `gross_fare` regardless of what the rider paid — so including `rider_payable`
  here would invite exactly the wrong join.

## Where it is written

Inside `credit_driver_wallet`'s existing `transaction.atomic()` block, **after**
the `WalletTransaction` insert:

```python
with transaction.atomic():
    wallet = get_wallet(..., lock=True)
    ...
    try:
        with transaction.atomic():
            txn = WalletTransaction.objects.create(
                ..., idempotency_key=f'TRIP_{trip.id}_EARNING')
    except IntegrityError:
        return                      # already settled — unchanged behaviour

    wallet.balance = new_balance
    wallet.save(update_fields=['balance'])

    TripSettlement.objects.create(            # <-- new, same transaction
        trip=trip, driver=trip.driver_id, wallet_transaction=txn,
        gross_fare=amount, commission_percent=commission_rate,
        commission_amount=commission, driver_net=amount - commission,
        payment_method=trip.payment_method or 'online',
        settled_at=timezone.now(),
    )
```

**Existing idempotency is preserved, not re-implemented.** The unique
`idempotency_key` on `WalletTransaction` already guards the whole block — the
function returns early on `IntegrityError` — so the settlement row is created
exactly once for the same reason the ledger row is. `credit_driver_wallet` has
seven callers and this property is what makes them all safe; the new insert sits
behind it rather than beside it.

`gross_fare` uses the same `amount` the commission was computed from, so the
recorded economics are internally consistent by construction, not by a later
join.

## Backfill

Reconstructable from the ledger plus the trip, exactly as the driver-earnings
report already does it:

- `gross_fare` = `trip.final_fare or trip.estimated_fare`
- cash (`purpose='trip_commission'`): `commission_amount` = the row amount;
  `driver_net` = gross − amount
- online (`purpose='trip_earnings'`): `driver_net` = the row amount;
  `commission_amount` = gross − amount
- `commission_percent` = derived, and therefore `source='reconstructed'`
- rows whose trip cannot be resolved are skipped and counted, not guessed

## What this fixes downstream

- **Admin revenue stops drifting.** `admin_dashboard` currently calls
  `commission_percent_for_trip` at *render* time, so editing a rate card today
  silently changes last month's reported platform revenue.
- **Driver earnings stop being inferred.** The read-only fix has to deduce
  commission from the ledger's direction; with this it reads three columns.
- **`platform_fees`** on the payment dashboard — currently commented "No
  authoritative platform-fee ledger currently exists" — gets one.

## Sequencing

1. Model + migration. Additive: one new table, no change to any existing one.
2. Write it inside `credit_driver_wallet`. No amount changes; one row is added
   alongside the two that already exist.
3. Backfill historical trips with `source='reconstructed'`.
4. Point the earnings report and admin revenue at it; delete the derivation code.
   **This is the step where dashboard numbers stop moving on their own** — not
   because money moved, but because it stops being recomputed.
5. Add `purpose` to `TransactionHistory` so the display log stops being ambiguous.
   Independent; do it whenever convenient.

Steps 1–3 are additive and safe before any fare policy is settled. Step 2 is the
only one that touches a money path, and it adds a row rather than changing an
amount.
