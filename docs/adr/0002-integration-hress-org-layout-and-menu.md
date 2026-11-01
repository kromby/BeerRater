# ADR-0002: Integration: hress.org layout and menu

**Status:** Accepted (2026-11-01)

## Context

BeerRater runs inside the Hress.Org app (ADR-0001). The hress.org menu is database rows served by `/api/menus`, not code.

## Decision

Decided by kromby. BeerRater uses hress.org's existing layout, menu and look and feel. A menu entry links to `/beer`. Because it's the same app, navigation and styling are consistent.

## Consequences

- No separate design system or navigation to build.
- The menu entry is added at launch as a row served by `/api/menus`, so `/beer` stays unlinked until then. Unfinished work can merge to `main` safely.

## Source

Design spec [§4.2](../superpowers/specs/2026-11-01-beerrater-mvp-design.md) and [§5.6](../superpowers/specs/2026-11-01-beerrater-mvp-design.md).
