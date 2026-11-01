# ADR-0007: Hosting and deployment: existing SWA and workflow

**Status:** Accepted (2026-11-01)

## Context

The Hress.Org frontend and API deploy together through the GitHub Actions workflow `azure-static-web-apps-green-rock-0b87cac03.yml` on push to `main`.

## Decision

Decided by kromby. Use the existing hress.org Static Web App and GitHub Actions workflow. There is no new Azure resource, no DNS change and no new CI/CD for the MVP. BeerRater code lives in the **Hress.Org repo**. The **BeerRater repo** holds the issues, the ADRs and the design, and later holds the extracted app.

## Consequences

- Nothing new to provision: no Azure resource, DNS change or CI/CD.
- Unfinished work can merge to `main` safely because nothing links to `/beer` until the menu row is added (see ADR-0002).
- The MVP goes live on hress.org in early December 2026.

## Source

Design spec [§4.7](../superpowers/specs/2026-11-01-beerrater-mvp-design.md), [§3](../superpowers/specs/2026-11-01-beerrater-mvp-design.md) and [§2](../superpowers/specs/2026-11-01-beerrater-mvp-design.md).
