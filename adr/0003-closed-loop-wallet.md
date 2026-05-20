# ADR-0003: Closed-loop wallet for Phase-0 (no external top-ups)

* Status: Accepted
* Date: 2026-05-20
* Deciders: Architect, Founder
* Supersedes: none
* Superseded by: none

## Context

The SaaradhiGo backend already ships a working rider wallet that
accepts external top-ups via Cashfree (card / UPI), stores a balance,
and lets a rider spend that balance on trips. Under RBI's *Master
Direction on Prepaid Payment Instruments, 2021* (and the 2023
amendments), a wallet that does all three of those things is a
**Prepaid Payment Instrument (PPI)** -- specifically, a *semi-closed
PPI*. Issuing a PPI in our own name requires:

- An RBI authorisation under the PSS Act 2007.
- ₹5 crore minimum positive net worth (₹15 crore within 3 years).
- Quarterly net-worth + escrow account reporting.
- KYC-tier balance caps (₹10K without full KYC, ₹2 lakh with).
- Annual statutory audit + PPI System Audit.
- Customer Grievance Officer + Ombudsman Scheme compliance.
- Data localisation + breach reporting to CERT-In.

We do not have any of this in Phase-0. Operating the wallet as-is at
launch would expose the company to RBI penalty + mandatory
cease-and-desist orders the moment volumes draw regulatory
attention. The risk is asymmetric: small downside if we never
notice, catastrophic downside (and reputational damage at exactly
the wrong fund-raising moment) if we do.

The competitive landscape supports this read:

* **Uber India** removed Uber Cash for new users in 2018 and now
  relies on UPI / card / cash at the point of trip.
* **Ola** spun out *Ola Financial Services Pvt Ltd* as a separate
  legal entity holding the RBI PPI licence.
* **Rapido** runs a closed-loop credit balance only (refunds +
  cashbacks; no external top-ups).
* **Namma Yatri / Yatri Sathi** have no wallet at all.

## Decision

For Phase-0, SaaradhiGo runs the wallet as a **closed-loop credit
store**, not a PPI. Concretely:

1. **External top-ups are disabled by default.** A boolean setting
   `WALLET_TOPUPS_ENABLED` (defaults to `False`) gates the three
   server entry points that can credit a rider wallet from external
   money:
   - `POST /api/v1/rider/wallet/create-order/`
   - `POST /api/v1/rider/wallet/verify/`
   - the Cashfree-webhook wallet-credit branch in
     `/api/v1/payments/webhook/`
   All three return HTTP 503 with `code: FEATURE_DISABLED` while the
   setting is False. The wallet *models* and the Flutter UI are
   retained -- only the funding flow is removed.

2. **Credits enter the wallet through three named channels only:**
   - `issue_refund_credit(...)` -- rider opts in to "instant credit"
     instead of waiting 5-7 business days for a card refund on a
     cancelled trip.
   - `issue_promo_credit(...)` -- marketing / referral / cashback
     credits issued by the platform.
   - `issue_support_credit(...)` -- customer-service goodwill
     credits issued by support staff.
   All three are idempotent (keyed on a caller-supplied
   `idempotency_key`) and cap-enforced (refuse a credit that would
   push the balance over `RIDER_CREDIT_BALANCE_CAP`, default ₹2000).

3. **Refunds gain a `mode` parameter** -- `mode='original'` (default;
   gateway refund to card / UPI, 5-7 days) or `mode='credit'`
   (instant credit to VahanGo Credits, subject to cap). When the cap
   would be exceeded, the refund automatically falls back to the
   original-method path so the rider is never left without recourse.

4. **The rider-facing brand changes from "Wallet" to "VahanGo
   Credits"** in the Flutter UI. The "Add Money" button is hidden
   behind a server-controlled feature flag read at app start from
   `GET /api/v1/config/`. The display name, balance cap, and
   supported refund modes all come from the same config blob so we
   can flip behaviour without shipping a new app build.

5. **A Credits Policy** lives at
   [legal/credits-policy.md](../legal/credits-policy.md). The policy
   states: credits are non-transferable, non-cashable, expire 12
   months after issue, capped at ₹2000 cumulative balance, and are
   *not money* in the legal sense. This is the defensive framing if
   RBI ever inquires about the surface.

6. **The driver wallet (earnings + withdrawals via Cashfree Payouts)
   is unaffected.** Driver settlement is an *intermediated payout*,
   not a PPI -- different RBI bucket (Payments Aggregator / Payouts
   guidelines), which we already comply with through Cashfree.

## Consequences

### Positive

* **We can launch.** The wallet feature stays in the app for refund
  + promo + support flows without exposing the company to RBI PPI
  enforcement risk.
* **One server flag re-enables top-ups** the day the PPI partnership
  or licence is in place. No code change, no new app build, no
  migration.
* **Refund UX improves.** Riders who pick `mode='credit'` get their
  money back in seconds instead of 5-7 days, with a clear opt-in.
* **Audit trail is centralised.** Every credit flows through
  `servers/rider/credits.py` with `idempotency_key` + `purpose` +
  `reference_id`, so a future regulator (or our own auditor) can
  reconstruct exactly why every rupee of credit was issued.

### Negative / costs

* **No new-user funding-of-wallet conversion** path. Riders who
  *want* to load money cannot. Mitigated by the fact that UPI /
  card / cash at trip-end remains fully supported -- the wallet was
  never a critical conversion lever; it was a feature parity item.
