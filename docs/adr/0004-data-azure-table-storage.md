# ADR-0004: Data: Azure Table Storage, five tables, production-only environment

**Status:** Accepted (2026-11-01)

## Context

BeerRater needs storage for seasons, beers, ratings and chat. There is **one** database and **one** storage account, production, and no test environment. BeerRater data must be isolated so it can move out later.

## Decision

**Decided by kromby:** use **Azure Table Storage** in the existing Hress storage account (`Ez.Hress.Shared.ContentStorage.ConnectionString`), not SQL Server, following the Table Storage pattern in `ARCHITECTURE_RULES.md` (entities in `Entities/`, `{Entity}TableEntity` adapters in `DataAccess/`, plural table names, URL-encoded RowKeys). Only production exists. Ratings are stored per user, one per user, per beer, per season.

**Designer's call** (kromby can override): the table layout below, and a separate `Ez.Hress.BeerRater.Storage.ConnectionString` setting (an example name) that points local development at Azurite and falls back to the content storage connection when unset.

| Table | PartitionKey | RowKey |
|---|---|---|
| `BeerSeasons` | `"season"` | season ID |
| `Beers` | `"beer"` | beer ID |
| `BeerSeasonBeers` | season ID | beer ID |
| `BeerRatings` | season ID | `{beerID}_{userID}` |
| `BeerChatMessages` | `{seasonID}_{beerID}` | sortable timestamp + unique suffix |

## Consequences

- Local development runs against the **Azurite** emulator, keeping test data out of production.
- Tables are created on first use with `CreateIfNotExists`. The `beer` blob container is created by kromby before launch, because the existing blob access code doesn't create containers.
- Beers persist across seasons and are registered once. Adding a beer to a season is a `BeerSeasonBeers` row.
- Leaderboard and averages are computed in code from the season's `BeerRatings` partition (a few hundred rows at most). There is no aggregate table.
- Only `Ez.Hress.BeerRater` uses these tables. All names start with `Beer`, and users are referenced by Hress user ID only.
- Groups and guests are not in the model. Adding them later should mean new properties or tables, not reshaping these.
- Exact keys, ID formats and property names are left to the builder.

## Source

Design spec [§4.4](../superpowers/specs/2026-11-01-beerrater-mvp-design.md) and [§9](../superpowers/specs/2026-11-01-beerrater-mvp-design.md).
