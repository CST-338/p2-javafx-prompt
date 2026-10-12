# Project 2: The Ten Projects

Your team **picks one of these**. You do not invent your own. Each one is built so that its
external API is load-bearing in exactly one slice, rather than bolted on at the end.

In every project **Slice 1 is Accounts**: the `users` table with `userId`, `username`,
`password` and `role`. A team of four builds four of the five slices, a team of three builds
three. Whatever you cut goes on your WILL NOT DO list.

Format: **core** is what the slice minimally is. *Beyond* items are stretch goals, eligible for
extra credit, and the things you choose not to do belong on the WILL NOT DO list.

## The APIs

All of these were tested and working in October 2026. None costs money.

| Project | API | Key needed? |
|---|---|---|
| A. Trivia / Study Guide | Open Trivia DB, `opentdb.com/api.php` | none |
| B. Recipe Box | TheMealDB, `themealdb.com/api/json/v1/1/` | published test key `1` |
| C. Collection Tour Planner | Met Museum, `collectionapi.metmuseum.org` | none |
| D. Field Station Log | USGS Earthquakes + Open-Meteo | none |
| E. Watchlist | TVmaze, `api.tvmaze.com` | none |
| F. Card Table | Deck of Cards, `deckofcardsapi.com` | none |
| G. Monster Arena | PokeAPI, `pokeapi.co/api/v2/` | none |
| H. Reading Log | Open Library, `openlibrary.org` | none |
| I. Sky Log | NASA, `api.nasa.gov` | **free key, register it** |
| J. Gallery Tour | Art Institute of Chicago, `api.artic.edu/api/v1/` | none |

### Two things to know before you choose

**Respect the rate limits.** Open Trivia DB asks for one request every five seconds.
TheMealDB's key `1` is a shared development key. Everyone expects a descriptive `User-Agent`,
which the starter already sends. If you are choosing **NASA**, register your own free key;
`DEMO_KEY` allows about 30 requests an hour per IP address, and on campus wifi the whole class
shares that.

**Two endpoints changed recently**, which is a useful lesson in itself: the Met Museum retired
its `/v1/search` endpoint on 1 October 2026 (use `/v1.1/search`, which pages with `offset` and
`limit`), and REST Countries has deprecated every version of its API. Any tutorial older than a
few weeks will have both of these wrong. Check the API's own documentation, not a blog post.

---

## The projects

### Option A: Trivia / Study Guide (Open Trivia DB)

1. **Accounts** *(required)*: `users`; login and registration. *Beyond:* password reset, roles.
2. **Question Bank + API Import**: `questions`, `categories`; admin CRUD **plus import from
   Open Trivia DB** with category and difficulty mapped onto local rows. *Beyond:* session
   tokens to avoid repeats, bulk import with a progress indicator, de-duplication.
3. **Quiz Engine**: `attempts`; quiz and result scenes, scoring. *Beyond:* timed quizzes,
   randomized order, hints.
4. **Leaderboard & History**: personal history and a global board; aggregation queries.
   *Beyond:* per-category boards, streaks, badges.
5. **Study Mode & Tagging**: flashcard mode, tag and favorite questions. *Beyond:* spaced
   repetition, shareable study sets.

### Option B: Recipe Box (TheMealDB)

1. **Accounts** *(required)*.
2. **Recipe Search & Import**: `recipes`, `ingredients`; **search TheMealDB and import** a meal
   with its ingredient list into local tables. *Beyond:* import by category or area, image
   caching, random-meal discovery.
3. **Meal Planner**: `plan_entries`; assign recipes to days, week view. *Beyond:* servings
   scaling, copy last week, drag to reschedule.
4. **Shopping List**: aggregate ingredients across a planned week, deduplicate and combine
   quantities. *Beyond:* pantry stock tracking, export to text, grouping by aisle.
5. **Ratings & Notes**: rate and annotate recipes you have cooked. *Beyond:* photos,
   substitution history, "cook again" suggestions.

### Option C: Collection Tour Planner (Met Museum)

1. **Accounts** *(required)*.
2. **Artwork Search & Import**: `artworks`, `artists`; **`v1.1/search` then `v1/objects/{id}`**
   to import records. *Beyond:* department filters, `hasImages`, paging past the first page.
3. **Tour Builder**: `tours`, `tour_items`; build an ordered tour from saved artworks.
   *Beyond:* reordering, time estimates, printable itinerary.
4. **Visit Log & Notes**: record visits, annotate pieces you saw. *Beyond:* photos, a map,
   visit statistics.
5. **Tags & Discovery**: tag artworks, browse by tag, "more like this". *Beyond:* tag
   suggestions from object metadata, shared tag vocabularies.

### Option D: Field Station Log (USGS Earthquakes + Open-Meteo)

1. **Accounts** *(required)*.
2. **Station Registry**: `stations`; name a place, **resolve it with Open-Meteo geocoding**,
   store the coordinates. *Beyond:* bulk import, a map view, elevation.
3. **Event Feed Import**: `events`; **pull the USGS GeoJSON feed** and store quakes near each
   station, filtered by magnitude and radius. *Beyond:* scheduled refresh, deduplication
   against existing rows, historical backfill.
4. **Observations Log**: `observations`; human notes attached to a station or an event,
   **with the Open-Meteo conditions at that time**. *Beyond:* attachments, templates, export.
