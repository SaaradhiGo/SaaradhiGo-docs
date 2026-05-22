# ADR-0005: Ops web console as a Next.js app under apps/web

* Status: Accepted
* Date: 2026-05-22

## Context

Ops + support staff need a UI to handle drivers (KYC approvals), trips (state lookups, refunds, dispute triage), withdrawals, support tickets, and pricing zones. Today everything happens in Django admin -- usable, but raw, with no role-aware UX, no batch workflows, and no purpose-built screens for the high-frequency tasks (approve driver, reply to ticket, eyeball day's KPIs).

The `SaaradhiGo-web` monorepo has empty `apps/web` and `apps/android` / `apps/ios` placeholders. The Phase-0 web need is the **ops console**; the rider web experience is Phase-2+.

## Decision

Scaffold `apps/web` as a **Next.js 14 (App Router) + TypeScript + Tailwind** application. The `(console)` route group holds the authenticated UI; `/login` is the public OTP entry point.

Tech choices:

| Concern | Choice | Why |
|---|---|---|
| Framework | Next.js 14 App Router | File-based routing, server-component option for later, React 18 mainstream. |
| Language | TypeScript (strict) | Catches API-shape mismatches against the backend payloads at compile time. |
| Styling | Tailwind | No design system needed; brand colours defined in `tailwind.config.ts`. |
| State | None (per-page `useState`) | The pages are coarse-grained CRUD; no shared client state worth pulling in TanStack Query / Redux yet. Revisit when we add cross-screen caches. |
| Auth | localStorage + bearer tokens | Same OTP flow as mobile; admin role bound at user model. No NextAuth or server-side sessions for now. |
| Hosting | Static-export friendly | The app is a pure SPA; can be served by S3+CloudFront, Vercel, or behind the same nginx as the backend. |

Phase-0 surface (shipped):

* `/dashboard` -- daily KPIs from `/ride/admin/dashboard/`.
* `/trips` -- status-filtered list from `/ride/admin/trips/`.
* `/drivers` -- list + KYC approve.
* `/support` -- ticket list (status-filtered) from `/support/admin/tickets/`.
* `/zones` -- list zones + show effective rate cards from `/pricing/admin/...`.

Deliberately deferred to Phase-1:

* Driver detail page (vehicle expiries, withdrawals, ride history, ratings).
* Trip detail (refund actions, chat history reader, SOS link).
* Support reply UI (model + endpoint exist; the UI is a follow-up).
* CSV export.
* Live map of active trips.
* Refresh-token rotation + idle timeout (currently a 15-min access window means an ops user signs in again every 15 min; acceptable while ops team is < 5 people).

## Consequences

### Positive
* Ops gets a purpose-built UI without spinning up a separate framework / language stack.
* Same backend, same endpoints -- if the API works for the console, it works for mobile and vice versa.
* TypeScript + the backend's existing envelope shape means most field-shape regressions are caught at build.
* Easy to host: `next build && next start` or pure static export.

### Negative
* Two app codebases (mobile + web) to keep in sync as the backend evolves. Mitigated by the strict typing in `src/lib/api.ts` -- the response shapes are documented inline.
* No automated test coverage on the console yet. Manual QA is the gate; an `npm run lint` + `tsc --noEmit` CI run is the next quick win.

### Neutral
* No design system. Tailwind + brand colours are sufficient for ops; if the rider web experience lands later, that one will want a shared component library.

## Alternatives considered

1. **Stick with Django admin.** Cheapest, but the ergonomics for non-engineers are poor (no purpose-built KYC approve flow, no inbox-style support reply, no dashboard tile view). Punted.
2. **Plain HTML + htmx served by Django.** Lower JS footprint but ops staff want SPA-style transitions and the cost of Next.js is genuinely small. Rejected.
3. **Vite + React Router.** Equivalent power; Next.js wins on file-based routing and the option to add server components later.
4. **Refine.dev / React-Admin.** Too prescriptive; we lose styling control and the abstractions don't map cleanly to our envelope/error format.

## Implementation

Scaffold under `apps/web/` in [SaaradhiGo-web](https://github.com/SaaradhiGo/SaaradhiGo-web). Single PR adds `package.json`, `next.config.js`, `tsconfig.json`, `tailwind.config.ts`, `postcss.config.js`, `.env.example`, the `src/` tree (login, layout, 5 pages, API client), and a README.

## Follow-ups

* GitHub Actions workflow producing the `build/web` + `test/web` status checks the monorepo branch-protection rules expect.
* Sentry SDK wiring.
* Vercel preview deployments per PR (or equivalent S3+CloudFront pipeline).
* Driver detail + Trip detail + Support reply pages.
* Refresh-token rotation.
