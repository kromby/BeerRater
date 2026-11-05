# ADR-0006: Images: existing image API, `Beer` container

**Status:** Accepted (2026-11-01). Amended by [ADR-0010](0010-chat-shared-hress-chat-module.md).

## Context

Chat messages can include a photo. The image API already exists: `POST /api/images` takes an `ImageEntity` with base64 `Content` and a `Container`, requires the JWT, and returns an image href. `GET /api/images/{id}/content` serves an image.

## Decision

**Decided by kromby:** reuse the existing image API. Chat photos are uploaded from the frontend through `POST /api/images`, as the rest of the site does, and the returned image ID is stored on the chat message.

**Designer's call** (kromby can override): add a `Beer` value to `ImageContainer`, in both the backend enum and the frontend's `src/types/image.ts`, with its blob container, so BeerRater photos are kept separate instead of going into `Other`.

## Consequences

- No new image service. The Hress API's image code is touched only for the enum value (this is the image container work in BR-01).
- Phone photos should be resized or compressed on the client before upload. The builder chooses the size.
- After extraction, BeerRater calls the Hress API from the server only for images (see ADR-0009).

## Source

Design spec [§4.6](../superpowers/specs/2026-11-01-beerrater-mvp-design.md), [§3](../superpowers/specs/2026-11-01-beerrater-mvp-design.md) and [§9](../superpowers/specs/2026-11-01-beerrater-mvp-design.md).
