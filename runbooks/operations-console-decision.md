# Operations console: which one is canonical

Decision date 2026-09-24. Decided from evidence gathered against the deployed QA
backend, not from preference.

## The situation

SaaradhiGo has **two** operations consoles, both live, neither declared canonical.

| | Django console | Next.js console |
|---|---|---|
| Location | `SaaradhiGo-backend/servers/admin_dashboard/` | `SaaradhiGo-web/apps/web/src/app/(console)/` |
| Rendering | Server-rendered Django templates | React 19 / Next 15.5.26 SPA |
| Pages | 19 | 11 |
| Mounted at | `path('', include(admin_urls))` — **the backend root** | Separate hosting required |
| Auth | Django session + role check | JWT in `localStorage` |
| Automated tests | Yes (`tests/test_admin_dashboard_authz.py` and others, inside the backend's 635-test suite) | **None** |
| CI | Yes, via the backend pipeline | **None until today** |

## Decision

**KEEP the Django console as the canonical operations console for the India
pilot. The Next.js console is not pilot-ready and must not be an operator's only
route to any control.**

This is a decision about the pilot, not a verdict on the architecture. See
[Why not simply delete one](#why-not-simply-delete-one).

## The evidence that decided it

### 1. The Next console had a broken driver-approval page

`app/(console)/drivers/page.tsx` called `/driver/admin/list/`. That endpoint does
not exist. Probed against the deployed QA backend, unauthenticated, where 404
means "no such route" and 401 means "exists, protected":

```
/api/v1/driver/admin/                       401  EXISTS (protected)
/api/v1/driver/admin/list/                  404  *** MISSING ***
```

The page caught the error and rendered "Could not load drivers". The Drivers page
is where an operator sees the driver queue and approves KYC — the exact screen
needed to onboard the first ten real drivers. **Fixed** in this pass; the backend
route is `/driver/admin/`.

Every other endpoint the Next console depends on does exist. All 19 were probed:

```
ride/admin/dashboard/            401     support/admin/tickets/          401
ride/admin/trips/                401     support/admin/tickets/1/reply/  401
ride/admin/trips/1/              401     support/admin/tickets/1/assign/ 401
ride/trip/1/receipt/resend/      401     support/tickets/1/              401
driver/admin/1/full/             401     driver/admin/withdrawals/       401
driver/admin/1/update-kyc/       401     .../withdrawals/1/approve/      401
sos/admin/                       401     .../withdrawals/1/reject/       401
payments/refund/                 401     pricing/admin/zones/            401
pricing/admin/rate-cards/        401
```

The SOS action names match the backend exactly (`acknowledge`, `resolve`,
`false-alarm`). So the console was **one typo away from being correctly wired** —
which is the point. It was not a mockup. It was a working console with an
undetected 404 on its most important page, and nothing in the repository could
have told anyone.

### 2. The Next console had no CI and no tests

No `.github/workflows`. No `*.test.*`, no `*.spec.*`. A single-character path
error on the driver approval screen survived to a deployed state because nothing
ran. **Added in this pass** (`.github/workflows/ops-console.yml`): lint,
type-check and build, all three verified green locally first.

While adding it, a second CI trap: `npm run lint` ran `next lint`, which is
deprecated in Next 15 and, with no ESLint config present, **prompts
interactively** — it would have hung a runner rather than failed it. There was no
`.eslintrc.json` at all. Both fixed; the script now uses the non-interactive
ESLint CLI.

Current state of the three gates, run locally:

```
npm run lint         exit 0
npm run type-check   exit 0   (tsc --noEmit)
npm run build        exit 0   (11 routes, 103 kB shared JS)
```

### 3. The Django console is a genuine feature superset

Not merely larger — it covers every control the Next console has:

| Capability | Django | Next |
|---|---|---|
| Fleet / live dashboard | `fleet_monitor` | `dashboard` |
| Trips list + detail | `ride`, `ride_detail` | `trips`, `trips/[id]` |
| Driver onboarding / KYC | `driver_onboarding`, `driver_profile` | `drivers`, `drivers/[id]` |
| Driver payouts approve/reject | `payment_dashboard` (with a `status != "pending"` guard) | `withdrawals` |
| SOS / emergency | `emergency_dashboard` | `sos` |
| Support / disputes | `dispute_support` | `support`, `support/[id]` |
| Zones + rate cards | `fare_surge` | `zones` |
| Refunds | `transaction_dashboard` | via `trips/[id]` |
| Riders | `riders` | — |
| Promo codes | `promo_codes` | — |
| Executive revenue | `executive_revenue` | — |
| Driver loyalty | `driver_loyalty` | — |
| Predictive heatmaps | `predictive_heatmaps` | — |
| Global search | `global_search` + API | — |
| Notifications | `notifications` | — |

### 4. Deployment and auth favour the Django console for a pilot

- It is mounted at the backend root, so it ships with the API. There is no second
  deploy target, no second domain, no second environment variable to get wrong on
  the night of a launch.
- Its auth was **read and verified**, not assumed. `admin_required` requires
  `user.is_authenticated` **and** (`role == "admin"` or `is_superuser`), and is
  applied to every view except `login` and `logout`; `update_global_config` uses
  `staff_member_required`. `test_admin_dashboard_authz.py` covers it.
- The Next console keeps its JWT in `localStorage`, which its own source comment
  acknowledges as a deliberate trade for static bundling. Any XSS in the console
  yields an admin token. That is a worse posture for a console that can approve
  payouts and issue refunds.

## What this decision does not excuse

Two findings that apply regardless of which console wins, and neither is fixed:

1. **The admin login has no rate limiting and no MFA.** It accepts a phone number
   and password at the backend root and is the front door to payout approval,
   refunds and KYC. Brute-force protection and a second factor are both absent.
   This is a launch-blocking security question, not a console-choice question.
2. **The Next console reads page 1 only.** `/driver/admin/` paginates at 10. Fine
   for a pilot cohort of ten drivers; wrong for a real fleet. Noted in the source.

## Why not simply delete one

Deleting the Next console now would be the wrong call, and so would promoting it:

- It is not dead code. It builds, type-checks, lints clean, and 18 of its 19
  endpoints were already correct. Someone invested real work in it and it is
  closer to done than a glance suggests.
- The Django console's 19 pages are ~6,500 lines in a single `views.py`. That is
  the thing that actually needs attention, and it is not a good long-term home.
- But **a pilot is not the time to migrate an operations console.** The canonical
  one for the pilot must be the one with auth coverage, tests and CI behind it.

So: **MIGRATE, later, deliberately** — not now, and not by accident.

### Actions taken now

- [x] Fix the Next console's 404 on the driver-approval page.
- [x] Add CI to the web repository (lint, type-check, build).
- [x] Add the missing ESLint config and make the lint script non-interactive.
- [x] Record the decision here so an operator is not guessing at 2am.

### Actions still open

- [ ] **Tell the pilot operators which console to use**, in writing, with a URL.
      Two consoles and no stated default is an operational hazard on its own.
- [ ] Rate-limit and add a second factor to the admin login. Launch-blocking.
- [ ] Add at least one end-to-end test per Next console page that asserts the
      endpoint it calls returns something other than 404. That single class of
      test would have caught the defect above.
- [ ] Decide the long-term target and write the migration down. Do it after the
      pilot, with the Django console's `views.py` split up as part of it.
- [ ] Either finish the Next console's missing pages (riders, promo codes,
      revenue, loyalty, heatmaps, global search, notifications) or state plainly
      that it is a partial console and the Django one remains authoritative.

## How the endpoint evidence was gathered

Repeatable without credentials, because 404 and 401 are distinguishable:

```bash
B='https://backend-qa-811d.up.railway.app'
for p in /api/v1/driver/admin/ /api/v1/driver/admin/list/ ; do
  code=$(curl -s -o /dev/null -m 20 -w "%{http_code}" "$B$p")
  printf "%-42s %s\n" "$p" "$code"
done
```

401 or 403 means the route exists and is protected. 404 means the console is
calling something that is not there. This is worth running against every admin
endpoint the console depends on whenever either side changes.
