# Ride economics model

Written for: the owner deciding how SaaradhiGo charges drivers, and the finance or
ops person who will re-run these numbers when the assumptions change.

Every formula is shown so the model can be recalculated. **Published figures are
labelled as such; everything else is a provisional assumption, not a market fact.**
Two inputs are real: the QA fare of ₹135.68 and the platform's configured commission
of 20.00%, both read off the running system on 2026-10-19.

---

## 1. Regulatory constraints — read before choosing a model

Under the **Motor Vehicles Aggregator Guidelines 2020** (issued under the amended
s.93, MV Act 1988), as reported by contemporaneous coverage:

- The aggregator's share of each fare is **capped at 20%**; the driver receives **at
  least 80%**.
- Surge is capped at **1.5× base fare**; aggregators may also go 50% below base.
- Cancellation penalties are capped at **10% of fare, maximum ₹100**.
- Enforcement is delegated to **state governments**, so the operative rules in
  Telangana and Andhra Pradesh must be confirmed locally.

Two consequences that decide this document:

1. SaaradhiGo's configured commission is **20.00%** — exactly at the legal ceiling.
   There is no headroom above it.
2. **A fixed driver subscription has no equivalent safe harbour.** The guidelines cap a
   *share of fare*; they do not contemplate a flat periodic fee, so its treatment is
   unsettled and state-dependent. A subscription that works out above 20% of fare for a
   low-utilisation driver is the obvious challenge. **Legal review is required before
   Model B ships** — this is not an engineering decision.

`RateCard` already carries `surge_cap_multiplier` and `night_surge_multiplier`; confirm
the deployed values are ≤ 1.5 for Indian zones.

---

## 2. Assumptions

| Input | Symbol | Vijayawada | Hyderabad | Source |
|---|---|---|---|---|
| Average completed fare | `F` | ₹150 | ₹180 | *assumed*; only real datapoint is ₹135.68 for 2.4 km |
| Trips per driver per day | `T` | 12 | 15 | *assumed* |
| Working days per week | `D` | 6 | 6 | *assumed* |
| Booking attempts per completed ride | `A` | 1.4 | 1.4 | *assumed* |
| Cash share of rides | — | 80% | 80% | *assumed*; QA is 100% cash |
| USD→INR | — | ₹88 | ₹88 | *assumed* |
| Monthly city fixed cost | `FC` | ₹250,000 | ₹400,000 | *assumed* — ops, support salaries, cloud baseline |

### Variable cost per completed ride

| Component | Formula | Vijayawada (₹) | Note |
|---|---|---|---|
| Maps / external APIs | from the architecture doc | **0.81** | **published** SKU prices, modelled volumes |
| Payment processing | `0.2 × 2% × F` (20% online at ~2% MDR) | 0.60 | cash is free |
| SMS + push | 2 OTP × ₹0.15 | 0.30 | push ≈ 0 |
| Cloud infrastructure | `FC_infra / rides` | 0.20 | at ~10k rides/day |
| Customer support | `5% × ₹25` | 1.25 | *assumed* contact rate |
| Incentives + refunds | `2% × F` | 3.00 | *assumed* |
| **Total `VC`** | | **₹6.16** | |

**Maps is 13% of variable cost.** Incentives and support together are 69%. Tuning Maps
spend is worth doing and is not where the money is.

---

## 3. Model A — commission

```
Platform revenue per ride   R_A = c × F
Contribution per ride       CM_A = R_A − VC
Break-even rides/month      BE_A = FC / CM_A
Driver net per day          DN_A = (1 − c) × F × T
```

Vijayawada, `F`=₹150, `VC`=₹6.16, `FC`=₹250,000:

| Commission `c` | `R_A` | `CM_A` | Break-even rides/mo | Break-even rides/day | Driver net/day |
|---|---|---|---|---|---|
| 5% | ₹7.50 | ₹1.34 | 186,567 | 6,219 | ₹1,710 |
| **10%** | **₹15.00** | **₹8.84** | **28,281** | **943** | **₹1,620** |
| 15% | ₹22.50 | ₹16.34 | 15,300 | 510 | ₹1,530 |
| 20% (cap, current) | ₹30.00 | ₹23.84 | 10,487 | 350 | ₹1,440 |

Hyderabad, `F`=₹180, `T`=15, `FC`=₹400,000, `VC`≈₹6.76:

| `c` | `CM_A` | Break-even rides/day | Driver net/day |
|---|---|---|---|
| 10% | ₹11.24 | 1,186 | ₹2,430 |
| 20% | ₹29.24 | 456 | ₹2,160 |

---

## 4. Model B — fixed weekly driver subscription

