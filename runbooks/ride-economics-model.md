# Ride economics model

Written for: the owner deciding how SaaradhiGo charges drivers, and the finance or
ops person who will re-run these numbers when the assumptions change.

Every formula is shown so the model can be recalculated. **Published figures are
labelled as such; everything else is a provisional assumption, not a market fact.**
Two inputs are real: the QA fare of ₹135.68 and the platform's configured commission
of 20.00%, both read off the running system on 2026-10-19.

---

## 1. Regulatory constraints - read before choosing a model

**Correction to the previous revision of this document:** it cited the Motor Vehicles
Aggregator Guidelines **2020**. Those have been revised. The operative framework is the
**Motor Vehicles Aggregator Guidelines 2025**, issued by MoRTH on **01.07.2025**. The
figures below come from secondary reporting of that notification; **the official MoRTH
text must be confirmed before anything is priced on it.**

| Rule | MVAG 2020 | **MVAG 2025** |
|---|---|---|
| Owner-driver share of fare | >= 80% | **>= 80%** (aggregator <= 20%) |
| Aggregator-owned vehicle | not specified | **driver >= 60%** (aggregator <= 40%) |
| Surge / dynamic pricing cap | 1.5x base fare | **2.0x base fare** |
| Off-peak floor | 0.5x base | 0.5x base |
| Cancellation penalty | 10% of fare, max Rs.100 | 10% of fare, max Rs.100 |
| Base fare | state-notified | **state-notified** |
| Driver insurance | - | **Rs.5 lakh health + Rs.10 lakh term, per driver, mandatory** |

Four consequences that decide this document:

1. SaaradhiGo's configured commission is **20.00%** - verified on the running system -
   which is exactly the owner-driver ceiling. There is no headroom above it.
2. **Mandatory driver insurance is a new, unavoidable platform cost.** It is a per-driver
   fixed cost, so it hurts low-utilisation drivers' unit economics most, and it must be
   carried whether or not that driver completes a single ride.
3. The surge ceiling **rose to 2.0x**. SaaradhiGo's configured night surge is **1.25x**
   (verified: trip #92 priced at Rs.169.60 against a Rs.135.68 day fare on the same
   route), comfortably inside both the old and new caps. `RateCard.surge_cap_multiplier`
   should still be confirmed <= 2.0 per zone, and tighter where a state says so.
4. **A fixed driver subscription still has no safe harbour.** MVAG 2025 continues to
   regulate a *share of fare*; a flat periodic fee is not addressed, so its treatment is
   unsettled and state-dependent. Because the fee is fixed while earnings are not, the
   effective take rate rises exactly for the drivers who work least - which is where a
   challenge under the 80% floor would land. **Legal review is required before Model B
   ships, in Telangana and Andhra Pradesh separately**, since enforcement is delegated
   to states (Chandigarh, for example, drafted a tighter 1.5x surge cap).

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
| **Driver insurance (MVAG 2025)** | `₹4,500/yr ÷ 312 rides/mo` | **1.20** | **mandated**; premium *assumed* |
| **Total `VC`** | | **₹7.36** | |

**Maps is 11% of variable cost.** Incentives, support and the newly mandated driver
insurance together are 74%. Tuning Maps spend is worth doing and is not where the
money is.

The insurance line is new and it is structural: it is **per driver, not per ride**, so
a driver completing 6 rides a day costs twice as much per ride to insure as one
completing 12. Utilisation now drives compliance cost as well as revenue.

---

## 3. Model A — commission

```
Platform revenue per ride   R_A = c × F
Contribution per ride       CM_A = R_A − VC
Break-even rides/month      BE_A = FC / CM_A
Driver net per day          DN_A = (1 − c) × F × T
```

Vijayawada, `F`=₹150, `VC`=₹7.36, `FC`=₹250,000:

| Commission `c` | `R_A` | `CM_A` | Break-even rides/mo | Break-even rides/day | Driver net/day |
|---|---|---|---|---|---|
| 5% | ₹7.50 | ₹0.14 | 1,785,714 | 59,524 | ₹1,710 |
| **10%** | **₹15.00** | **₹7.64** | **32,723** | **1,091** | **₹1,620** |
| 15% | ₹22.50 | ₹15.14 | 16,513 | 550 | ₹1,530 |
| 20% (cap, current) | ₹30.00 | ₹22.64 | 11,042 | 368 | ₹1,440 |

