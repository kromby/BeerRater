# BeerRater MVP Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Ship BeerRater as a self-contained section at hress.org/beer. Hress members browse the season's beers, register missing ones, rate them 1–5 stars, chat about each beer with text and photos, and follow a live leaderboard.

**Architecture:**
- **Backend.** A new .NET domain module, `Ez.Hress.BeerRater`, inside the Hress.Org API. It stores its data in five Azure Table Storage tables and is exposed through Azure Functions under `api/beer/…`.
- **Frontend.** A React section under `src/pages/beer/` in the same SPA.
- **Hress dependencies.** Login, user lookup and image upload are reached through one frontend module (`src/pages/beer/hress.ts`) and the existing auth helper, so the section can be lifted out to beer.hress.org later.

**Tech Stack:**
- **Backend:** .NET 9 isolated Azure Functions, Azure.Data.Tables 12.11.0, xUnit 2.9 and Moq 4.20.
- **Frontend:** React 19 with TypeScript, React Router 7, TanStack Query v5, Axios, and Jest with Testing Library.

**Spec:** `docs/superpowers/specs/2026-11-01-beerrater-mvp-design.md` (BeerRater repo). Read it before starting any task.

## Delivery: one PR per issue slice

Each slice is a vertical piece of work: backend, then frontend, then a PR that closes its issues.

**Before the first step of each slice,** wait for the previous PR **in the same repo** to merge (A and B are independent), then branch from an up-to-date `main`: `git switch main && git pull && git switch -c <branch>`, using the branch from the table below. The Hress.Org checkout is currently on `update/fb-image-stuff`; never branch from it.

