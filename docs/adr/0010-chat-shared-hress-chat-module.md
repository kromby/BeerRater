# ADR-0010: Chat: shared Hress chat module

**Status:** Accepted (2026-11-05). Amends ADR-0003, ADR-0004, ADR-0006, ADR-0008 and ADR-0009.

## Context

ADR-0004 planned a `BeerChatMessages` table for the chat about each beer. The legacy hress.org site already has a chatboard: categories and threads are `adm_Component` rows (`TypeId = 10`, a thread's `ParentId` is its category), and messages are `scr_Text` rows (`TypeId = 36`) with insert, update and soft-delete columns. Images already reuse an existing Hress feature (ADR-0006). Chat can do the same, and porting it gives Hress.Org a chat backend it will need anyway when the rest of the chatboard moves to React.

## Decision

**Decided by kromby:**

- Port the chat backend to `Hress.Org/api` as a new, general domain module, **`Ez.Hress.Chat`**, over the existing `adm_Component`/`scr_Text` tables.
- BeerRater's chat is a thin wrapper around it. There's one "Beer Rater" category, and one thread per beer per season, created with the first message.

**Designer's call** (kromby can override):

- The `BeerChatThreads` table (PartitionKey season ID, RowKey beer ID, both URL-encoded, property `ThreadID`) maps a beer in a season to its thread. It replaces `BeerChatMessages`.
- `HressChatAccess` in `Ez.Hress.BeerRater` is the only BeerRater class that uses `Ez.Hress.Chat`.
- The beer chat API keeps its `beer/` routes and its own `ChatMessage` shape, so the frontend doesn't depend on Hress's chat model.
- **Text format:** members write plain text, with no HTML or Markdown. It's stored HTML-encoded, with `<br />` for line breaks, so that every `scr_Text` row stays HTML like the legacy content. The legacy chat page prints `TextValue` unescaped, so storing raw text would allow script injection there. The API decodes stored text back to plain text, and the frontend renders it as text, never as HTML.
- **Photo per message:** later, as a nullable `scr_Text.ImageId` (BR-30 sub-issues #42–#44).

## Consequences

- **ADR-0003:** `Ez.Hress.BeerRater` also references `Ez.Hress.Chat`, through `HressChatAccess` only. The chat module joins `adm_User` and `upf_Image` for author names and photos, so the beer chat gets them from the API instead of from the Hress users API.
- **ADR-0004:** chat messages are in Hress SQL, not Table Storage. BeerRater's chat storage is `BeerChatThreads`, a mapping only.
- **ADR-0006/ADR-0009:** after extraction, BeerRater calls the Hress API from the server for chat as well as images. `HressChatAccess` is the class to swap.
- **ADR-0008:** the plain-text rule is now concrete: encode on write, decode on read, render as text. Ownership checks for edit and delete (BR-31) work against `scr_Text.InsertedBy`.
- Beer chats also appear in the hress.org chatboard under "Beer Rater".
- The "Beer Rater" category row is created by kromby in production, and its ID is set in the Function App settings.
- Whether to migrate legacy HTML messages to plain text is left until the full chatboard UI is built.

## Source

[kromby/BeerRater#20](https://github.com/kromby/BeerRater/issues/20) and its sub-issues #38–#44.
