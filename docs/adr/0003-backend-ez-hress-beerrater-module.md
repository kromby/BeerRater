# ADR-0003: Backend: `Ez.Hress.BeerRater` module in the Hress.Org API

**Status:** Accepted (2026-11-01)

## Context

The Hress API can't be called from another site (no CORS), and BeerRater runs inside hress.org (ADR-0001). A growing Hress.Org monolith makes React and .NET upgrades harder, so BeerRater code must not get tangled into the rest of Hress.

## Decision

Decided by kromby. BeerRater's endpoints are added to `Hress.Org/api` as a new domain module, **`Ez.Hress.BeerRater`**. It follows the same Entities / UseCases / DataAccess structure as `Ez.Hress.Hardhead`, and its functions are registered in `Ez.Hress.FunctionsApi`. There is no separate API app.

## Consequences

Rules for keeping extraction cheap:

- All BeerRater backend code is in `Ez.Hress.BeerRater`. Other modules don't reference it, and it only depends on `Ez.Hress.Shared` for auth.
- BeerRater's use of Hress login goes through one small backend class.
- It stores Hress user IDs only and never reads Hress SQL tables. User names and photos come through the existing Hress users API from the frontend, not through joins in the backend.
- API routes are prefixed `beer/` (e.g. `/api/beer/seasons/current/beers`) so they can be moved together.
- Endpoint shapes beyond the `beer/` prefix are left to the builder.

## Source

Design spec [§4.3](../superpowers/specs/2026-11-01-beerrater-mvp-design.md) and [§6](../superpowers/specs/2026-11-01-beerrater-mvp-design.md).