* **Some UI complexity** in the Flutter wallet screen (a feature
  flag branch for the Add Money button + heading + filter chip).
  Mitigated: one Consumer<RemoteConfigProvider> wrap in
  `wallet_screen.dart`.
* **Risk that "VahanGo Credits" still gets characterised as a
  PPI** by an aggressive interpreter of the rules. RBI's stance has
  tightened over the years on closed-loop wallets that look too
  much like money. Mitigated by the Credits Policy framing
  (non-cashable, non-transferable, expiring, capped, *not money*),
  but **a 30-minute consult with a fintech-qualified lawyer is
  REQUIRED before this PR's behavior is publicly launched.**

### Neutral

* **The wallet schema is unchanged.** Wallet + WalletTransaction
  models, the `wallet_payment` debit flow, and the existing rider
  wallet UI all stay. We're disabling the funding path, not deleting
  the surface.

## Alternatives considered

1. **Apply for our own RBI PPI licence now.** ₹5 crore minimum
   capital + 6-12 months of regulatory process before launch is
   feasible. Rejected: blocks Phase-0 launch entirely; revisit at
   Phase-2 once trip volume justifies it.

2. **Co-branded PPI with a licensed issuer (Pine Labs, Zaakpay,
   Mobikwik, etc.).** Their licence, our UX, integration setup +
   partner fees, 3-6 months to negotiate + integrate. Rejected for
   Phase-0 (timeline), accepted as the Phase-1 / Phase-2 target
   once we have the trip volume to negotiate decent partner
   economics.

3. **Delete the wallet entirely** for Phase-0 and re-introduce it
   later. Wastes the working refund + promo infrastructure and
   leaves no place for refund-to-credit, which is a *desirable*
   product feature even under the closed-loop posture. Rejected.

4. **Keep top-ups live and hope nobody notices.** Rejected. The
   reputational + regulatory downside dominates the (small) upside.

## Implementation

### Backend (PR SaaradhiGo-backend#TBD, branch `feat/closed-loop-wallet`)

* `base/settings.py` -- adds `WALLET_TOPUPS_ENABLED` (default
  `False`), `RIDER_CREDIT_BALANCE_CAP` (₹2000),
  `RIDER_CREDIT_EXPIRY_DAYS` (365).
* `servers/rider/credits.py` -- new module with
  `issue_refund_credit`, `issue_promo_credit`, `issue_support_credit`,
  all idempotent + cap-enforced + atomic.
* `servers/rider/views.py` -- gates `create_wallet_order` and
  `verify_wallet_payment` behind `WALLET_TOPUPS_ENABLED`.
* `servers/payments/views.py` -- gates the Cashfree-webhook wallet
  credit branch; adds `mode` parameter to `refund_payment` with
  `credit` / `original` semantics and automatic fallback when the
  cap would be exceeded.
* `base/config_view.py` -- new `GET /api/v1/config/` (unauthenticated)
  returning the closed-loop wallet flags for the mobile client.
* `tests/test_closed_loop_wallet.py` -- 11 new tests:
  config payload, top-up 503 gate, top-up gate re-enable,
  credit issuance, idempotency, cap enforcement, invalid-amount
  rejection, refund mode=credit, refund cap-fallback, invalid mode.
  Full suite: 92 passed, 9 skipped, 0 regressions.

### Mobile (PR SaaradhiGo-mobile#TBD, branch `feat/closed-loop-wallet`)

* `lib/providers/remote_config_provider.dart` -- new ChangeNotifier
  that fetches `/api/v1/config/` at app start. Conservative
  defaults (`walletTopupsEnabled = false`) until the response lands.
* `lib/main.dart` -- registers the provider, fires
  `..fetch()` immediately.
* `lib/screens/home/wallet_screen.dart` -- heading switches between
  "Wallet" and "VahanGo Credits" based on the flag; "Add Money"
  button hidden when top-ups are disabled; balance card label
  changes to "Credit Balance"; filter chip renamed Recharges →
  Credits; an explanatory line under the balance card explains how
  credits are earned.

### Docs (this repo, PR #TBD)

* `adr/0003-closed-loop-wallet.md` (this file).
* `legal/credits-policy.md` -- new user-facing policy document.
* `legal/terms-of-service.md` -- gains a reference to the Credits
  Policy in the Payments section.

## Follow-ups (out of scope for this ADR)

* **Credit expiry job.** A daily Celery task (TBD) that writes off
  credits older than `RIDER_CREDIT_EXPIRY_DAYS` and sends a
  notification 30 days before expiry.
* **Per-user credit ledger view in admin.** So support can answer
  "why does this rider have ₹X in credits?" without SQL.
* **ADR-0004 (Phase-1):** evaluate co-branded PPI providers when
  trip volume + funding allow.
* **ADR-0005 (Phase-2+):** apply for our own RBI PPI licence if the
  unit economics of a partner arrangement no longer make sense.

## Required follow-up before public launch

**A 30-minute consult with a fintech-qualified advocate** (e.g.
Khaitan, Trilegal, AZB, IndusLaw payments practice) to confirm that
the closed-loop framing in `legal/credits-policy.md` + the technical
implementation in this ADR are sufficient to keep SaaradhiGo outside
the RBI PPI perimeter. Approximate cost: ₹25K-50K. Without this
consult, do not turn `WALLET_TOPUPS_ENABLED` on or run an aggressive
promo campaign that would push large numbers of riders to
>₹1500-balance state.
