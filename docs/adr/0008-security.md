# ADR-0008: Security

**Status:** Accepted (2026-11-01). Amended by [ADR-0010](0010-chat-shared-hress-chat-module.md).

## Context

BeerRater runs same-origin inside hress.org, for logged-in members only (ADR-0001, ADR-0005).

## Decision

Designer's call (proposed and not objected to; kromby can override):

- CORS doesn't apply, because everything is same-origin.
- No new secrets.
- Invite-link abuse doesn't arise until guests exist.
- Chat text goes through DOMPurify (or is rendered as plain text), like the rest of the site.
- Users can only edit or delete their own messages and change their own ratings. The API checks this against the JWT's user ID.

## Consequences

- Chat text is the user-generated content to guard, hence the DOMPurify or plain-text rule.
- Ownership checks live in the API, not only in the UI.

## Source

Design spec [§4.8](../superpowers/specs/2026-11-01-beerrater-mvp-design.md).
