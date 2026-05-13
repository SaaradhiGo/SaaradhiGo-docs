# SaaradhiGo Docs

Central documentation repository for the SaaradhiGo / VahanGo ride-hailing platform. Engineering teams across the [backend](https://github.com/SaaradhiGo/SaaradhiGo-backend), [mobile](https://github.com/SaaradhiGo/SaaradhiGo-mobile), and [web](https://github.com/SaaradhiGo/SaaradhiGo-web) repos consult this repo for architecture, decisions, and operational runbooks.

## Contents

- **[system-design.md](system-design.md)** — Top-level system architecture, covering Phase-0 (Hyderabad pilot) and the 12-month north-star target. Start here.
- **[adr/](adr/)** — Architecture Decision Records. Short, dated records of significant decisions and their context. Add a new one (`adr/NNNN-title.md`) whenever a load-bearing decision is made.
- **[diagrams/](diagrams/)** — Source files for diagrams that don't render inline as Mermaid (PlantUML, Excalidraw, etc.) and their exported SVG/PNG versions.
- **[runbooks/](runbooks/)** — On-call runbooks: incident response, deploy/rollback, common alerts. Populated as the platform stabilises.

## How to use this repo

- Diagrams in `system-design.md` are written in [Mermaid](https://mermaid.js.org/), which GitHub renders natively.
- When you change architecture in code, update `system-design.md` in the same PR (or open a follow-up within the week).
- New significant decisions get a numbered ADR. Past ADRs are immutable — supersede with a new one rather than editing.
- Keep this repo lean. Code-level docs (READMEs, API references, WebSocket event tables) live in the repo they describe, not here.

## Related repositories

- [SaaradhiGo-backend](../SaaradhiGo-backend) — Django REST + Channels + Celery backend
- [SaaradhiGo-mobile](../SaaradhiGo-mobile) — Flutter rider app (VahanGo)
- [SaaradhiGo-web](../SaaradhiGo-web) — Web client monorepo & branch protection rules
