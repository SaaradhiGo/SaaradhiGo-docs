# VahanGo Credits Policy

* **Effective date:** {{LAUNCH_DATE}}
* **Version:** {{POLICY_VERSION}}
* **Owner:** {{COMPANY_LEGAL_NAME}}
* **Status:** DRAFT — REVIEW BY INDIAN-QUALIFIED FINTECH COUNSEL
  BEFORE PUBLICATION

> This document defines what "VahanGo Credits" are, how they are
> issued, how they can be used, and -- critically -- what they are
> *not*. The framing matters: VahanGo Credits are a **closed-loop
> rider-loyalty store of value, not a Prepaid Payment Instrument
> (PPI) under the RBI Master Direction on Prepaid Payment
> Instruments, 2021**.

---

## 1. What VahanGo Credits are

VahanGo Credits are a **non-transferable, non-cashable, loyalty-style
credit balance** maintained by {{COMPANY_LEGAL_NAME}} ("SaaradhiGo")
on behalf of a registered VahanGo rider. Each rider has at most one
credit balance; balances are denominated in Indian Rupees but are
**not redeemable for cash** under any circumstance.

VahanGo Credits can be applied **only** to:

* Fares for rides booked through the VahanGo rider application.
* Cancellation charges or platform fees levied by VahanGo.

VahanGo Credits **cannot** be:

* Withdrawn to any bank account, UPI handle, card, or other payment
  instrument.
* Transferred between rider accounts.
* Combined or aggregated with credits held by another rider.
* Used to purchase goods or services from any third party,
  including any merchant outside the VahanGo platform.
* Exchanged, re-sold, or assigned to any other person or entity.

## 2. How credits are earned

VahanGo Credits are issued by SaaradhiGo to a rider account through
**three channels only**:

| Channel | When | How recorded |
|---|---|---|
| **Refund credit** | A rider cancels a ride that had a completed online payment, and at the time of refund chooses "instant credit" instead of waiting 5-7 business days for a card / UPI refund. | `purpose='refund'`, `reference_id='TRIP_<id>'` |
| **Promo credit** | Marketing / referral / first-ride / cashback / loyalty campaigns run by SaaradhiGo. | `purpose='promo:<campaign>'`, `reference_id=<campaign>` |
| **Support credit** | Customer service issues a goodwill credit in response to a complaint, service incident, or compensable disruption. | `purpose='support:<ticket_ref>'`, `reference_id=<ticket_ref>` |

**Credits cannot be added to a rider account by the rider themselves.**
There is no top-up flow, no card-load, no UPI-load, no
add-money flow. The rider does not transfer external money to
SaaradhiGo for the purpose of holding credit.

## 3. Balance cap and expiry

| Parameter | Value (Phase-0) |
|---|---|
| Maximum cumulative credit balance per rider | ₹2,000 |
| Credit expiry (from date of issue) | 365 days |
| Notification before expiry | 30 days in advance |
| Behaviour at expiry | Credits expire and are written off; no cash equivalent paid |

Credit issuances that would push the cumulative balance over the
cap are refused. For refunds, the system falls back automatically
to a card / UPI refund through the payment gateway so the rider is
never left without recourse.

## 4. Not money; no interest

VahanGo Credits are a contractual loyalty arrangement between the
rider and SaaradhiGo. They are not deposits, not negotiable
instruments, not currency, and not money in the legal sense.
SaaradhiGo holds no escrow account, makes no representations as to
the safekeeping of credit balances, and pays no interest on credits
held in a rider account.

## 5. Order of application

When a rider pays for a trip, payment sources are applied in the
following order unless the rider explicitly selects otherwise:

1. Promo credits (oldest first by issue date).
2. Refund credits (oldest first).
3. Support credits (oldest first).
4. The rider's selected payment method (UPI / card / cash) for any
   remaining balance.

A trip may be paid for partially with credits and partially with
another payment method.

## 6. Refunds and reversals

If a trip paid for with credits is itself refunded, the credits are
returned to the rider account in the order they were consumed,
**with the original expiry dates preserved** (so reversing a
consumption of a soon-to-expire credit does not artificially extend
its life). The rider is not entitled to a cash refund of credit
consumed and later reversed.

## 7. Suspension, forfeiture, and clawback

SaaradhiGo may suspend, forfeit, or claw back VahanGo Credits at
any time if it has reasonable grounds to believe that:

* The credits were issued in error or through a system fault.
* The credits were issued in connection with promotional abuse
  (e.g. multiple-account fraud, referral self-dealing).
* The rider account is associated with fraudulent activity, an
  active SOS investigation, or any breach of the Terms of Service.
* SaaradhiGo terminates or suspends the rider account for any
  reason permitted by the Terms of Service.

Forfeited credits are written off and are not paid out in cash.

## 8. Closure of the rider account

If a rider closes their VahanGo account, any unused credit balance
is **forfeited** and not paid out in cash. The rider is encouraged
to use any outstanding credit balance before closing the account.
This is the explicit, knowing trade-off the rider accepts in
exchange for the convenience of a closed-loop credit store.

## 9. Changes to this policy

SaaradhiGo may change this policy from time to time. Changes that
materially adversely affect the rider (lowering the cap, shortening
expiry, narrowing the channels through which credits may be used)
will be notified at least 30 days in advance through the rider
application. Changes that benefit the rider (raising the cap,
lengthening expiry, broadening usage) may take effect immediately.

## 10. Grievances

For any issue concerning VahanGo Credits -- a missing credit, an
incorrectly forfeited balance, a refund-to-credit that should have
been a refund-to-original -- contact:

* **Email:** {{SUPPORT_EMAIL}}
* **In-app:** Profile → Help & Support → Credits

Unresolved grievances escalate to the SaaradhiGo Grievance Officer
named in the [Terms of Service](terms-of-service.md). The IT Rules
2021 grievance timeline applies (acknowledgement within 24 hours,
resolution within 15 days).

## 11. Regulatory framing

VahanGo Credits are operated as a **closed-loop loyalty arrangement
under contract law**. They are *not* a Prepaid Payment Instrument
("PPI") under the *Master Direction on Prepaid Payment Instruments,
2021* (and as amended) issued by the Reserve Bank of India because:

* External top-ups are not permitted.
* Credits are non-transferable and non-cashable.
* Credits expire and have a low cumulative cap (₹2,000).
* Credits are usable only for fares and platform charges on the
  VahanGo platform itself.

If at any future date SaaradhiGo introduces external top-ups, or
makes credits cashable, that change will be implemented through a
licensed PPI issuer (either via a co-branded PPI partnership or
through SaaradhiGo's own PPI licence) and the rider will be
explicitly notified before any such change takes effect.

---

*This policy is subject to the
[Terms of Service](terms-of-service.md) and
[Privacy Policy](privacy-policy.md). In the event of conflict, the
Terms of Service prevail.*
