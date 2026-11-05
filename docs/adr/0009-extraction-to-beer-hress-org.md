# ADR-0009: Extraction to beer.hress.org and a shared login service

**Status:** Accepted (2026-11-01). Amended by [ADR-0010](0010-chat-shared-hress-chat-module.md).

## Context

The first users have Hress accounts, but later people should be able to share a BeerRater link with people who don't. A growing Hress.Org monolith also makes React and .NET upgrades harder. The Hress API has no CORS and its login token lives in `localStorage`, so a separate site can't use either directly.

## Decision

This is direction, not MVP. It is recorded so the intent isn't lost.

- **Trigger:** guests, or anyone without a Hress account, needing access.
- **Shape:**
  - BeerRater gets its own Static Web App at beer.hress.org, with its own React frontend and .NET API, upgraded on its own schedule.
  - It calls the Hress API from the server only (no CORS needed) for anything it still needs, such as images.
  - A shared login service (e.g. auth.hress.org) is extracted from Hress.Org. It issues tokens signed with an asymmetric key, so both sites can validate them with a public key and neither needs the signing secret. The same service issues limited guest tokens.
- **Then:** season types and groups.

## Consequences

- The MVP must be built so extraction doesn't need a rewrite: the module, folder, storage and route rules in ADR-0001, ADR-0003 and ADR-0004.
- Extraction then means swapping the small Hress frontend module and backend class.
- Guests, a standalone site, a shared login service, season types and groups are all out of the MVP.

## Source

Design spec [§7](../superpowers/specs/2026-11-01-beerrater-mvp-design.md), [§6](../superpowers/specs/2026-11-01-beerrater-mvp-design.md) and [§2](../superpowers/specs/2026-11-01-beerrater-mvp-design.md).
