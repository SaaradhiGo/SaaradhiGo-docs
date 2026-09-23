# Payout reconciliation — why it no-ops, and the smallest fix

- **Status:** investigation. **Nothing implemented.** Payout semantics unchanged.
- **Date:** 2026-09-23
- **Subject:** `payments.reconcile_stuck_withdrawals`, scheduled every 10 minutes.

The earlier report said this task no-ops because `CashfreeGateway.get_payout_status`
does not exist. That is true, and it is the *least* of three defects. Even with
that method implemented, the task would still reconcile nothing.

## The trace, end to end

```
driver requests       driver.views.driver_withdrawal_request
  -> wallet debited   at REQUEST time (not at payout)
  -> maker/checker    admin_dashboard / driver.admin_views -> status='approved'
  -> transfer         driver.services -> CashfreeGateway.create_upi_payout
                      POST /payout/v1.2/requestTransfer, in-line beneficiary
  -> persisted        WithdrawalRequest.payout_reference_id = our transferId
                     status='processed', payout_status=<gateway status>
  -> webhook          payments.views cashfree payout webhook
                     TRANSFER_SUCCESS -> 'completed'
                     TRANSFER_FAILED/REVERSED -> 'failed' + wallet refund
  -> reconciliation   payments.tasks.reconcile_stuck_withdrawals   <-- broken
  -> terminal         'completed' | 'failed'
```

## Defect 1 — the method does not exist (the reported one)

```python
status_fn = getattr(gateway, 'get_payout_status', None)
if not callable(status_fn):
    logger.warning("... gateway has no get_payout_status; skipping batch.")
    return {'ok': False, 'reason': 'no get_payout_status'}
```

`CashfreeGateway` implements `create_order`, `verify_payment_signature`,
`verify_webhook_signature`, `create_refund`, `get_order_status` and
`create_upi_payout`. There is no payout-status method. `BasePaymentGateway` has
the signature commented out at line 145. So every run returns early. The fail-soft
is deliberate and correctly logged — the task announces its own impotence — but
the warning has presumably been firing every 10 minutes since it shipped.

## Defect 2 — the status filter can never match

```python
WithdrawalRequest.objects.filter(status__in=['processing', 'approved'], ...)
    .exclude(Q(payout_reference_id__isnull=True) | Q(payout_reference_id=''))
```

* **`'processing'` is not a status this system has.** `WithdrawalRequest.STATUS_CHOICES`
  is `pending, approved, rejected, processed, completed, failed`. No code writes
  `'processing'`. It is a typo for `'processed'`.
* **`'approved'` cannot co-exist with a payout reference.** `payout_reference_id`
  is written in the same block that sets `status='processed'`. A row in `approved`
  has not been dispatched, so it has no reference and the `.exclude()` drops it.

So the two conditions are mutually exclusive: the status set contains only
pre-dispatch states, and the exclusion requires a post-dispatch artefact. The
queryset is empty by construction. **The state that actually gets stuck —
`processed` with a reference and no webhook — is the one state not selected.**

This is the more serious defect, because it would survive implementing
`get_payout_status` and the task would keep reporting success while reconciling
nothing.

## Defect 3 — the window is keyed on the wrong timestamp

`requested_at__lt grace_cutoff` (5 min) and `requested_at__gt lookback_cutoff`
(48 h) both filter on **`requested_at`**, but the thing being reconciled is the
*transfer*, which happens at `processed_at` — after maker/checker approval, which
is a human step. A withdrawal requested on Friday and approved on Monday is
already outside the 48-hour lookback the moment it is dispatched, so it can never
be reconciled even once defects 1 and 2 are fixed.

The window must be keyed on `processed_at`.

## Defect 4 — reconciliation and the webhook disagree about money

The webhook's failure path **refunds the driver's wallet**
(`purpose='refund_failed_withdrawal'`, keyed for idempotency). The reconciler's
failure path sets `status`, `payout_status`, `failure_reason`, `failure_count`,
`last_failure_at` — and **no refund**.

The task's docstring says "Driver wallet debit happens at request time, NOT here —
the only thing this task does is move 'processing' → 'completed' or 'failed'". That
is an accurate description of the code and an incorrect policy: a driver whose
payout failed and was reconciled rather than webhooked would be left debited with
no refund and a `failed` status. Because the task currently no-ops, this has never
happened. It must not become possible as a side effect of fixing defect 1.

## What the Cashfree adapter actually proves about the provider

Stated from the adapter's own code only. **Not asserted from memory of Cashfree's
API**, because inventing provider behaviour here would be a way to lose real money.

