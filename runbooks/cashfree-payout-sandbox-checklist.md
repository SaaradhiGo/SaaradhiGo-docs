# Cashfree payout status — sandbox verification checklist

- **Status:** checklist for a human. **Nothing invented.** Every item below is a
  question, not an assumption.
- **Date:** 2026-09-23
- **Why it exists:** `fix/payout-reconciliation-selection` fixes the selection and
  window defects, which are provable from our own state machine. It deliberately
  does **not** poll Cashfree, because the contract is unverified. This is the list
  of what must be confirmed before that boundary opens.
- **Related:** [payout-reconciliation-contract](payout-reconciliation-contract.md)

## What our code already proves (do not re-verify)

Read out of `payment_gateways/cashfree_gateway.py`, so these are facts about **our
integration**, not claims about Cashfree's current API:

| Fact | Where |
|---|---|
| Payouts use a separate base URL from payments | `CASHFREE_PAYOUT_BASE_URL`, default `https://payout-api.cashfree.com` |
| Payouts use separate credentials | `CASHFREE_PAYOUT_APP_ID`, `CASHFREE_PAYOUT_SECRET_KEY` |
| Auth is a bearer token from an authorize call, cached ~25 minutes | `POST /payout/v1/authorize` |
| Transfers are created on V1.2 with an in-line beneficiary | `POST /payout/v1.2/requestTransfer`, `beneId: ""` |
| The identifier we can look a transfer up by is **ours** | `transferId = reference_id = withdrawal_<id>_driver_<driver_id>`, stored as `WithdrawalRequest.payout_reference_id` |
| Cashfree also returns its own reference | read as `data.referenceId` / `data.reference_id`, but **not persisted** |
| Response envelopes already vary by version | the adapter reads `status` or `data.status` defensively |

Two things in that list are worth flagging before anything else:

1. **We do not store Cashfree's own reference id.** Only ours. If the status
   endpoint keys on Cashfree's reference rather than our `transferId`, we currently
   cannot call it for historical payouts at all — that is a data gap, not just an
   API question. It is item 2 below.
2. **The mixed `v1` / `v1.2` paths and the "Phase-0 shortcut" note** in
   `create_upi_payout` suggest this account may need to move to the V2 payouts API,
   where the endpoint, the auth scheme and the status vocabulary all differ. If so,
   items 1–9 below must be answered **for the version we will actually use**, not
   for V1.

## The checklist

Each item: what to confirm, and why the code cannot proceed without it.

### 1. Status endpoint

- [ ] Exact method and path for querying one transfer's status, for the payout API
      version this account is on.
- [ ] Whether it is the same base URL as `requestTransfer`.

*Why:* the reconciler needs one call. Guessing the path yields 404s that look
identical to "transfer not found", which would be read as a missing payout.

### 2. Transfer identifier

- [ ] Which identifier the status endpoint accepts: **our** `transferId`, or
      **Cashfree's** `referenceId`, or either.
- [ ] Whether a `transferId` that was never accepted returns "not found" or an
      error.

*Why:* we persist only our `transferId`. If the endpoint requires Cashfree's
reference, we must start persisting it on dispatch **before** reconciliation can
work for any payout dispatched from now on — and historical payouts may be
unreconcilable. This is the highest-value item on the list.

### 3. Request parameters

- [ ] Query string or JSON body; exact parameter names.
- [ ] Any required headers beyond authorization.

### 4. Authentication

- [ ] Whether the status endpoint accepts the same bearer token as
      `requestTransfer`, or needs its own scope.
- [ ] Token lifetime, so the adapter's ~25-minute cache is still correct.
- [ ] For V2, whether it is `x-client-id` / `x-client-secret` plus `x-api-version`
      rather than a bearer token.

### 5. Response envelope

- [ ] The exact success shape, captured **verbatim** from sandbox.
- [ ] Where the status lives: top level, or nested under `data`.
- [ ] Where a failure reason lives, and its field name.
- [ ] Whether amounts/UTR/settlement time are returned (useful for the ledger, not
      required).

*Why:* the adapter must read one known shape, not guess across three.

### 6. Provider status vocabulary

- [ ] The **complete** list of statuses a transfer can be in.

*Why:* the reconciler's current lists — `SUCCESS | COMPLETED | PROCESSED` and
`FAILED | REVERSED | CANCELLED | REJECTED` — appear to have been written from the
**webhook's** event names, not from a status endpoint. They may be wrong, and a
status we do not recognise must map to *skip*, never to failed.

### 7. Success mapping

- [ ] Which statuses mean the money has irrevocably reached the driver.
- [ ] Whether any "success" status can still reverse later.

*Why:* if a success can reverse, marking the withdrawal `completed` is premature
and the webhook must remain the authority.

### 8. Failure mapping

- [ ] Which statuses are terminal failures.
- [ ] Whether a failure means the funds are already back with us, or a separate
      reversal follows.

*Why:* this decides the refund policy question. It is why the failure branch ships
disabled.

### 9. Pending mapping

- [ ] Which statuses mean "still in flight".
- [ ] Typical and maximum settlement time for UPI payouts.

*Why:* it sets the grace window. Ours is 5 minutes, which was chosen without
evidence.

### 10. Idempotency

- [ ] Whether re-sending `requestTransfer` with the same `transferId` is safe, and
      what it returns.
- [ ] Whether the status endpoint is rate limited, and at what rate.

*Why:* `execute_upi_payout` retries up to 3 times with a stable `transferId`
specifically so a retry cannot create a second payout. **That assumption has never
been verified with the provider.** If Cashfree treats a repeated `transferId` as a
new transfer, we have a double-payout risk in code that is live today — which is
more urgent than reconciliation.

## What I need from you

Only two things, and only one of them is blocking:

1. **Sandbox payout credentials** (`CASHFREE_PAYOUT_APP_ID`,
   `CASHFREE_PAYOUT_SECRET_KEY` for the sandbox base URL), or someone with
   dashboard access to run one test transfer and paste the responses.
2. **Which payout API version this account is onboarded for** (V1 vs V2). The
   dashboard shows it; our code straddles both.

With those, the verification is one transfer plus one status call, and the answers
to items 1–10 get pasted into this file verbatim. Until then the reconciler stays
at its closed boundary and reports how many payouts are waiting.

**Item 10 is worth answering even if reconciliation is deprioritised** — it is
about a retry path that is already in production.

## Definition of done for this checklist

- [ ] Every item above answered, with the raw sandbox request/response recorded in
      this file.
- [ ] `CashfreeGateway.get_payout_status` implemented against the recorded contract,
      returning `{'status', 'failure_reason', 'raw'}` and `None` on any transport
      failure.
- [ ] Status-mapping tests written from the recorded vocabulary, with unknown
      statuses asserted to map to *skip*.
- [ ] The refund-policy decision made, and only then
      `WITHDRAWAL_RECON_FAILURE_HANDLING_ENABLED` flipped.
