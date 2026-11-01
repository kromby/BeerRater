# BeerRater MVP: architecture and design

- **Date:** 2026-11-01
- **Issue:** [kromby/BeerRater#36](https://github.com/kromby/BeerRater/issues/36) (BR-00: Define architecture and integration with hress.org)
- **Release:** MVP, Jólabjór 2026, live in early December 2026
- **Status:** draft for approval

## 1. Intent

BeerRater lets Hress members rate and talk about seasonal beers. The first season is Jólabjór 2026.

Tasting is not one sitting. Each person tries beers at home, on their own schedule, over several weeks. One person rates Tuborg Lille Jul in early November and another tries it three weeks later, just before Christmas. BeerRater is where each person logs what they tasted, rates it, and joins a running chat about each beer. When the season ends, the group has a leaderboard and a record of every beer, with averages and other details.

The first users all have Hress accounts. Later, people should be able to share a BeerRater link with people who don't have Hress access. That's the long-term reason BeerRater should become its own site at beer.hress.org.

## 2. Goals, non-goals, anti-goals

**Goals (MVP)**
- Hress members, logged in with their existing account, can on a phone:
  - see this season's beers
  - register a beer that isn't there yet
  - rate a beer from 1 to 5 stars
  - post to a chat about a beer, with text and an optional photo
  - see a leaderboard and each beer's average and details
- Live on hress.org in early December 2026.
- Built so it can later be moved out to beer.hress.org without a rewrite.

**Non-goals (not in the MVP; later)**
- Guests and invite links, and users without a Hress account.
- A standalone site at beer.hress.org, and a shared login service.
- Season types in use (Jólabjór, Þorrabjór, Páskabjór…) and multiple seasons running at once.
- Groups (separate groups of users rating separately).
- Any admin UI. The single admin (kromby) creates and ends seasons, merges duplicate beers and removes messages directly in the database.
- Likes, reactions, threaded replies and notifications.

**Anti-goals (would make this a failure even if it works)**
- BeerRater code or tables tangled into the rest of Hress.Org, so that extracting it later costs a rewrite. The reason for wanting it standalone is that a growing Hress.Org monolith makes React and .NET upgrades harder.
- Logging a beer on a phone being slow or fiddly. It has to be quick to do right after tasting.
- Duplicate beers piling up because finding an existing beer is harder than registering a new one.

## 3. Constraints and context

- **The Hress API can't be called from another site.** It runs as Azure Functions managed by the Static Web App (SWA). It's reached at `https://www.hress.org/api/...` and has no CORS support, so a browser on another origin can't call it. This is why the MVP lives inside hress.org.
- **Login.** hress.org stores a 7-day JWT (HS256, `sub` = user ID) in `localStorage` and sends it as `X-Custom-Authorization: token <jwt>`. There is no cookie, so nothing can be shared across subdomains.
- **Images.** The image API exists: `POST /api/images` takes an `ImageEntity` with base64 `Content` and a `Container`, requires the JWT, and returns an image href. `GET /api/images/{id}/content` serves an image.
- **Stack.**
  - React 19 with TypeScript, React Router v7, TanStack Query v5 and Axios.
  - .NET 9 isolated Azure Functions (`staticwebapp.config.json` → `dotnet-isolated:9.0`).
  - SQL Server, Azure Table Storage and Blob Storage.
  - Conventions are in `Hress.Org/ARCHITECTURE_RULES.md` and `Hress/CLAUDE.md`.
- **Deployment.** The GitHub Actions workflow `azure-static-web-apps-green-rock-0b87cac03.yml` deploys the frontend and the API together on push to `main`.
- **Language.** The UI language is Icelandic, like the rest of hress.org.

## 4. Architecture decisions

Each of these becomes an ADR in the BeerRater repo (see §8). *Decided* means kromby decided it in the brainstorming conversation. *Designer's call* means it was proposed and not objected to, and kromby can override it.

### 4.1 Frontend: a section of the existing hress.org frontend (decided)
BeerRater lives at **hress.org/beer**, inside the Hress.Org React app, for the MVP. It doesn't get its own frontend app or its own subdomain yet.

### 4.2 Integration with hress.org (decided)
BeerRater uses hress.org's existing layout, menu and look and feel. A menu entry links to `/beer`. Because it's the same app, navigation and styling are consistent.

### 4.3 Backend: new endpoints in the Hress.Org API (decided)
BeerRater's endpoints are added to `Hress.Org/api` as a new domain module, **`Ez.Hress.BeerRater`**. It follows the same Entities / UseCases / DataAccess structure as `Ez.Hress.Hardhead`, and its functions are registered in `Ez.Hress.FunctionsApi`. There is no separate API app.

### 4.4 Data (decided: Azure Table Storage; table layout is the designer's call)
- **Storage.** BeerRater uses **Azure Table Storage**, in the existing Hress storage account (`Ez.Hress.Shared.ContentStorage.ConnectionString`), not SQL Server (decided). It follows the Table Storage pattern in `ARCHITECTURE_RULES.md`: domain entities in `Entities/`, `{Entity}TableEntity` adapters in `DataAccess/`, plural table names and URL-encoded RowKeys.
- **Only production exists.** There is **one** database and **one** storage account: production (decided; there is no test environment). That shapes how BeerRater is built and tested:
  - **Local development** of BeerRater's own tables runs against the **Azurite** storage emulator. A separate setting, e.g. `Ez.Hress.BeerRater.Storage.ConnectionString`, points the BeerRater tables at Azurite locally and falls back to the content storage connection when it's unset (designer's call). This keeps test data out of production.
  - **Creating tables and the blob container.** BeerRater's tables are created on first use with `CreateIfNotExists`, so no manual DDL is needed. The `beer` blob container is created by kromby before launch, because the existing blob access code doesn't create containers.
  - **Hiding work until launch.** Unfinished work can merge to `main` safely because nothing links to `/beer` until the menu row is added (see §5.6).
- **Isolation.** BeerRater's tables are used only by `Ez.Hress.BeerRater`, and nothing else in Hress reads or writes them. Users are referenced by Hress user ID only. All table names start with `Beer`.
- **Tables (MVP):**

  | Table | PartitionKey | RowKey | Other properties |
  |---|---|---|---|
  | `BeerSeasons` | `"season"` | season ID | name (e.g. "Jólabjór 2026"), type (a plain field, unused in the MVP), start date, end date (empty while running) |
  | `Beers` | `"beer"` | beer ID | name, producer, alcohol %, country, Vínbúðin URL (optional), registered by (user ID), registered at |
  | `BeerSeasonBeers` | season ID | beer ID | (link only) |
  | `BeerRatings` | season ID | `{beerID}_{userID}` | beer ID, user ID, stars (1–5), created, updated |
  | `BeerChatMessages` | `{seasonID}_{beerID}` | sortable timestamp + unique suffix | user ID, text, image ID (optional, from the image API), created, edited, deleted |

  - **Beers persist across seasons.** A beer is registered **once** and kept between seasons, so Tuborg Lille Jul next year is the same Beer. Adding a beer to a season is a `BeerSeasonBeers` row.
  - **Ratings.** There is **one rating per user, per beer, per season**, and a user can change theirs. Ratings are stored per user (decided). The MVP shows only your own rating, the average and the count, but the data supports showing who rated what later without changes.
  - **Leaderboard and averages** are computed in code by reading the season's `BeerRatings` partition, a few hundred rows at most. There's no aggregate table.
  - **Exact keys and ID formats** (GUIDs, slugs or ints) are the builder's call within this layout.
- **Not in the model yet:** groups and guests. Adding them later should only mean new properties or tables (for example a group ID in the partition key), not reshaping these.

### 4.5 Authentication and roles (decided: existing Hress login)
- Use the existing Hress JWT and `X-Custom-Authorization` header, validated with the existing `AuthenticationInteractor` (`Ez.Hress.Shared`) via `AuthenticationUtil` (`Ez.Hress.FunctionsApi`), so auth checks happen in the function layer. You have to be logged in to see anything under `/beer`.
- **Roles in the MVP:** every logged-in Hress user is a member. There is no admin role in code, because admin work is done in the database.
- The guest token and guest role are deferred (see §7).

### 4.6 Images (decided: reuse the existing image API)
- Chat photos are uploaded through the existing `POST /api/images` from the frontend, in the same way the rest of the site uploads images. The returned image ID is stored on the ChatMessage.
- *Designer's call:* add a `Beer` value to `ImageContainer`, in both the backend enum and the frontend's `src/types/image.ts`, with its blob container, so BeerRater photos are kept separate, instead of going into `Other`.
- **Size.** Phone photos should be resized or compressed on the client before upload. The builder chooses the size.

### 4.7 Hosting and deployment (decided)
Use the existing hress.org Static Web App and GitHub Actions workflow. There's no new Azure resource, no DNS change and no new CI/CD for the MVP. BeerRater code lives in the **Hress.Org repo**. The **BeerRater repo** holds the issues, the ADRs and this design, and later holds the extracted app.

### 4.8 Security (designer's call)
- CORS doesn't apply, because everything is same-origin.
- No new secrets.
- Invite-link abuse doesn't arise until guests exist.
- Chat text goes through DOMPurify (or is rendered as plain text), like the rest of the site.
- Users can only edit or delete their own messages and change their own ratings. The API checks this against the JWT's user ID.

## 5. MVP features

The wireframes from the conversation are in `.superpowers/brainstorm/` in the BeerRater repo. That folder isn't committed. Layouts there are rough and are the builder's to refine.

1. **Season beer list (`/beer`).** Shows the current season's beers only. There's no browser for past seasons in the MVP. It has a search box. Each beer shows name, producer, country, alcohol % and whether *you* have rated it, with your stars if so. A "register a new beer" button sits below the list.
2. **Register a beer.** kromby **loads the season's beer list before launch** (decided), directly in Table Storage or via a small one-off script; the builder's choice. Members can still register a beer that's missing from the list, as in the flow kromby described. Fields: name, producer, alcohol %, country, and an optional Vínbúðin link. Before saving, the user should see existing beers that match the name, including beers from earlier seasons, so they can pick one instead of creating a duplicate. Saving adds the beer to the current season and goes straight to its page.
3. **Beer page.**
   - **Details:** name, producer, country, alcohol % and the Vínbúðin link.
   - **Ratings:** your rating with the existing `StarRating` component (`src/components/StarRating`, as used in Hardhead), plus the beer's average and number of ratings.
   - **Chat:** below the ratings.
   - **No spoiler protection:** everyone sees the average and the chat, rated or not.
4. **Chat.**
   - A running conversation per beer per season, oldest first. Anyone can post any number of times.
   - A message is text, an optional photo, or both.
   - You can edit or delete your own messages.
   - There are no likes, reactions or threads.
5. **Leaderboard.**
   - **Live:** it updates all season, and when the season ends it becomes the final result.
   - **Ranking:** by average rating. A beer needs a **minimum of 3 ratings** to be ranked (designer's call). Beers below that are listed underneath, unranked. Ties go to the beer with more ratings.
   - **Sorting:** you can also sort by number of ratings, alcohol % or country.
   - **Rows:** each row shows the average, number of ratings, number of chat messages and your own rating, and links to the beer page.
6. **Menu entry.** A link to `/beer` in the hress.org menu. The menu is database rows served by `/api/menus`, not code, so the entry can be added at launch, keeping `/beer` unlinked until then.

**The current season** is the one season with no end date. If there's none, `/beer` shows the most recently ended season read-only: no new ratings, beers or messages. This is how the admin "ends" a season in the database.

## 6. Keeping extraction cheap (rules for the builder)

- **Backend.** All BeerRater backend code is in `Ez.Hress.BeerRater`. Other modules don't reference it, and it only depends on `Ez.Hress.Shared` for auth.
- **Frontend.** All BeerRater frontend code is under `src/pages/beer/` and its own `components/`, `hooks/` and `types/` folders, or a `beer` subfolder in each. Other pages don't import from it.
- **Hress dependencies.** BeerRater's use of Hress login, the current user and images goes through one small frontend module (e.g. `src/pages/beer/hress.ts`) and one backend class. Extraction then means swapping those pieces.
- **Storage.** BeerRater's Table Storage tables store Hress user IDs only and never read Hress SQL tables. Showing user names and photos (e.g. in the chat) goes through the existing Hress users API from the frontend's Hress module, not through joins in BeerRater's backend.
- **API routes.** API routes are prefixed `beer/` (e.g. `/api/beer/seasons/current/beers`) so they can be moved together.

## 7. Later: extraction to beer.hress.org (direction, not MVP)

This gets recorded as an ADR so the intent isn't lost.

- **Trigger:** guests, or anyone without a Hress account, needing access.
- **Shape:**
  - BeerRater gets its own Static Web App at beer.hress.org, with its own React frontend and .NET API, upgraded on its own schedule.
  - It calls the Hress API from the server only (no CORS needed) for anything it still needs, such as images.
  - A shared login service (e.g. auth.hress.org) is extracted from Hress.Org. It issues tokens signed with an asymmetric key, so both sites can validate them with a public key and neither needs the signing secret. The same service issues limited guest tokens.
- **Then:** season types and groups.

## 8. Deliverables from this design

1. **ADRs** in the BeerRater repo, one per decision in §4 plus one for §7.
2. **BR-01 rewrite.** BR-01 is rewritten to cover only building the integration: the menu link, the `/beer` route, `Ez.Hress.BeerRater` wiring and the image container. BR-00's analysis is answered by these ADRs.
3. **Backlog alignment (decided: this design wins).** The existing backlog (BR-01…BR-54) has `mvp` stories that contradict this design. Stories for guests, groups, admin screens, hidden averages (BR-22), the photo gallery and main image (BR-32–35), the my-ratings page (BR-23) and season progress (BR-42) are relabelled `later`. Stories that stay in the MVP but differ from this design are rewritten to match it: BR-13, BR-20, BR-21, BR-24, BR-30 and BR-41. kromby sees every change before it's posted.
4. **`.gitignore`.** Add `.superpowers/` to `.gitignore` in the BeerRater repo.

## 9. Left to the builder

- Exact key formats and property names within the table layout in §4.4.
- How kromby's pre-launch beer list gets loaded: by hand in Storage Explorer, or with a small script.
- API endpoint shapes beyond the `beer/` prefix.
- UI layout and styling within hress.org's look, Icelandic copy, and star-level labels (Hardhead has its own text per star).
- Image resize limits, and how duplicate-beer matching works (simple name matching is fine).
- Whether the leaderboard is a separate route or a tab on `/beer`.
- Test approach, following the repo's existing xUnit and Jest setup.