Insurance moves break-even at 10% from ~943 to **~1,091 rides/day**, and it makes 5%
commission structurally unviable: at ₹0.14 contribution per ride the platform cannot
carry its own compliance cost.

Hyderabad, `F`=₹180, `T`=15, `FC`=₹400,000, `VC`≈₹7.96:

| `c` | `CM_A` | Break-even rides/day | Driver net/day |
|---|---|---|---|
| 10% | ₹10.04 | 1,328 | ₹2,430 |
| 20% | ₹28.04 | 475 | ₹2,160 |

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
| ₹300 | ₹4.17 | −₹3.19 | 2.8% | **never** | ₹1,750 |
| ₹500 | ₹6.94 | −₹0.42 | 4.6% | **never** | ₹1,717 |
| ₹700 | ₹9.72 | ₹2.36 | 6.5% | 3,531 | ₹1,683 |
| ₹1,080 | ₹15.00 | ₹7.64 | 10.0% | 1,091 | ₹1,617 |

Mandatory insurance pushes the ₹500/week subscription **below break-even per ride**. A
flat fee has to cover a flat compliance cost before it covers anything else, which is
precisely the structure a subscription is worst at.

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

Model A at 10%, Vijayawada, break-even rides/day (base **1,091**):

| Variable | −20% | base | +20% |
|---|---|---|---|
| Average fare `F` | 1,773 | 1,091 | 789 |
| Incentives + refunds | 988 | 1,091 | 1,217 |
| Support cost | 1,047 | 1,091 | 1,139 |
| **Maps cost** | **1,085** | **1,091** | **1,097** |
| Fixed cost `FC` | 873 | 1,091 | 1,309 |

A ±20% swing in Maps spend moves break-even by **6 rides/day**. A ±20% swing in average
fare moves it by **~490**. Price realisation and fixed-cost discipline dominate; Maps
does not.

Utilisation sensitivity, Model B at ₹500/week:

| Rides/driver/week | `c_eff` | `CM_B`/ride |
|---|---|---|
| 24 | 13.9% | ₹13.47 |
| 48 | 6.9% | ₹3.06 |
| 72 | 4.6% | −₹0.42 |
| 96 | 3.5% | −₹2.15 |

Above ~65 rides/week the platform **loses money per ride** at ₹500. Model B needs a
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
3. **Break-even is reachable.** ~1,091 rides/day in Vijayawada; the cheapest
   driver-acceptable subscription (₹500/week) never breaks even at all once mandatory
   insurance is carried.
4. **It is a real differentiator.** Incumbents charge ~20%, which drivers publicly
   object to. Half that is a recruiting argument that costs us ₹180/driver/day against
   the 20% case — buy supply with it rather than with cash incentives, which are already
   the single largest variable cost at ₹3.00/ride.

**Hybrid, only after legal review:** a 5% base commission with an optional weekly fee
that caps total platform take for high-utilisation drivers. It keeps the per-fare share
as the legal instrument while giving committed drivers a predictable ceiling. Do not
build it until a lawyer has signed off on the subscription leg.

**Do not stay at 20%.** It is the owner-driver legal maximum, it matches the incumbents,
and the sensitivity table shows we do not need it: 10% breaks even inside the pilot's
plausible volume range, even after carrying MVAG 2025's mandatory driver insurance.

---

## 7. Recalculating

```
VC   = maps + payment + sms + infra + support + incentives + insurance
CM_A = c × F − VC
BE_A = FC / CM_A                        (rides per month)
R_B  = S / (T × D)
CM_B = R_B − VC
c_eff= S / (T × D × F)
```

Change `F`, `T`, `D`, `c`, `S`, `FC` or any `VC` component in §2 and re-run. Maps cost
per ride by volume comes from `maps-pricing-architecture.md` §2; it is ₹0 below roughly
1,000 rides/day because of India's 70,000-event free tier per SKU.