| Fact | Evidence in `cashfree_gateway.py` |
|---|---|
| Payouts use a **separate** base URL and credentials from payments | `CASHFREE_PAYOUT_BASE_URL` default `https://payout-api.cashfree.com`; `CASHFREE_PAYOUT_APP_ID` / `CASHFREE_PAYOUT_SECRET_KEY` |
| Auth is a bearer token obtained from an authorize call, cached ~25 min | `POST /payout/v1/authorize`, `_payout_token_expires_at = now + 25*60` |
| Transfers are created on the V1.2 endpoint, in-line beneficiary mode | `POST /payout/v1.2/requestTransfer`, `beneId: ""` |
| The identifier we can look a transfer up by is **ours** | `transferId = reference_id = f"withdrawal_{id}_driver_{driver_id}"`, returned as `payout_id` and stored in `payout_reference_id` |
| Response envelopes vary by version | the adapter already reads `data.get('status') or data['data']['status']`, and `referenceId or reference_id` |

**Unproven and required before implementation:** the exact status-query endpoint on
this API version, its query parameter (our `transferId` vs Cashfree's
`referenceId`), its response envelope, and its status vocabulary. The reconciler
currently assumes `SUCCESS | COMPLETED | PROCESSED` and `FAILED | REVERSED |
CANCELLED | REJECTED`, which appear to have been written from the webhook's
vocabulary rather than from a status endpoint.

This must be confirmed against Cashfree's own documentation for the version in use
and then **proven in sandbox** — one real transfer, one real status query, with the
response recorded verbatim in this document. The note in `create_upi_payout` that
V1 in-line beneficiaries were a Phase-0 shortcut, plus the mixed `v1`/`v1.2` paths,
makes it likely the account should be moving to the V2 payouts API, in which case
the endpoint, the auth scheme and the vocabulary all differ. **Do not implement
against an assumed contract.**

## The smallest fix, in order

### Step 0 — prove the contract (no code)

Obtain the payout status endpoint for the API version this account uses; call it
in sandbox for a known `transferId`; paste the response here. Until this exists,
steps 1–3 cannot be written correctly.

### Step 1 — fix the queryset and the window (no provider dependency)

Independently correct, independently testable, and it does nothing dangerous while
`get_payout_status` is still absent:

```python
status__in=['processed']            # the state that actually gets stuck
processed_at__lt=grace_cutoff       # the transfer's clock, not the request's
processed_at__gt=lookback_cutoff
```

Keep the `payout_reference_id` exclusion. Tests: a `processed` row with a
reference is selected; an `approved` row is not; a row dispatched 72 h ago is not;
a row dispatched 2 min ago is not.

### Step 2 — implement `get_payout_status` on `CashfreeGateway`

Against the contract from step 0, returning exactly what the reconciler consumes:

```python
{'status': str, 'failure_reason': str | None, 'raw': dict}
```

Shaped like the existing `get_order_status`: same token handling, same defensive
envelope reading, `None` on network failure or non-2xx so the reconciler skips
rather than guessing. Add the abstract signature to `BasePaymentGateway` (it is
already there, commented out) so a future gateway cannot silently omit it.

Tests mock the HTTP layer and assert: a success envelope maps to a terminal
success; a failure envelope carries the reason through; an unknown status maps to
*skip*, never to failed; a 500 returns `None`; a timeout returns `None`.

### Step 3 — decide the failure policy before enabling the failure branch

**DECISION NEEDED.** Either:

* **(a) Reconciliation refunds, like the webhook does.** Consistent outcomes
  regardless of which path observed the failure. Requires reusing the webhook's
  refund helper with the same idempotency key, so a webhook arriving later cannot
  double-refund. This is the recommendation.
* **(b) Reconciliation only marks, and refunds are a separate audited sweep.**
  Keeps the reconciler read-mostly, at the cost of a driver sitting debited until
  the second job runs.

Until this is decided, ship step 2 with the **success branch only** and leave
failures as `skipped` with a log line. A payout that really failed is then visible
to ops without the system moving money down an untested path.

### Step 4 — observability

`settled`, `failed`, `skipped` are returned but only logged as an f-string. Emit
them as structured fields (the worker logger now redacts PII), and alert on
`skipped > 0` sustained — that is the signal that the provider contract has
drifted again.

## What must not change

* The wallet is debited at **request** time. Reconciliation must never debit.
* Idempotency keys on the refund path are what make webhook-plus-reconciler safe.
  Any refund from reconciliation must reuse the webhook's key, not invent one.
* `payout_reference_id` remains our `transferId`. The webhook resolves by it with
  two fallbacks; changing it would break webhook matching for in-flight payouts.
