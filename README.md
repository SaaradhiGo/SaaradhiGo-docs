# SaaradhiGo Docs

Central documentation repository for the SaaradhiGo / VahanGo ride-hailing platform. Engineering teams across the [backend](https://github.com/SaaradhiGo/SaaradhiGo-backend), [mobile](https://github.com/SaaradhiGo/SaaradhiGo-mobile), and [web](https://github.com/SaaradhiGo/SaaradhiGo-web) repos consult this repo for architecture, decisions, and operational runbooks.

## Contents

- **[system-design.md](system-design.md)** — Top-level system architecture, covering Phase-0 (Hyderabad pilot) and the 12-month north-star target. Start here.
- **[adr/](adr/)** — Architecture Decision Records. Short, dated records of significant decisions and their context. Add a new one (`adr/NNNN-title.md`) whenever a load-bearing decision is made.
- **[diagrams/](diagrams/)** — Source files for diagrams that don't render inline as Mermaid (PlantUML, Excalidraw, etc.) and their exported SVG/PNG versions.
- **[runbooks/](runbooks/)** — On-call runbooks: incident response, deploy/rollback, common alerts. Populated as the platform stabilises.
- **[legal/](legal/)** — User-facing legal documents and compliance SOPs (drafts; require Indian-qualified lawyer review before publication). See [legal/README.md](legal/README.md) for the index.
- **[business/](business/)** — Phase-0 commercial decisions: pricing schedule, commission split, surge policy, incentive budgets, KPIs.

### Phase-0 launch documents

Business + legal artefacts required before the Hyderabad pilot can go live:

| Document | Purpose |
|---|---|
| [legal/terms-of-service.md](legal/terms-of-service.md) | Rider-facing Terms of Service (aggregator framing, liability cap, arbitration, grievance officer per IT Rules 2021). |
| [legal/privacy-policy.md](legal/privacy-policy.md) | DPDP Act 2023 compliant privacy policy, including third-party processor list, retention, and `/me/export`+`/me/delete` rights. |
| [legal/driver-agreement.md](legal/driver-agreement.md) | Driver Partner agreement — independent contractor framing, commission terms, payout cadence, TDS u/s 194O. |
| [legal/mva-2020-driver-verification-sop.md](legal/mva-2020-driver-verification-sop.md) | Operations SOP for onboarding & verifying drivers per Motor Vehicles Aggregator Guidelines 2020. |
| [legal/mva-2020-driver-verification-checklist.md](legal/mva-2020-driver-verification-checklist.md) | Per-driver fillable checklist used during the SOP. File one copy per applicant in the KYC archive. |
| [legal/gst-tds-registration-guide.md](legal/gst-tds-registration-guide.md) | Step-by-step founder guide: GST (Sec 9(5) aggregator), TAN + TDS u/s 194O, Professional Tax, Shops & Establishments, GHMC trade licence. |
| [business/pricing-phase-0.md](business/pricing-phase-0.md) | Phase-0 fare schedule (auto/hatchback/sedan/SUV), surge cap, cancellation policy, commission split, incentive + promo budgets, KPI watchlist. |

## How to use this repo

- Diagrams in `system-design.md` are written in [Mermaid](https://mermaid.js.org/), which GitHub renders natively.
- When you change architecture in code, update `system-design.md` in the same PR (or open a follow-up within the week).
- New significant decisions get a numbered ADR. Past ADRs are immutable — supersede with a new one rather than editing.
- Keep this repo lean. Code-level docs (READMEs, API references, WebSocket event tables) live in the repo they describe, not here.

## Related repositories

- [SaaradhiGo-backend](../SaaradhiGo-backend) — Django REST + Channels + Celery backend
- [SaaradhiGo-mobile](../SaaradhiGo-mobile) — Flutter rider app (VahanGo)
- [SaaradhiGo-web](../SaaradhiGo-web) — Web client monorepo & branch protection rules
