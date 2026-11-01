# ADR-0005: Authentication: existing Hress JWT, members only

**Status:** Accepted (2026-11-01)

## Context

hress.org stores a 7-day JWT (HS256, `sub` = user ID) in `localStorage` and sends it as `X-Custom-Authorization: token <jwt>`. There is no cookie. Admin work in the MVP is done directly in the database.

## Decision

Decided by kromby. Use the existing Hress JWT and `X-Custom-Authorization` header, validated with the existing `AuthenticationInteractor` (`Ez.Hress.Shared`) via `AuthenticationUtil` (`Ez.Hress.FunctionsApi`), so auth checks happen in the function layer. You have to be logged in to see anything under `/beer`.

Roles in the MVP: every logged-in Hress user is a member. There is no admin role in code.

## Consequences

- No new login, token or secret.
- Guests and the guest token and role are deferred (see ADR-0009).
- A user's identity is the JWT's user ID, which the API checks for edits and deletes (see ADR-0008).

## Source

Design spec [§4.5](../superpowers/specs/2026-11-01-beerrater-mvp-design.md) and [§3](../superpowers/specs/2026-11-01-beerrater-mvp-design.md).