```
Rides per driver per week   T_w = T × D
Platform revenue per ride   R_B = S / T_w
Contribution per ride       CM_B = R_B − VC
Driver net per day          DN_B = F × T − S/D
Effective commission        c_eff = S / (T_w × F)
```

Vijayawada, `T_w` = 12 × 6 = **72 rides/week**:

| Weekly fee `S` | `R_B`/ride | `CM_B`/ride | `c_eff` | Break-even rides/day | Driver net/day |
|---|---|---|---|---|---|
| ₹300 | ₹4.17 | −₹1.99 | 2.8% | **never** | ₹1,750 |
| ₹500 | ₹6.94 | ₹0.78 | 4.6% | 10,684 | ₹1,717 |
| ₹700 | ₹9.72 | ₹3.56 | 6.5% | 2,341 | ₹1,683 |
| ₹1,080 | ₹15.00 | ₹8.84 | 10.0% | 943 | ₹1,617 |

**The structural property:** `R_B` is fixed per driver while `CM_B` depends entirely on
how much that driver works.

- A driver doing 72 rides/week at ₹500 pays an effective **4.6%** — cheaper than any
  commission we could offer, and the platform earns ₹0.78/ride.
- A driver doing **24** rides/week at the same ₹500 pays an effective **13.9%**, and
  `CM_B` = ₹20.83 − ₹6.16 = ₹14.67/ride.

So subscription **transfers utilisation risk to the driver** and pays the platform most
for the drivers who work least. That is backwards from a marketplace's incentives, and
it is also where the regulatory exposure sits: the low-utilisation driver is the one
whose effective rate drifts toward and past the 20% cap.

---

## 5. Sensitivity

Model A at 10%, Vijayawada, break-even rides/day (base **943**):

| Variable | −20% | base | +20% |
|---|---|---|---|
| Average fare `F` | 1,398 | 943 | 712 |
| Incentives + refunds | 863 | 943 | 1,040 |
| Support cost | 910 | 943 | 978 |
| **Maps cost** | **938** | **943** | **949** |
| Fixed cost `FC` | 755 | 943 | 1,132 |

A ±20% swing in Maps spend moves break-even by **6 rides/day**. A ±20% swing in average
fare moves it by **~340**. Price realisation and fixed-cost discipline dominate; Maps
does not.

Utilisation sensitivity, Model B at ₹500/week:

| Rides/driver/week | `c_eff` | `CM_B`/ride |
|---|---|---|
| 24 | 13.9% | ₹14.67 |
| 48 | 6.9% | ₹4.26 |
| 72 | 4.6% | ₹0.78 |
| 96 | 3.5% | −₹0.95 |

Above ~90 rides/week the platform **loses money per ride** at ₹500. Model B needs a
utilisation ceiling or tiered pricing to be safe, which reintroduces the complexity it
was meant to avoid.

---

## 6. Recommendation

**Adopt Model A at 10% commission for the pilot.** Reasons, in order of weight:

1. **Legal certainty.** A fare share is exactly what MVAG 2020 contemplates and caps.
   10% sits at half the ceiling with room to move either way. A flat driver fee has no
   safe harbour and would need state-by-state clearance before a single rupee is charged.
2. **Revenue tracks reality.** Platform income scales with rides and with fare, so a
   slow week costs the platform proportionally rather than crushing drivers with a fee
   they owe regardless of earnings.
3. **Break-even is reachable.** 943 rides/day in Vijayawada against 2,341 for the
   cheapest driver-acceptable subscription.
4. **It is a real differentiator.** Incumbents charge ~20%, which drivers publicly
   object to. Half that is a recruiting argument that costs us ₹180/driver/day against
   the 20% case — buy supply with it rather than with cash incentives, which are already
   the single largest variable cost at ₹3.00/ride.

**Hybrid, only after legal review:** a 5% base commission with an optional weekly fee
that caps total platform take for high-utilisation drivers. It keeps the per-fare share
as the legal instrument while giving committed drivers a predictable ceiling. Do not
build it until a lawyer has signed off on the subscription leg.

**Do not stay at 20%.** It is the legal maximum, it matches the incumbents, and the
sensitivity table shows we do not need it: 10% breaks even inside the pilot's plausible
volume range.

---

## 7. Recalculating

```
VC   = maps + payment + sms + infra + support + incentives
CM_A = c × F − VC
BE_A = FC / CM_A                        (rides per month)
R_B  = S / (T × D)
CM_B = R_B − VC
c_eff= S / (T × D × F)
```

Change `F`, `T`, `D`, `c`, `S`, `FC` or any `VC` component in §2 and re-run. Maps cost
per ride by volume comes from `maps-pricing-architecture.md` §2; it is ₹0 below roughly
1,000 rides/day because of India's 70,000-event free tier per SKU.
