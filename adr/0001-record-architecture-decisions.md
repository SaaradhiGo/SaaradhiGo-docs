# ADR 0001: Record Architecture Decisions

- **Status:** Accepted
- **Date:** 2026-05-13
- **Deciders:** Founding engineering

## Context

As we approach Phase-0 launch (Hyderabad pilot), we need a lightweight way to record significant architectural decisions — what we chose, why, and what we considered. Decisions made in chat, Slack, or PR descriptions get lost; new engineers re-litigate them; and we lose institutional context as the team grows.

## Decision

We will record significant architecture decisions as **Architecture Decision Records (ADRs)** in `adr/NNNN-short-title.md` in this repository.

- "Significant" = anything that changes how a load-bearing component is built, deployed, or talks to others. Choice of database, payment gateway, queue, deploy model, auth mechanism, scaling strategy, third-party SDK — yes. Variable rename — no.
- Each ADR is numbered, dated, and immutable once Accepted. Superseded ADRs are not edited; a new ADR is added that references the prior.
- Format follows Michael Nygard's classic template: Context → Decision → Consequences.

## Consequences

- New engineers can read `adr/` chronologically and understand the system's evolution.
- We accept the small ongoing cost of writing 1–4 ADRs per quarter.
- ADRs are not specifications; they don't replace `system-design.md` or in-code documentation, they explain *why* the system looks the way it does.
