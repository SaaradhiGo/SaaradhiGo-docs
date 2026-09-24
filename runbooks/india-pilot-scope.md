# India pilot scope

- **Date:** 2026-09-24
- **Assumption:** one controlled city, a small vetted driver cohort, cash-first.
- **Basis:** the actual state of SaaradhiGo as inventoried on 2026-09-24, not a
  generic ride-hailing feature list.

## The principle

A pilot exists to find out whether the core loop works with real people in real
traffic. Anything that does not serve that loop is a distraction, and anything the
loop cannot survive without is a blocker no matter how small it looks.

Uber has fifteen years of features. Copying that list is how a pilot never launches.

## MUST HAVE

Not negotiable, because without any one of these a rider or driver cannot complete a
ride safely or the business cannot explain what happened.

| Capability | State today |
|---|---|
| Rider OTP login + token refresh | implemented, refresh verified in the driver client |
| Driver OTP login + token refresh | implemented, 401-refresh-and-replay verified |
| Driver KYC + approval through the audited path | implemented, exercised in QA |
| Location permission + GPS on both apps | implemented |
| Pickup and destination selection | implemented (incl. a precise-pickup screen) |
| Server-authoritative fare quote | implemented, single `quote_fare` engine |
| Ride request with idempotency | implemented — per-rider `client_request_id` + PostgreSQL partial unique index |
| Dispatch to nearby drivers | implemented, Celery-owned waves |
| Offer accept with race resolution | implemented under `SELECT FOR UPDATE` |
| OTP-gated trip start | implemented, with a 5-attempt lockout |
| Lifecycle commands with acknowledgement | implemented — `command_ack` committed/already_done/rejected |
| Reconnect into durable state | implemented, greeting carries PostgreSQL truth |
| GPS trail for evidence | implemented, bounded at 12 points/min, per-trip cap enforced |
| Cash payment + driver cash confirmation | implemented, settlement verified end to end in QA |
| Driver earnings visible and correct | implemented, read-only |
| Receipt | implemented, versioned, PDF + S3 |
| SOS, rider and driver | implemented, durable-first, repeat-collapsed, unaffected by GPS load |
| Support ticket path | implemented, rider-facing + admin reply |
| Cancellation, all paths | implemented; **no cancellation fee** and none displayed |
| Admin: driver approval, live trips, trip search, SOS queue, payment lookup | implemented across two consoles — see below |
| Production app builds that cannot point at QA | implemented this run, both apps |
| Immutable fare snapshot | implemented this run (schema; nothing reads it yet) |
| Immutable settlement evidence | implemented this run (`TripSettlement`) |

### MUST HAVE that is NOT yet done

| Gap | Why it is a must |
|---|---|
| **Choose ONE operations console** | Two exist and both are developed. An operator who approves a payout in one and cannot find it in the other will conclude the platform is broken, during an incident. |
| **Cashfree payout contract verified in sandbox** | The refund-on-ambiguous-failure path can plausibly pay a driver *and* refund the platform. Cash-first reduces but does not remove this: drivers still withdraw. |
| **Rider app tests actually passing** | 19 tests were quarantined this run because they had never passed. Three whole screens are unverified. |
| **Production S3, database, Redis and secrets provisioned** | Nothing is deployed to production today. |
| **A named production deployment owner** | Railway and an EC2 workflow both exist; nobody has said which is production. |

## SHOULD HAVE

Real value, and a pilot survives a few weeks without them.

- **Wire the fare breakdown into the rider UI.** `_FareBreakdown` is a finished
  widget and the API already returns matching fields — verified live in QA:
  `{base_fare 60.00, distance_fare 47.60, time_fare 16.80, total 124.40}`. Riders
  currently see a total with no explanation, which is a trust cost in this market.
- **Point earnings and admin revenue at `TripSettlement`.** The table now exists;
  until something reads it, admin revenue still recomputes commission at render time
  and drifts when a rate card changes.
- **Backfill settlements for historical trips**, marked `reconstructed`, after
  running the classification command.
- **Celery Beat separated from the worker.** Embedded `-B` works; it means a worker
  restart is also a scheduler restart.
- **Notification coverage audit.** FCM and in-app exist; which transitions are
  guaranteed has not been established.
- **Pickup landmark / gate note field.** Indian pickups are landmark-driven and a
  coordinate plus a long locality string is often not enough for a driver to find a
  gate.
- **Rider-visible driver rating.** Ratings are captured; surfacing them builds trust.
- **Per-message-type WebSocket rate limits.** Lifecycle and SOS must never be
  throttled by a GPS quota.

## POST-PILOT

Deliberately out of scope. Each would consume the pilot's whole engineering budget
and none of them is needed to learn whether the core loop works.

- Pooling / shared rides
- Scheduled and recurring rides
- Multi-stop trips
- Subscriptions, loyalty, referrals
- Complex promo mechanics (stacking, targeting, budgets) — promo stays **inactive**
- Corporate / business accounts
- Advanced heatmaps and predictive positioning (an aspirational
  `predictive_heatmaps` screen already exists and should not be taken as a commitment)
- AI or ML dispatch — dispatch is wave-based and that is correct for one city
- Driver loyalty tiers (a `driver_loyalty` screen exists; the mechanics do not)
- In-app wallet as a primary payment method — it exists; cash-first is the pilot
- Metered billing from the GPS trail. The evidence chain is being built for this
  deliberately and slowly; `final_fare` stays NULL until someone decides the policy.
- iOS, unless a business reason appears. Both apps are Android-first today.
- Multi-city / multi-zone operations beyond the single pilot zone
- Localisation into additional languages — no localisation architecture exists and
  inventing one now is not a pilot activity

## The honest shape of the risk

The backend is the most finished part of this platform and the most tested. The
weakest parts of the pilot are, in order:

1. **Payout correctness**, because it depends on a provider contract nobody has
   verified.
2. **The rider app**, because it is the less-maintained half of a two-sided product
   and three of its screens have no working tests.
3. **Operations**, because there are two consoles and no decision.
4. **Production infrastructure**, because none of it exists yet.

None of those is a code-volume problem, which is why adding features would make the
pilot less likely to succeed rather than more.
