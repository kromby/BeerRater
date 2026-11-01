# ADR-0001: Frontend: section of hress.org at /beer

**Status:** Accepted (2026-11-01)

## Context

The first users all have Hress accounts. The Hress API runs as Azure Functions managed by the Static Web App (SWA), is reached at `https://www.hress.org/api/...` and has no CORS support, so a browser on another origin can't call it. Login is a 7-day JWT in `localStorage`, with no cookie, so nothing can be shared across subdomains. A standalone site would therefore be cut off from both the API and the login.

## Decision

Decided by kromby. For the MVP, BeerRater lives at **hress.org/beer**, inside the existing Hress.Org React app. It doesn't get its own frontend app or its own subdomain yet.

## Consequences

- The browser calls the Hress API and uses the Hress login same-origin, so no CORS or cross-site login work is needed.
- All BeerRater frontend code stays under `src/pages/beer/` (with its own `components/`, `hooks/` and `types/`), and other pages don't import from it, so it can be extracted later (see ADR-0009).
- Its use of Hress login, the current user and images goes through one small frontend module (e.g. `src/pages/beer/hress.ts`).
- The UI language is Icelandic, like the rest of hress.org.

## Source

Design spec [§4.1](../superpowers/specs/2026-11-01-beerrater-mvp-design.md) and [§6](../superpowers/specs/2026-11-01-beerrater-mvp-design.md).