| PR | Repo | Branch | Tasks | Closes |
|---|---|---|---|---|
| A | BeerRater (`/home/kromby/source/BeerRater`) | `docs/br-00-architecture` | 1, 2 | BR-00 (#36) |
| B | Hress.Org (`/home/kromby/source/Hress/Hress.Org`) | `feat/br-01-beer-section` | 3, 4 | BR-01 (#1), except the menu row (Task 12) |
| C | Hress.Org | `feat/br-10-beers` | 5, 6 | BR-10 (#8), BR-11 (#9), BR-12 (#10), BR-13 (#11) |
| D | Hress.Org | `feat/br-20-ratings` | 7, 8 | BR-20 (#15), BR-21 (#16), BR-24 (#19) |
| E | Hress.Org | `feat/br-30-chat` | 9, 10 | BR-30 (#20), BR-31 (#21) |
| F | Hress.Org | `feat/br-41-leaderboard` | 11 | BR-41 (#27) |
| G | BeerRater | `docs/launch-runbook` | 12 | Last step of BR-01 (#1): the launch |

**PRs A and B are opened together and link to each other.** Every PR body links its issues and the spec. Opening a PR is outward-facing: show kromby the title and body first.

**Merging Hress.Org PRs deploys to production.** That's safe because nothing links to `/beer` until the menu row is added at launch (Task 12).

## Global Constraints

- **Tables:** `BeerSeasons`, `Beers`, `BeerSeasonBeers`, `BeerRatings`, `BeerChatMessages`. Keys are exactly as in spec §4.4.
- **API routes:** all start with `beer/`.
- **Module isolation:**
  - No module other than `Ez.Hress.FunctionsApi` references `Ez.Hress.BeerRater`.
  - `Ez.Hress.BeerRater` references only `Ez.Hress.Shared`.
  - BeerRater never reads Hress SQL tables.
- **Frontend isolation:** all BeerRater frontend code lives under `src/pages/beer/`. The only edits outside it are:
  - the `/beer` routes in `src/pages/App.js`
  - `ImageContainer.Beer` in `src/types/image.ts`
  - an optional `container` parameter on `useImages().uploadImage` in `src/hooks/useImages.ts`
- **`hress.ts` is the only file in `src/pages/beer` that imports from outside the folder.** The exception is `src/components/StarRating`.
- **Ratings:** whole stars, integer 1–5. A user has one rating per beer per season and can change it but not remove it.
- **Leaderboard:** a beer needs at least **3** ratings to be ranked. It ranks by average (descending), then by rating count (descending). Beers with the same average and count share a rank (competition ranking: 1, 1, 3). Unranked beers come after the ranked ones, ordered by name.
- **Auth:** every `beer/` endpoint requires the Hress JWT via `AuthenticationUtil.GetAuthenticatedUserID` and returns 401 otherwise. The frontend sends `X-Custom-Authorization: token <jwt>`.
- **Writes need an open season.** Any write to a season whose `EndDate` is set fails with 409.
- **Exception → HTTP mapping** lives in one helper, `BeerResults.From(Exception)`. Every beer function uses it.

  | Exception | Status |
  |---|---|
  | `ArgumentException` | 400 |
  | `UnauthorizedAccessException` | 403 |
  | `KeyNotFoundException` | 404 |
  | `DuplicateBeerException` | 409 with body `{existing}` |
  | `InvalidOperationException` | 409 |
  | anything else | 500, logged |

- **JSON:** all responses are camelCase JSON (the existing Newtonsoft MVC setup).
- **Local storage:** set the setting `Ez.Hress.BeerRater.Storage.ConnectionString` to `UseDevelopmentStorage=true` (Azurite). When it's unset, BeerRater falls back to `Ez.Hress.Shared.ContentStorage.ConnectionString`. Never run BeerRater locally against production storage, and never commit `local.settings.json`.
- **Backend conventions:**
  - Logging uses the format `[{Class}.{Method}] message`.
  - Interactors have no interfaces; data access classes do.
  - Everything is registered as a singleton in `Program.cs`.
  - Table access constructors take `(BeerStorageInfo, ILogger<T>)` and call `CreateIfNotExists()`. Follow `Ez.Hress.Hardhead/DataAccess/RuleChangeTableDataAccess.cs` and `RuleChangeTableEntity.cs`.
  - Store all `DateTime`s as UTC.
- **Testing conventions:**
  - Backend tests live in `api/Ez.Hress.UnitTest/BeerRater/`, with Moq mocks and an Arrange/Act/Assert layout like `Hardhead/AwardNominateTests.cs`. Run them with `dotnet test api/Ez.Hress.FunctionsApi.sln --filter "FullyQualifiedName~Ez.Hress.UnitTest.BeerRater"`.
  - Frontend tests use the setup from `src/pages/hardhead/components/guestsEdit.test.tsx`: mock axios and `react-global-configuration`, and wrap in `QueryClientProvider` + `MemoryRouter` + `AuthContext.Provider`. Run them with `npm test -- --watchAll=false --testPathPattern=src/pages/beer`.
- **Frontend conventions:**
  - Prettier style: double quotes, trailing commas (es5), 2-space indent, semicolons.
  - The UI copy is Icelandic, and its wording is the builder's choice.
  - Tests find elements by role, label or `data-testid`, not by copy.
- **Before each Hress.Org PR,** run the full `dotnet test api/Ez.Hress.FunctionsApi.sln`, `npm test -- --watchAll=false` and `npm run dbuild`. All must pass.
- **Anything posted to GitHub** (issue edits, labels, PRs) is shown to kromby first and posted only after an explicit yes.

## Review Focus

1. **A closed season.** If kromby sets `EndDate`, every write must return 409. The UI shows the season read-only: no register button, read-only stars, no chat box. Tested in Tasks 3, 4, 5, 6, 7, 8, 9 and 10.
2. **Changing someone else's message.** Editing or deleting another user's message returns 403 and leaves the message unchanged. Tested in Task 9.
3. **Near-duplicate beers.** Registering "  tuborg  LILLE jul " by "Carlsberg" when "Tuborg Lille Jul" by "Carlsberg" exists returns 409 with the existing beer. Search finds it case- and whitespace-insensitively. Tested in Tasks 5 and 6.
4. **Re-rating.** Rating the same beer again replaces the earlier rating, so the average counts each user once. Tested in Task 7.
5. **HTML in chat text.** A message like `<img src=x onerror=alert(1)>` renders as literal text. Tested in Task 10.

---

## PR A — BR-00 (BeerRater repo)

### Task 1: Commit the design and write the ADRs

**Files:**
- Create: `.gitignore` containing `.superpowers/`
- Create: `docs/adr/0001-frontend-section-of-hress-org.md` … `docs/adr/0009-extraction-to-beer-hress-org.md`
- Add: `docs/superpowers/specs/2026-11-01-beerrater-mvp-design.md`, `docs/superpowers/plans/2026-11-01-beerrater-mvp.md`

**Interfaces:**
- Produces: the ADR paths that Task 2 and the PR bodies link to.

- [ ] **Step 1:** Create the `docs/br-00-architecture` branch (see Delivery).
- [ ] **Step 2: Write the nine ADRs.** Each uses the format `# ADR-000N: <title>`, then **Status:** Accepted (2026-11-01), then sections **Context**, **Decision**, **Consequences** and **Source** (link to the spec section). Write the content from the spec; don't add new decisions. Mark "designer's call" items as such. The ADRs map to the spec like this:

  | ADR | Spec |
  |---|---|
  | 0001 Frontend: section of hress.org at /beer | §4.1 |
  | 0002 Integration: hress.org layout and menu | §4.2 |
  | 0003 Backend: `Ez.Hress.BeerRater` module in the Hress.Org API | §4.3, §6 |
  | 0004 Data: Azure Table Storage, five tables, production-only environment | §4.4 |
  | 0005 Authentication: existing Hress JWT, members only | §4.5 |
  | 0006 Images: existing image API, `Beer` container | §4.6 |
  | 0007 Hosting and deployment: existing SWA and workflow | §4.7 |
  | 0008 Security | §4.8 |
  | 0009 Extraction to beer.hress.org and a shared login service | §7 |

- [ ] **Step 3: Verify.** Run `ls docs/adr | wc -l`. Expected: `9`. Run `git status --short`. Expected: `.superpowers/` doesn't appear.
- [ ] **Step 4: Commit.**

```bash
git add .gitignore docs/
git commit -m "docs: BR-00 architecture decisions, MVP design and plan"
```

### Task 2: Align the GitHub backlog (needs kromby's approval)

**Files:**
- Create: `/tmp/claude-1000/-home-kromby-source/af1c6548-2f3b-4900-8b06-8e02efee2be9/scratchpad/backlog-changes.md` (proposal only, not committed)

**Interfaces:**
- Consumes: the ADR paths from Task 1, used as links.

- [ ] **Step 1: Write the proposal file**, with exact new titles and bodies, covering:
  - **Relabel `mvp` → `later`** (remove `mvp`, add `later`): #2 BR-02, #3 BR-03, #4 BR-04, #6 BR-06, #7 BR-07, #12 BR-14, #17 BR-22, #18 BR-23, #22 BR-32, #23 BR-33, #24 BR-34, #25 BR-35, #26 BR-40, #28 BR-42, #30 BR-44.
  - **#1 BR-01.** Retitle to "BR-01: BeerRater section at hress.org/beer". The body covers the `Ez.Hress.BeerRater` wiring, `ImageContainer.Beer`, the `/beer` route behind login, and the menu entry at launch. It links ADR-0001, ADR-0003 and ADR-0006. Add a line saying the original "open beer.hress.org already signed in" intent moved to ADR-0009. Keep the `mvp` label.
  - **#11 BR-13.** Retitle to "BR-13: Browse and search this season's beers" (search, no filters).
  - **#15 BR-20.** Retitle to "BR-20: Rate a beer with 1–5 stars" (whole stars).
  - **#16 BR-21.** Retitle to "BR-21: Change my rating" (removing a rating is out of scope).
  - **#19 BR-24.** Retitle to "BR-24: Beer page shows my rating, the average and the number of ratings" (no groups).
  - **#20 BR-30.** Retitle to "BR-30: Chat about a beer" (per beer per season, text and/or one photo).
  - **#27 BR-41.** Retitle to "BR-41: Season leaderboard" (live, minimum 3 ratings, ties by count).
- [ ] **Step 2: Show the proposal to kromby and wait for an explicit yes.** Apply their edits to the file.
- [ ] **Step 3: Apply** with `gh issue edit <n> --repo kromby/BeerRater --title … --body-file … --add-label later --remove-label mvp`.
- [ ] **Step 4: Verify.** Run `gh issue list --repo kromby/BeerRater --label mvp --json number -q 'map(.number)|sort'`. Expected: `[1,8,9,10,11,15,16,19,20,21,27,36]`.

**PR A:** push `docs/br-00-architecture` and open a PR with "Closes #36". Open it together with PR B (after Task 4) so each body can link the other.

---

## PR B — BR-01: BeerRater section at hress.org/beer (Hress.Org)

### Task 3: Module, seasons, current-season endpoint and the image container

**Files:**
- Create the project `api/Ez.Hress.BeerRater/Ez.Hress.BeerRater.csproj`. Copy the `PropertyGroup` from `Ez.Hress.Hardhead.csproj`. Packages: `Azure.Data.Tables` 12.11.0 and `Microsoft.Extensions.Logging.Abstractions` 9.0.16. ProjectReference: `..\Ez.Hress.Shared`.
- Create:
  - `api/Ez.Hress.BeerRater/Entities/Season.cs`, `Entities/BeerStorageInfo.cs`
  - `UseCases/IBeerSeasonDataAccess.cs`, `UseCases/SeasonInteractor.cs`
  - `DataAccess/BeerSeasonTableAccess.cs`, `DataAccess/SeasonTableEntity.cs`, `DataAccess/SeasonBeerTableEntity.cs`
  - `api/Ez.Hress.FunctionsApi/Beer/BeerSeasonFunctions.cs`, `api/Ez.Hress.FunctionsApi/Beer/BeerResults.cs`
- Modify:
  - `api/Ez.Hress.FunctionsApi.sln`: add the project with `dotnet sln add`
  - `api/Ez.Hress.UnitTest/Ez.Hress.UnitTest.csproj` and `api/Ez.Hress.FunctionsApi/Ez.Hress.FunctionsApi.csproj`: add a ProjectReference to `Ez.Hress.BeerRater`
  - `api/Ez.Hress.FunctionsApi/Program.cs`: add a `// BeerRater` block at the end of `ConfigureServices`
  - `api/Ez.Hress.Shared/Entities/ImageEntity.cs`: append `Beer` to `enum ImageContainer`, after `ATVR`
- Test: `api/Ez.Hress.UnitTest/BeerRater/SeasonInteractorTests.cs`, `api/Ez.Hress.UnitTest/Shared/ImageInteractorTests.cs`

**Interfaces:**
- Produces:
  - `public record BeerStorageInfo(string ConnectionString);`
  - `public class Season { string ID; string Name; int? TypeID; DateTime StartDate; DateTime? EndDate; bool IsOpen => EndDate == null; }`
  - `IBeerSeasonDataAccess`: `Task<IList<Season>> GetSeasonsAsync()`, `Task<IList<string>> GetSeasonBeerIDsAsync(string seasonID)`, `Task AddBeerToSeasonAsync(string seasonID, string beerID)`. Implement all three now; the season–beer link is the `BeerSeasonBeers` table (PK season ID, RK beer ID).
  - `SeasonInteractor(IBeerSeasonDataAccess, ILogger<SeasonInteractor>)`:
    - `Task<Season?> GetCurrentSeasonAsync()`
    - `Task<Season> GetSeasonAsync(string seasonID)`. Throws `KeyNotFoundException`.
    - `Task<Season> GetOpenSeasonAsync(string seasonID)`. Throws `KeyNotFoundException` if the season is unknown, or `InvalidOperationException` if it's closed.
  - `BeerResults.From(Exception ex, ILogger log): IActionResult`, mapping exceptions as in Global Constraints
  - HTTP: function `beerCurrentSeason`, GET `beer/seasons/current` → `Season` (including `isOpen`), or 404 when there are no seasons

- [ ] **Step 1: Write the failing tests:**

```csharp
// SeasonInteractorTests
[Fact] CurrentSeason_ReturnsRunningSeason()            // seasons: ended 2025 + running 2026 → ID "jolabjor-2026"
[Fact] CurrentSeason_TwoRunning_ReturnsLatestStart()
[Fact] CurrentSeason_NoneRunning_ReturnsLatestEnded()  // result.IsOpen == false
[Fact] CurrentSeason_NoSeasons_ReturnsNull()
[Fact] OpenSeason_Closed_ThrowsInvalidOperation()
[Fact] OpenSeason_Unknown_ThrowsKeyNotFound()

// ImageInteractorTests (existing file, same mock setup as its other tests)
[Fact] SaveBeerContainer_SetsBlobBeerPhotoUrl()
//   save new ImageEntity(0, "x", "") { Content = [1], Container = ImageContainer.Beer }
//   Assert: blob content data access is called; saved entity PhotoUrl starts with "BLOB:/beer/"
```

- [ ] **Step 2: Run the tests and confirm they fail.** Run the BeerRater filter and `--filter "FullyQualifiedName~ImageInteractorTests"`. Expected: build errors, because the types and `Beer` don't exist yet.
- [ ] **Step 3: Implement the module, `ImageContainer.Beer`, the function and DI.** The function's first statement after logging is the `AuthenticationUtil.GetAuthenticatedUserID` check, as in `Hardhead/HardheadRuleFunctions.RunRuleChanges`. Register DI like this:

```csharp
// BeerRater
var beerStorage = config["Ez.Hress.BeerRater.Storage.ConnectionString"];
services.AddSingleton(new BeerStorageInfo(string.IsNullOrEmpty(beerStorage) ? config["Ez.Hress.Shared.ContentStorage.ConnectionString"]! : beerStorage));
services.AddSingleton<IBeerSeasonDataAccess, BeerSeasonTableAccess>();
services.AddSingleton<SeasonInteractor>();
```

- [ ] **Step 4: Run the tests and the build.** Run `dotnet build api/Ez.Hress.FunctionsApi.sln && dotnet test api/Ez.Hress.FunctionsApi.sln`. Expected: all tests pass, including the existing ones.
- [ ] **Step 5: Smoke-test locally against Azurite.**
  1. Start Azurite (`npx azurite --silent --location /tmp/azurite`).
  2. Add `"Ez.Hress.BeerRater.Storage.ConnectionString": "UseDevelopmentStorage=true"` to your own `local.settings.json`.
  3. Run `func start` from `api/Ez.Hress.FunctionsApi`.
  4. Check these:
     - `curl -i localhost:7072/api/beer/seasons/current` with no header → `401`
     - the same call with `-H "X-Custom-Authorization: token $JWT"` (a token from a local login) → `404`
     - insert a `BeerSeasons` row (`PartitionKey=season`, `RowKey=jolabjor-2026`, `Name=Jólabjór 2026`, `StartDate=2026-11-01T00:00:00Z`) with Storage Explorer → the call returns `200` with `"id":"jolabjor-2026"` and `"isOpen":true`
- [ ] **Step 6: Commit** with `git commit -m "feat(beer): BeerRater module, seasons and Beer image container"`.

### Task 4: `/beer` route, login check and the Hress adapter

**Files:**
- Create in `src/pages/beer/`: `types.ts`, `hress.ts`, `hooks/useBeerApi.ts`, `RequireLogin.tsx`, `index.tsx` (BeerHome)
- Modify:
  - `src/pages/App.js`: add `<Route path="beer"><Route path="" element={<RequireLogin><BeerHome /></RequireLogin>} /></Route>`
  - `src/types/image.ts`: add `Beer = 6`
  - `src/hooks/useImages.ts`: add the optional `container?: ImageContainer` to `UploadImageParams`, defaulting to `ImageContainer.Album`
- Test: `src/pages/beer/index.test.tsx`, `src/pages/beer/RequireLogin.test.tsx`

**Interfaces:**
- Consumes: GET `beer/seasons/current` (Task 3).
- Produces:
  - `types.ts`: `Season { id: string; name: string; typeID?: number; startDate: string; endDate?: string; isOpen: boolean }`
  - `hress.ts`:
    - `useHressAuth(): { isLoggedIn: boolean; userID?: number; headers: Record<string, string> }`. It wraps `useAuth()` from `src/context/auth`; the user ID comes from `localStorage.getItem("userID")`.
    - `hressApiUrl(path: string): string`, which returns `${config.get("apiPath")}/api/${path}`
  - `hooks/useBeerApi.ts`: `useCurrentSeason()` using TanStack Query with key `["beer", "season", "current"]`. It returns `null` on 404.
  - `RequireLogin({ children })`: when logged out, renders a link to `/login` instead of its children.

- [ ] **Step 1: Write the failing tests:**

```tsx
// RequireLogin.test.tsx
it("renders a link to /login and not its children when logged out")
it("renders its children when logged in")

// index.test.tsx
it("shows the current season's name")              // /current → {id:"jolabjor-2026", name:"Jólabjór 2026", isOpen:true}
it("shows an empty state when there is no season") // /current → 404; data-testid="no-season"
it("marks a closed season as finished")            // isOpen:false → data-testid="season-closed"
it("sends the Hress token header")                 // axios.get called with headers {"X-Custom-Authorization": "token fake-token"}
```

- [ ] **Step 2: Run the tests and confirm they fail.** Expected: they fail because the modules aren't found.
- [ ] **Step 3: Implement the files above.**
- [ ] **Step 4: Run the full checks** (see Global Constraints). Expected: everything passes, and `npm run dbuild` succeeds.
- [ ] **Step 5: Commit** with `git commit -m "feat(beer): /beer section behind login with Hress adapter"`.

**PR B:** push the branch and open a PR in kromby/Hress.Org. The body says "Part of kromby/BeerRater#1 (menu entry at launch)" and links PR A and the spec. **Show kromby PRs A and B together.**

---

## PR C — BR-10, BR-11, BR-12, BR-13: beers (Hress.Org)

### Task 5: Beers backend

**Files:**
- Create in `api/Ez.Hress.BeerRater/`:
  - `Entities/Beer.cs`, `Entities/DuplicateBeerException.cs`, `Entities/BeerSummary.cs`
  - `UseCases/IBeerDataAccess.cs`, `UseCases/BeerInteractor.cs`, `UseCases/SummaryInteractor.cs`
  - `DataAccess/BeerTableAccess.cs`, `DataAccess/BeerTableEntity.cs`
- Create: `api/Ez.Hress.FunctionsApi/Beer/BeerFunctions.cs`, `api/Ez.Hress.FunctionsApi/Beer/BeerInput.cs`
- Modify: `Program.cs` (the BeerRater block)
- Test: `api/Ez.Hress.UnitTest/BeerRater/BeerInteractorTests.cs`, `api/Ez.Hress.UnitTest/BeerRater/SummaryInteractorTests.cs`

**Interfaces:**
- Consumes: `SeasonInteractor` and `IBeerSeasonDataAccess` (Task 3).
- Produces:
  - `Beer { string ID; string Name; string Producer; double AlcoholPercentage; string Country; string? VinbudinUrl; int RegisteredBy; DateTime RegisteredAt; void Validate(); string NormalizedKey { get; } }`. `NormalizedKey` is the lower-case invariant form of name and producer, trimmed, with inner whitespace collapsed to one space, joined by `|`.
  - `DuplicateBeerException(Beer existing) : Exception { Beer Existing { get; } }`
  - `BeerSummary { Beer Beer; double? Average; int RatingCount; int? MyRating; int MessageCount; int? Rank; }`. In this task only `Beer` is filled; Tasks 7, 9 and 11 fill the rest.
  - `IBeerDataAccess`: `Task<Beer?> GetBeerAsync(string id)`, `Task<IList<Beer>> GetBeersAsync()`, `Task SaveBeerAsync(Beer beer)`
  - `BeerInteractor(IBeerDataAccess, IBeerSeasonDataAccess, SeasonInteractor, ILogger<BeerInteractor>)`:
    - `Task<IList<Beer>> SearchBeersAsync(string? query)`
    - `Task<Beer> RegisterBeerAsync(Beer input, string seasonID, int userID)`
    - `Task AddBeerToSeasonAsync(string seasonID, string beerID)`
    - `Task<IList<Beer>> GetSeasonBeersAsync(string seasonID)`
    - `Task<Beer> GetSeasonBeerAsync(string seasonID, string beerID)`. Throws `KeyNotFoundException` if the beer isn't in that season.
  - `SummaryInteractor(BeerInteractor, ILogger<SummaryInteractor>)`:
    - `Task<IList<BeerSummary>> GetSeasonSummariesAsync(string seasonID, int userID)`, ordered by beer name
    - `Task<BeerSummary> GetBeerSummaryAsync(string seasonID, string beerID, int userID)`
  - HTTP (function names in brackets; `{s}` is the season ID, `{b}` the beer ID):

    | Function | Method and route | Body | Returns |
    |---|---|---|---|
    | `beerSeasonBeers` | GET `beer/seasons/{s}/beers` | | `BeerSummary[]` |
    | `beerSeasonBeers` | POST `beer/seasons/{s}/beers` | `BeerInput {name, producer, alcoholPercentage, country, vinbudinUrl?}` | 201 `Beer`, or 409 `{existing}` |
    | `beerSeasonBeer` | GET `beer/seasons/{s}/beers/{b}` | | `BeerSummary` |
    | `beerSeasonBeer` | POST `beer/seasons/{s}/beers/{b}` | | 204 (add an existing beer to the season) |
    | `beerSearch` | GET `beer/beers?q=` | | `Beer[]` |

- [ ] **Step 1: Write the failing tests:**

```csharp
// BeerInteractorTests
[Fact] Register_Valid_SavesBeerAndAddsToSeason()
//   Assert: SaveBeerAsync called with RegisteredBy == 7, RegisteredAt.Kind == Utc, ID non-empty without '_';
//           AddBeerToSeasonAsync("jolabjor-2026", saved.ID) called once
[Fact] Register_SameNormalizedNameAndProducer_ThrowsDuplicateWithExisting()
//   existing: "Tuborg Lille Jul" / "Carlsberg"; input: "  tuborg  LILLE jul " / "carlsberg"
//   Assert: ex.Existing.ID == existing.ID; SaveBeerAsync never called
[Fact] Register_ClosedSeason_ThrowsInvalidOperation()
[Theory] Register_Invalid_ThrowsArgument(name, producer, country, abv, url)
//   ("", "C", "DK", 5.6, null), ("N", "", "DK", 5.6, null), ("N", "C", "", 5.6, null),
//   ("N", "C", "DK", 0, null), ("N", "C", "DK", 20.1, null),
//   ("N", "C", "DK", 5.6, "https://evil.example/x"), (new string('a',101), "C", "DK", 5.6, null)
[Fact] Register_VinbudinUrl_Accepted()     // "https://www.vinbudin.is/heim/vorur/stoek-vara.aspx/?productid=12345"
[Fact] Search_MatchesNameOrProducer_CaseAndWhitespaceInsensitive()  // "  lille JUL" and "carls" both find Tuborg Lille Jul
[Fact] Search_EmptyQuery_ReturnsEmpty()
[Fact] AddToSeason_AlreadyInSeason_DoesNotAddTwice()
[Fact] AddToSeason_ClosedSeason_ThrowsInvalidOperation()
[Fact] GetSeasonBeer_NotInSeason_ThrowsKeyNotFound()

// SummaryInteractorTests
[Fact] SeasonSummaries_OnePerSeasonBeer_OrderedByName()
[Fact] BeerSummary_NotInSeason_ThrowsKeyNotFound()
```

- [ ] **Step 2: Run the tests and confirm they fail.**
- [ ] **Step 3: Implement.** Validation in `Beer.Validate()` throws `ArgumentException` when any of these fail:
  - name, producer and country are required after trimming
  - name and producer are at most 100 characters; country is at most 60
  - 0 < AlcoholPercentage ≤ 20
  - `VinbudinUrl` is null, empty or starts with `https://www.vinbudin.is/`

  Beer IDs are `Guid.NewGuid().ToString("N")`. The `Beers` table has PK `"beer"` and RK the beer ID.
- [ ] **Step 4: Run the tests and confirm they pass.** Then smoke-test against Azurite: POST the same beer twice → `201`, then `409` with `existing`.
- [ ] **Step 5: Commit** with `git commit -m "feat(beer): register, search and add beers to a season"`.

### Task 6: Season beer list and the register page

**Files:**
- Create: `src/pages/beer/register.tsx`
- Modify:
  - `src/pages/beer/types.ts`: add `Beer`, `BeerSummary` and `BeerInput`
  - `src/pages/beer/hooks/useBeerApi.ts`: add `useSeasonBeers`, `useBeerSearch`, `useRegisterBeer` and `useAddBeerToSeason`
  - `src/pages/beer/index.tsx`: show the list under the season name
  - `src/pages/App.js`: add the `register` route
- Test: `src/pages/beer/index.test.tsx`, `src/pages/beer/register.test.tsx`

**Interfaces:**
- Consumes: the Task 5 HTTP API.
- Produces:
  - `Beer { id; name; producer; alcoholPercentage; country; vinbudinUrl?; registeredBy; registeredAt }`
  - `BeerSummary { beer: Beer; average?: number; ratingCount: number; myRating?: number; messageCount: number; rank?: number }`
  - `BeerInput`
  - Hooks:
    - `useSeasonBeers(seasonID?)` (key `["beer", "season", seasonID, "beers"]`)
    - `useBeerSearch(q)` (key `["beer", "search", q]`, enabled when `q.trim().length >= 2`)
    - `useRegisterBeer(seasonID)` and `useAddBeerToSeason(seasonID)`, which invalidate `["beer"]` on success

- [ ] **Step 1: Write the failing tests:**

```tsx
// index.test.tsx (add)
it("lists the season's beers with producer, country and alcohol %")  // two rows, data-testid="beer-row"
it("filters the list by search text, case-insensitively")             // type "lille" → 1 row
it("links each beer to /beer/:beerID")
it("hides the register button when the season is closed")             // no data-testid="register-beer"

// register.test.tsx
it("shows matching existing beers while typing the name")   // "tuborg" → GET beer/beers?q=tuborg → data-testid="existing-beer"
it("picking an existing beer adds it to the season and navigates to it")  // POST beer/seasons/jolabjor-2026/beers/<id>; location /beer/<id>
it("submits the form and navigates to the new beer")        // POST body {name, producer, alcoholPercentage: 5.6, country, vinbudinUrl}
it("on 409 shows the existing beer instead of an error")    // POST rejects {response:{status:409,data:{existing}}} → existing-beer link
it("blocks submit until name, producer, country and alcohol % are filled")
it("rejects a non-vinbudin.is link before submitting")
```

- [ ] **Step 2: Run the tests and confirm they fail.**
- [ ] **Step 3: Implement.**
  - The list sorts by beer name and is filtered on the client by name or producer.
  - The register page debounces search by 300 ms.
  - Alcohol % uses `type="number"`, `step="0.1"` and `inputMode="decimal"`.
  - The inputs are sized for a phone.
  - The `/beer/:beerID` link target can be a placeholder route until Task 8.
- [ ] **Step 4: Run the full checks.**
- [ ] **Step 5: Commit** with `git commit -m "feat(beer): season beer list and register a beer"`.

**PR C:** "Closes kromby/BeerRater#8, #9, #10, #11".

---

## PR D — BR-20, BR-21, BR-24: ratings and the beer page (Hress.Org)

### Task 7: Ratings backend

**Files:**
- Create: `api/Ez.Hress.BeerRater/Entities/Rating.cs`, `UseCases/IBeerRatingDataAccess.cs`, `UseCases/RatingInteractor.cs`, `DataAccess/BeerRatingTableAccess.cs`, `DataAccess/RatingTableEntity.cs`, `api/Ez.Hress.FunctionsApi/Beer/BeerRatingFunctions.cs`
- Modify: `UseCases/SummaryInteractor.cs` (add an `IBeerRatingDataAccess` dependency) and `Program.cs`
- Test: `api/Ez.Hress.UnitTest/BeerRater/RatingInteractorTests.cs`, `SummaryInteractorTests.cs`

**Interfaces:**
- Consumes: `BeerInteractor.GetSeasonBeerAsync` and `SeasonInteractor.GetOpenSeasonAsync`.
- Produces:
  - `Rating { string SeasonID; string BeerID; int UserID; int Stars; DateTime Created; DateTime Updated; }`
  - `IBeerRatingDataAccess`: `Task<IList<Rating>> GetRatingsAsync(string seasonID)`, `Task<Rating?> GetRatingAsync(string seasonID, string beerID, int userID)`, `Task UpsertRatingAsync(Rating rating)`
  - `RatingInteractor(IBeerRatingDataAccess, BeerInteractor, SeasonInteractor, ILogger<RatingInteractor>)`: `Task<Rating> SaveRatingAsync(string seasonID, string beerID, int userID, int stars)`
  - `SummaryInteractor` now fills `Average`, `RatingCount` and `MyRating`
  - HTTP: function `beerRating`, PUT `beer/seasons/{s}/beers/{b}/rating`, body `{stars}` → `Rating`

- [ ] **Step 1: Write the failing tests:**

```csharp
// RatingInteractorTests
[Fact] Save_NewRating_Upserts()                 // Stars 4, Created == Updated, Kind Utc
[Fact] Save_ExistingRating_KeepsCreated_UpdatesUpdatedAndStars()
[Theory] Save_StarsOutOfRange_ThrowsArgument(0), (6), (-1)
[Fact] Save_ClosedSeason_ThrowsInvalidOperation()
[Fact] Save_BeerNotInSeason_ThrowsKeyNotFound()

// SummaryInteractorTests (add)
[Fact] Summaries_AverageCountsEachUserOnce_AndMyRating()
//   ratings for beer X: user 1 → 5, user 2 → 3, user 3 → 4 (one row per user)
//   Assert: Average == 4.0, RatingCount == 3, MyRating for userID 2 == 3
[Fact] Summaries_NoRatings_AverageNull_CountZero()
```

- [ ] **Step 2: Run the tests and confirm they fail.**
- [ ] **Step 3: Implement.** The `BeerRatings` table has PK season ID and RK `$"{beerID}_{userID}"`. Upsert with `TableUpdateMode.Replace`. Round averages to 2 decimals.
- [ ] **Step 4: Run the tests and confirm they pass.** Smoke test: PUT `{"stars":6}` → `400`; PUT `{"stars":4}` twice → the summary shows `ratingCount` 1.
- [ ] **Step 5: Commit** with `git commit -m "feat(beer): ratings and beer averages"`.

### Task 8: Beer page with stars, and my rating in the list

**Files:**
- Create: `src/pages/beer/beer.tsx` (BeerPage), `src/pages/beer/components/BeerRatingStars.tsx`
- Modify: `hooks/useBeerApi.ts` (add `useSeasonBeer` and `useSaveRating`), `index.tsx` (show my rating per row) and `App.js` (the `:beerID` route)
- Test: `src/pages/beer/beer.test.tsx`, `src/pages/beer/index.test.tsx`

**Interfaces:**
- Consumes: GET `beer/seasons/{s}/beers/{b}`, PUT `…/rating`, and `src/components/StarRating` (`numberOfStars={5}`, `changeRating`, `readonly`).
- Produces: `BeerPage` with a slot under the ratings where Task 10 adds `<ChatThread seasonID beerID isOpen />`.

- [ ] **Step 1: Write the failing tests:**

```tsx
// beer.test.tsx
it("shows name, producer, country, alcohol % and a Vínbúðin link when present")
it("shows my rating, the average with one decimal and the rating count")  // average 3.8333 → /3[.,]8/
it("clicking the 4th star PUTs {stars:4} and refetches")
it("stars are read-only when the season is closed")   // clicking does not call axios.put
it("shows not-found for an unknown beer")             // GET → 404

// index.test.tsx (add)
it("shows my rating on rows I have rated")            // data-testid="my-rating" contains "4"
```

- [ ] **Step 2: Run the tests and confirm they fail.**
- [ ] **Step 3: Implement.** `BeerRatingStars` wraps `StarRating` with the colours used in `src/pages/hardhead/components/rating.tsx`.
- [ ] **Step 4: Run the full checks.**
- [ ] **Step 5: Commit** with `git commit -m "feat(beer): beer page with my rating and average"`.

**PR D:** "Closes kromby/BeerRater#15, #16, #19".

---

## PR E — BR-30, BR-31: chat (Hress.Org)

### Task 9: Chat backend

**Files:**
- Create: `api/Ez.Hress.BeerRater/Entities/ChatMessage.cs`, `UseCases/IBeerChatDataAccess.cs`, `UseCases/ChatInteractor.cs`, `DataAccess/BeerChatTableAccess.cs`, `DataAccess/ChatMessageTableEntity.cs`, `api/Ez.Hress.FunctionsApi/Beer/BeerChatFunctions.cs`
- Modify: `UseCases/SummaryInteractor.cs` (add an `IBeerChatDataAccess` dependency for `MessageCount`) and `Program.cs`
- Test: `api/Ez.Hress.UnitTest/BeerRater/ChatInteractorTests.cs`, `SummaryInteractorTests.cs`

**Interfaces:**
- Consumes: `SeasonInteractor.GetOpenSeasonAsync` and `BeerInteractor.GetSeasonBeerAsync`.
- Produces:
  - `ChatMessage { string ID; string SeasonID; string BeerID; int UserID; string? Text; int? ImageID; DateTime Created; DateTime? Edited; DateTime? Deleted; }`
  - `IBeerChatDataAccess`:
    - `Task<IList<ChatMessage>> GetMessagesAsync(string seasonID, string beerID)`
    - `Task<IList<ChatMessage>> GetSeasonMessagesAsync(string seasonID)`
    - `Task<ChatMessage?> GetMessageAsync(string seasonID, string beerID, string messageID)`
    - `Task SaveMessageAsync(ChatMessage message)` (upsert)
  - `ChatInteractor(IBeerChatDataAccess, SeasonInteractor, BeerInteractor, ILogger<ChatInteractor>)`:
    - `GetMessagesAsync(seasonID, beerID)`: excludes deleted messages, oldest first
    - `PostMessageAsync(seasonID, beerID, userID, string? text, int? imageID)`
    - `EditMessageAsync(seasonID, beerID, messageID, userID, string text)`
    - `DeleteMessageAsync(seasonID, beerID, messageID, userID)`: a soft delete that sets `Deleted`
  - `SummaryInteractor` now fills `MessageCount`, excluding deleted messages
  - HTTP:

    | Function | Method and route | Body | Returns |
    |---|---|---|---|
    | `beerMessages` | GET `beer/seasons/{s}/beers/{b}/messages` | | `ChatMessage[]` |
    | `beerMessages` | POST `beer/seasons/{s}/beers/{b}/messages` | `{text?, imageID?}` | 201 `ChatMessage` |
    | `beerMessage` | PUT `…/messages/{m}` | `{text}` | `ChatMessage` |
    | `beerMessage` | DELETE `…/messages/{m}` | | 204 |

- [ ] **Step 1: Write the failing tests:**

```csharp
[Fact] Post_TextOnly_Saves()                 // Text trimmed; ImageID null; Created.Kind == Utc
[Fact] Post_ImageOnly_Saves()
[Fact] Post_NeitherTextNorImage_ThrowsArgument()   // text "   ", imageID null
[Fact] Post_TextOver2000_ThrowsArgument()
[Fact] Post_ClosedSeason_ThrowsInvalidOperation()
[Fact] Post_BeerNotInSeason_ThrowsKeyNotFound()
[Fact] Post_IDsSortChronologically()         // string.CompareOrdinal(first.ID, second.ID) < 0
[Fact] Get_ExcludesDeleted_OldestFirst()
[Fact] Edit_Own_UpdatesTextAndEdited()
[Fact] Edit_OthersMessage_ThrowsUnauthorizedAccess()   // SaveMessageAsync never called
[Fact] Edit_DeletedMessage_ThrowsKeyNotFound()
[Fact] Delete_Own_SetsDeleted()
[Fact] Delete_OthersMessage_ThrowsUnauthorizedAccess()
// SummaryInteractorTests (add)
[Fact] Summaries_MessageCount_ExcludesDeleted()
```

- [ ] **Step 2: Run the tests and confirm they fail.**
- [ ] **Step 3: Implement.** The message ID is `$"{created:yyyyMMddHHmmssfff}-{Guid.NewGuid():N}"[..27]`. The `BeerChatMessages` table has PK `$"{seasonID}_{beerID}"` and RK the message ID. `GetSeasonMessagesAsync` uses a partition range query, because season IDs never contain `_` and `` ` `` is the character after `_`:

```csharp
_tableClient.QueryAsync<ChatMessageTableEntity>(e =>
    e.PartitionKey.CompareTo(seasonID + "_") >= 0 && e.PartitionKey.CompareTo(seasonID + "`") < 0);
```

- [ ] **Step 4: Run the tests and confirm they pass.** Smoke test: editing another user's message → `403`.
- [ ] **Step 5: Commit** with `git commit -m "feat(beer): chat messages per beer"`.

### Task 10: Chat on the beer page

**Files:**
- Create: `src/pages/beer/components/ChatThread.tsx`, `src/pages/beer/components/ChatComposer.tsx`, `src/pages/beer/resizeImage.ts`
- Modify:
  - `src/pages/beer/hress.ts`: add the user and photo helpers below
  - `types.ts`: add `ChatMessage`
  - `hooks/useBeerApi.ts`: add `useMessages`, `usePostMessage`, `useEditMessage` and `useDeleteMessage`
  - `beer.tsx`: render `ChatThread`
- Test: `src/pages/beer/components/ChatThread.test.tsx`, `src/pages/beer/resizeImage.test.ts`

**Interfaces:**
- Consumes: the Task 9 HTTP API.
- Produces:
  - `ChatMessage { id; seasonID; beerID; userID; text?; imageID?; created; edited? }`
  - `hress.ts`:
    - `useHressUser(userID: number)`: wraps `useUserByHref(`/api/users/${userID}`)` from `src/hooks/useSingleUser`
    - `useHressPhotoUpload(): (file: File) => Promise<number>`: wraps `useImages().uploadImage({ file, name: file.name, container: ImageContainer.Beer })` and returns its `id`
    - `hressImageUrl(imageID: number, width: number): string`: returns `${config.get("apiPath")}/api/images/${imageID}/content?width=${width}`
  - `fitWithin(width: number, height: number, max: number): { width: number; height: number }`
  - `resizeImage(file: File, max = 1600, quality = 0.8): Promise<File>`, which outputs a JPEG and returns the original file when it already fits

- [ ] **Step 1: Write the failing tests:**

```ts
// resizeImage.test.ts
expect(fitWithin(4000, 3000, 1600)).toEqual({ width: 1600, height: 1200 });
expect(fitWithin(3000, 4000, 1600)).toEqual({ width: 1200, height: 1600 });
expect(fitWithin(800, 600, 1600)).toEqual({ width: 800, height: 600 });
```

```tsx
// ChatThread.test.tsx
it("renders messages oldest first with author name, date and photo")  // img src contains "/api/images/55/content?width="
it("renders HTML in message text as literal text")  // "<img src=x onerror=alert(1)>" found by getByText; no img[src='x']
it("posts a text message and clears the box")       // POST body {text:"Góður"}
it("uploads a photo first, then posts its imageID") // useHressPhotoUpload mocked → 55; POST body {imageID:55}
it("send is disabled when text is empty and no photo is chosen")
it("shows edit/delete only on my own messages")     // userID 7: own message has data-testid="edit-message"; others don't
it("hides the composer when the season is closed")
```

- [ ] **Step 2: Run the tests and confirm they fail.**
- [ ] **Step 3: Implement.**
  - Render text only as React text children: no `dangerouslySetInnerHTML`. Keep line breaks with `white-space: pre-wrap`.
  - Delete asks for confirmation with `window.confirm`.
  - Resize the photo with `resizeImage` before upload.
  - The file input uses `accept="image/*"` so phones offer the camera.
  - The list's rows show `messageCount`.
- [ ] **Step 4: Run the full checks.**
- [ ] **Step 5: Commit** with `git commit -m "feat(beer): chat with photos on the beer page"`.

**PR E:** "Closes kromby/BeerRater#20, #21".

---

## PR F — BR-41: season leaderboard (Hress.Org)

### Task 11: Ranking and the leaderboard page

**Files:**
- Create: `api/Ez.Hress.BeerRater/UseCases/Leaderboard.cs`, `src/pages/beer/leaderboard.tsx`
- Modify: `UseCases/SummaryInteractor.cs` (apply `Leaderboard.Rank`), `App.js` (the `leaderboard` route) and `index.tsx` (a link to the leaderboard)
- Test: `api/Ez.Hress.UnitTest/BeerRater/LeaderboardTests.cs`, `SummaryInteractorTests.cs`, `src/pages/beer/leaderboard.test.tsx`

**Interfaces:**
- Produces:
  - `public static class Leaderboard { public const int MinimumRatings = 3; public static IList<BeerSummary> Rank(IEnumerable<BeerSummary> summaries); }`
  - `GetSeasonSummariesAsync` now returns ranked order, and `GetBeerSummaryAsync.Rank` is taken from the season ranking. The beer list (`index.tsx`) keeps its own name sort.

- [ ] **Step 1: Write the failing tests:**

```csharp
// LeaderboardTests (pure)
[Fact] Rank_ByAverageThenCount()    // A 4.4/8, B 3.8/6, C 3.8/9 → order A,C,B ranks 1,2,3
[Fact] Rank_EqualAverageAndCount_ShareRank()   // A 4.0/3, B 4.0/3, C 3.0/5 → ranks 1,1,3
[Fact] Rank_FewerThan3Ratings_UnrankedAfterRanked_ByName()  // D 5.0/2 → Rank null, after ranked; unranked sorted by Beer.Name
[Fact] Rank_NoRatings_Unranked()
// SummaryInteractorTests (add)
[Fact] BeerSummary_RankMatchesSeasonRanking()
```

```tsx
// leaderboard.test.tsx
it("shows ranked beers in backend order with rank, average, count, messages and my rating")
it("lists unranked beers (rank null) in a separate section below")   // data-testid="unranked"
it("shows shared ranks as the same number")                         // ranks [1,1,3]
it("sorts the ranked section by rating count, alcohol % or country when chosen")  // unranked keep name order
it("links each row to /beer/:beerID")
```

- [ ] **Step 2: Run the tests and confirm they fail.**
- [ ] **Step 3: Implement.** The default sort is rank. The other sorts are client-side and descending, except country, which is ascending.
- [ ] **Step 4: Run the full checks.**
- [ ] **Step 5: Commit** with `git commit -m "feat(beer): season leaderboard"`.

**PR F:** "Closes kromby/BeerRater#27".

---

## PR G — Launch runbook (BeerRater repo)

### Task 12: Launch runbook

**Files:**
- Create: `docs/launch-runbook.md`

- [ ] **Step 1: Write the runbook.** These are kromby's manual steps, in order. Give exact names and values.
  1. **Blob container.** Create the blob container `beer` in the production storage account. Uploads do not create it, so a missing container makes every beer photo upload fail.
  2. **Season row.** Create a `BeerSeasons` row: `PartitionKey=season`, `RowKey=jolabjor-2026`, `Name=Jólabjór 2026`, `TypeID` left empty (the `gen_Type` ID once season types exist), `StartDate` set, and no `EndDate`. If the API hasn't run yet, create the tables first in Storage Explorer.
  3. **Beer list.** Load the beers with Storage Explorer's CSV import. Use a CSV template for `Beers` with the header `PartitionKey,RowKey,Name,Producer,AlcoholPercentage,AlcoholPercentage@type,Country,VinbudinUrl,RegisteredBy,RegisteredBy@type,RegisteredAt,RegisteredAt@type`, plus one example row (`beer,<32-hex guid>,Tuborg Lille Jul,Carlsberg,5.6,Double,Danmörk,,<kromby's user ID>,Int32,2026-11-20T00:00:00Z,DateTime`). Also give the matching `BeerSeasonBeers` CSV (`PartitionKey,RowKey` = season ID, beer ID).
  4. **Check before launch.** Open `https://www.hress.org/beer` while logged in.
  5. **Menu entry.** Add the `/beer` menu entry the same way existing menu rows (served by `/api/menus`) were added. This completes BR-01; close kromby/BeerRater#1.
  6. **Ending the season.** To end the season later, set `EndDate` on the `BeerSeasons` row.
- [ ] **Step 2: Commit** with `git commit -m "docs: BeerRater launch runbook"`.

**PR G:** "Closes kromby/BeerRater#1".