5. **Alerts & Thresholds**: per-station magnitude thresholds that raise an in-app alert.
   *Beyond:* notification history, quiet hours, summary digests.

### Option E: Watchlist (TVmaze)

1. **Accounts** *(required)*.
2. **Show Search & Import**: `shows`, `episodes`; **search TVmaze and import** a show with its
   episode list. *Beyond:* cast import, images, refresh on demand.
3. **Watchlist & Progress**: `watch_entries`; mark episodes watched, resume point, progress
   bars. *Beyond:* bulk mark-season, next-up queue, time-watched statistics.
4. **Ratings & Reviews**: rate shows and episodes, averages. *Beyond:* spoiler-tagged reviews,
   recommendation by rating overlap.
5. **Schedule & Calendar**: upcoming episodes for shows on your list. *Beyond:* a month view,
   reminders, timezone handling.

### Option F: Card Table (Deck of Cards)

Chosen deliberately as the option closest to Project 1, for teams that want familiar ground.

1. **Accounts** *(required)*.
2. **Game Engine**: `games`, `hands`; **draw from the Deck of Cards API**, implement one game's
   rules. *Beyond:* a second game, shoe and reshuffle handling, dealer strategy.
3. **Hand History**: `rounds`; persist every hand with its outcome, replayable. *Beyond:*
   replay animation, filtering, export.
4. **Bankroll & Statistics**: track wins, losses and streaks with aggregation queries.
   *Beyond:* charts, session summaries, variance.
5. **Rules Variants**: `rulesets`; configurable rules selected at table creation. *Beyond:*
   per-ruleset statistics, custom rule editor.

### Option G: Monster Arena, legally distinct (PokéAPI)

Kept from the existing catalog. PokéAPI confirmed working, so this option is live.

1. **Accounts** *(required)*.
2. **Creature Roster + API Import**: `creatures`, `types`; **import species, base stats and
   types from PokéAPI** into local tables, then collect and train. *Beyond:* evolution chains,
   sprites, moves, leveling curves.
3. **Battle Engine**: `battles`; battle scene and combat logic driven by the imported stats.
   *Beyond:* the type-effectiveness matrix (also importable), status effects, animations.
4. **Admin & Arenas**: `arenas`; creature and arena admin CRUD. *Beyond:* tournaments,
   seasonal arenas.
5. **Marketplace & Trading**: trade and buy creatures, currency and inventory. *Beyond:*
   auctions, trade history.

PokéAPI is the best teaching API in this list: deeply nested JSON with referenced sub-resources,
so a naive "one call, one object" import does not work. Students have to model relationships and
decide what to fetch eagerly. That is the lesson.

### Option H: Reading Log (Open Library)

1. **Accounts** *(required)*.
2. **Book Search & Import**: `books`, `authors`; **search Open Library and import** by title or
   ISBN. *Beyond:* cover images, edition handling, author detail import.
3. **Shelves & Reading Status**: `shelves`, `shelf_items`; want-to-read, reading, finished.
   *Beyond:* custom shelves, reading goals, progress by page.
4. **Reviews & Ratings**: rate and review finished books, averages. *Beyond:* spoiler tags,
   re-read history, quotes.
5. **Statistics & Discovery**: books per month, pages read, by subject. *Beyond:* charts,
   recommendations by subject overlap.

### Option I: Sky Log (NASA)

Requires a free API key, see Section 2.

1. **Accounts** *(required)*.
2. **APOD Archive Import**: `apod_entries`; **import Astronomy Picture of the Day** by date
   range. *Beyond:* image caching, full-text search of explanations, favorites.
3. **Near-Earth Objects**: `neos`, `approaches`; **import the NeoWs feed**, store close
   approaches with distance and velocity. *Beyond:* hazard filtering, a timeline view.
4. **Observation Log**: personal observing notes tied to a date and an object. *Beyond:*
   equipment records, seeing conditions, attachments.
5. **Alerts & Digest**: thresholds on approach distance or size raising an in-app alert.
   *Beyond:* digests, quiet hours, history.

### Option J: Gallery Tour (Art Institute of Chicago)

A drop-in alternative to Option C for teams that want a different collection. Same five slices
as Option C, with `api.artic.edu/api/v1/artworks` in place of the Met endpoints. Its `fields`
parameter lets students request exactly the columns their table needs, which is a cleaner
lesson in not over-fetching than the Met's fixed object payload.

---

---

## Picking well

Some honest guidance.

- **G, Monster Arena, has the most interesting API.** PokeAPI returns deeply nested JSON with
  references to other resources, so a naive "one call, one object" import does not work. You
  have to model relationships and decide what to fetch eagerly. That is the most realistic API
  work on this list, and the most work.
- **F, Card Table, is the closest to Project 1.** If your team wants familiar ground, take it.
  The API is trivially simple; the interest is in the game logic and the hand history.
- **A, Trivia, has the cleanest import.** Flat JSON, obvious mapping to a table, and a natural
  reason to cache (the rate limit). It is the safest choice and nobody will think less of it.
- **D, Field Station Log, is the only one using two APIs**, which satisfies a Tier B
  enhancement on its own. It is also the only one where the data keeps changing underneath you.

Whatever you pick, the grade is on your slice and your process, not on the ambition of the app.
