# JavaFX Application (Project 2)

> **CST 338 Software Design, CSU Monterey Bay.**
> The ten projects and their slice breakdowns are in
> [`PROJECT_CATALOG.md`](PROJECT_CATALOG.md). Your team's starter repository is handed to you
> in Part 0; you do not build the project from scratch.

You will work as a team of **3 to 4 students** to build a JavaFX desktop application, but you
are graded as an **individual**. Each of you owns one complete **feature slice** of the app end
to end, and your grade is driven mostly by *how you work*: your GitHub contributions, your code
reviews, and your teamwork, with concrete code milestones along the way.

The project is **5 assignments** (Part 0 through Part 4). 
Part 4 is about half the total points, because that is where your finished work comes together. 
Two further assignments sit alongside the project: 
  - a **Presentation** (5 points, graded individually) 
  - a **Teammate Evaluation** (10 points).

---

## How Grading Works

Each assignment is graded **per student** against its own rubric. 
Your score reflects your own slice and your own process. 
A teammate's shortfall does not lower your grade, and you cannot inherit their points.

### What the project measures

| Area | What it measures |
| :---- | :---- |
| **Feature slice** | Your scenes work, CRUD runs through your DAO, your logic is correct, your enhancements are applied |
| **Testing** | Your unit and UI tests, a failure-path test, plus AI-drafted-then-curated tests with a reflection |
| **GitHub contributions** | `username/feature` branches, **3+ merged Pull Requests**, 3+ issues closed by linked PRs |
| **Code review** | Substantive reviews of others' PRs, one AI code review you adjudicate, one cross-team review |
| **Teamwork / integration** | The assembled app builds and runs, the scene factory wires all slices, your README and issues stay current |

Every area is measured in every assignment that touches it, and the weight shifts toward the
later parts as there is more finished work to measure.

**See Canvas for the point value of each assignment and for every rubric.** Canvas is the
authority on scoring. This document is the authority on what the work is.

### Extra credit, up to +10

Extra credit comes from a **fixed list**, not from your own scope declaration:

| | Points |
| :---- | :----: |
| An extra slice from your project's catalog, meeting the full Milestone 2 slice standard | up to +6 |
| A *beyond* item from your project's published catalog | +2 each |
| An enhancement beyond the one or two you are required to choose | +2 each |
| An additional documented AI experiment | +2 |

**Extra credit applies only when your slice itself is finished.** Every Feature slice
criterion on the Part 4 rubric must be scored Excellent, or at most one of them Proficient.
Extra credit is for work on top of a finished slice, never instead of one. It is credited to the
individual who did the work and applied on Part 4.

---

## Starter Repository

You do **not** build the project from scratch. A starter repository is provided to each team,
already carrying:

- A working Gradle build, pinned, with JavaFX resolved on macOS, Windows and Windows on ARM
- H2, JUnit 5 and TestFX configured and running
- A `SceneFactory` with a registration method, and a main menu
- `Database.init()` which runs `schema.sql` on startup
- A `Session` object holding the current user
- A **development login bypass** that seeds one user per role and skips the login screen

The bypass matters: it means **no slice waits for the Accounts slice to exist.** Everyone codes
against `Session.currentUser()` from day one. The student who owns Accounts removes the bypass
as part of finishing that slice.

Do not restructure the starter. Build your slice inside it. This is a valuable exercise in shoehorning new code into an existing structure.

---

## The Vertical Feature Slice

Every student owns **one feature, built top to bottom**. Not a horizontal layer, not "the UI
person" and "the database person". A slice is a complete deliverable that can be graded on its
own:

| Your slice includes | Requirement |
| :---- | :---- |
| **Database** | 1 table (plus its relationships) merged into the shared `schema.sql` |
| **Data access** | 1 DAO with CRUD **and** at least one query or filter method |
| **UI** | 1 to 2 FXML scenes with controllers, reachable from the main menu |
| **Logic** | 1 or more domain/service classes holding the feature's rules |
| **Tests** | unit tests (DAO and logic), **a failure-path test**, and at least 1 TestFX UI test |
| **Enhancements** | one Tier B item, **or** two Tier A items (see the menu) |
| **Seed data** | fixtures for the tables your slice *reads*, so your slice demos and tests with the upstream slices absent |
| **Design artifacts** | 2 use cases (main and alternate flow), your ERD fragment, a mockup of your scene |
| **GitHub trail** | 3+ SMART issues, `username/feature` branches, 3+ reviewed and merged PRs |

**CRUD, or a justified subset.** For event-log tables such as `attempts`, `orders`, `battles` or
`approaches`, update and delete may be meaningless. Omitting them is fine; state the omission and
the reason in the DAO's Javadoc. Do not invent a fake delete to satisfy the word "CRUD".

### Seed data is what makes your slice gradeable alone

If your slice reads a table another student owns, you ship fixtures for it. The Leaderboard owner
ships fake attempts; the Cart owner ships fake items. This is not busywork, it is the rule that
lets your slice be demonstrated and tested whether or not your teammates finished.

### Schema first

Your table's DDL is merged into `schema.sql` during Part 1, as its own small reviewed Pull
Request. The ERD stops being a drawing and becomes a contract the whole team has agreed to.

### Choosing your scope

Each project offers **5 candidate slices**, and a team builds one per member:

- **4 students:** build 4 of the 5 (Accounts required); 1 slice is cut
- **3 students:** build 3 of the 5 (Accounts required); 2 slices are cut

Each slice also lists **beyond** stretch items. Depth is a dial, not a switch.

### Scope declaration and the WILL NOT DO list

During Part 1 each student opens three labeled GitHub Issues:

- `scope:core` what your slice will definitely do
- `scope:will-do` the beyond items you are committing to
- `scope:will-not-do` what you are deliberately not building

The team's cut slices go on the README's **WILL NOT DO list**.

At Part 4 you are graded on **scope accuracy**, not on ambition: shipped scope matches your Part
1 declaration, *or* every change is an edit on the issue with a dated one-line reason ("cut timed
quizzes Nov 20, the import took longer than planned"). 
*Changing your mind is normal engineering*; Changing it silently is not. 

**The change log is what is graded.**

---

## Choose Your Project

Teams **select one of the projects below**. _You do not invent your own._(sorry) 
Each is built so that its external API is load-bearing in exactly one slice.

The full slice breakdowns, with core and beyond items for all five slices of each project, are in
[`PROJECT_CATALOG.md`](PROJECT_CATALOG.md). In every project **Slice 1 is Accounts**
(the `users` table with `userId`, `username`, `password`, `role`).

| | Project | API it consumes |
| :---- | :---- | :---- |
| **A** | Trivia / Study Guide | Open Trivia DB |
| **B** | Recipe Box | TheMealDB |
| **C** | Collection Tour Planner | Met Museum |
| **D** | Field Station Log | USGS Earthquakes + Open-Meteo |
| **E** | Watchlist | TVmaze |
| **F** | Card Table | Deck of Cards |
| **G** | Monster Arena (legally distinct) | PokéAPI |
| **H** | Reading Log | Open Library |
| **I** | Sky Log | NASA |
| **J** | Gallery Tour | Art Institute of Chicago |

All of these were tested and working as of October 2026. None requires payment. Only NASA needs
a key, and it is free and takes a minute to register; do not rely on `DEMO_KEY`, because the
whole class shares its rate limit on campus wifi.

---

## Consuming the API

Every project consumes an external REST API. This is a requirement, not an enhancement, and it
has to be load-bearing. Four rules define what counts:

1. **The API feeds the database.** At least one API response is parsed into domain objects and
   **persisted through a DAO**. A call whose result is printed and discarded does not count.
2. **The app works offline after import.** The local database is the system of record; the API is
   an import source. Cache into your own tables rather than calling the API on every scene load.
   This is also why your Part 4 video will work on any network.
3. **Failures are handled.** A timeout, a non-200 response, and a malformed body each produce a
   **message the user can read**, not a stack trace. **At least one test covers a failure path.**
   This is the single most-graded part of the API requirement, and it is the difference between a
   demo and software.
4. **No hard-coded endpoints or keys in a scene.** Base URL, key and timeouts live in
   configuration, the same no-magic-values rule this course applies everywhere else.

Respect published rate limits. Open Trivia DB asks for one request per five seconds.
TheMealDB and TheCocktailDB key `1` is a shared development key. Everyone expects a descriptive
`User-Agent`.

---

## The Enhancement Menu

The slice is the required floor. On top of it, **choose one Tier B item, or two Tier A items.**

**An enhancement may never be something your slice's core already requires.** If your slice is a
browse-and-filter list, then a `TableView` is your core, and you need a different enhancement.

### Tier A, choose two

| | Item |
| :---- | :---- |
| A1 | Data binding with `ObservableList` or `Property`, the UI updating *from* the model |
| A2 | Notifications or alerts raised by an application event, not by a button press |
| A3 | An advanced query or report: aggregation, grouping, or a multi-table join |
| A4 | A `TableView` with column sorting, a live filter and paging (only if a list is not your core) |
| A5 | Response caching with a TTL: store `fetched_at`, re-fetch only when stale, show the user how old the data is |
| A6 | Externalized configuration: base URL, key and timeouts in a properties file or environment |
| A7 | Search as you type, debounced so typing does not fire one request per keystroke |
| A8 | Export your slice's tables to CSV or JSON, re-importable |
| A9 | CSS theming with a light and dark pair, colors defined once |
| A10 | A JavaFX chart driven by a real aggregation query |

### Tier B, choose one

| | Item |
| :---- | :---- |
| B1 | Background import off the JavaFX Application Thread (`Task`, `Platform.runLater`), UI responsive, with progress and a working cancel |
| B2 | Retry with exponential backoff, distinguishing retryable (timeout, 429, 5xx) from permanent (400, 404), with a capped attempt count |
| B3 | Client-side rate limiting: a request governor that respects the API's published limit |
| B4 | Pagination against a paged API, resumable, without loading everything into memory |
| B5 | Offline mode: detect no network, serve from the local database, mark the data stale, resync when it returns |
| B6 | A stubbed API in tests, so the import path is tested with **no network**, including a malformed body and a non-200 |
| B7 | A DTO and mapping layer so the API's shape never reaches your tables |
| B8 | Asynchronous image loading with a disk cache, placeholder while loading, bounded memory |
| B9 | Transactional bulk import: a mid-import failure leaves the database untouched |
| B10 | A second external API, distinct from your project's required one, feeding a different table |
| B11 | An external library used meaningfully (Jackson, Gson, OkHttp, a charting library) |
| B12 | A Supabase backend, with local persistence still required |
| B13 | A reusable custom FXML component via `fx:include`, used in 2+ scenes, with its own controller |

B6 is worth a look: it makes the required failure-path test straightforward instead of awkward.

---

## Testing

Testing is a graded area in its own right, worth 15 points.

- **Unit tests** covering your DAO (insert, read, update, delete as applicable) against an
  in-memory H2 database, and your domain logic, including at least one **negative or edge case**.
- **A failure-path test** for your API import: what happens on a non-200, a timeout, or a body
  that will not parse.
- **A UI test** using TestFX verifying at least one scene transition in your slice.
- **AI-assisted tests**, curated. See below.

Your tests must **run on screen** in the Part 4 video.

---

## Using AI (required and documented)

AI use here is **required and transparent**. You are graded on your **judgment**, meaning how you
curate and adjudicate AI output, not on the raw output.

### AI-drafted, then curated, tests

Use an AI tool to *draft* tests for your slice, then make them yours. In `TESTING.md`, record:

1. The **prompt(s)** you gave it.
2. What it **produced** (snippet or commit link).
3. **Either** one test you rewrote or rejected, with the before and after, **or**, if you kept
   everything, a one-line statement per test naming the behavior that test pins down.
4. **Always: one test the AI did not write**, which you added because you found a coverage gap
   it missed.

Point 4 is the one that matters most. Spotting what is *missing* is the harder skill and the
one that does not go away as models improve.

### AI as a second code reviewer

For at least one of your own Pull Requests, run an **AI code review** *in addition to* the
required human review, and record in the PR:

1. The **tool** used and which PR.
2. The AI's **findings**.
3. Your **adjudication**: each finding accepted (with the fixing commit) or rejected (with your
   reasoning).

Run it on a **substantial** PR, your Milestone 2 PR or one of comparable size. "The AI found
nothing" on a two-line diff is not an adjudication.

The human reviewer catches intent and design, the AI catches mechanical and coverage gaps, and
**you** decide. That decision is what is graded.

> **Integrity:** AI use here must be disclosed and documented. Undocumented AI use elsewhere
> still follows course policy.

---

## Code Review

### What counts as a substantive review

A review counts when it has **both**:

- at least **two diff-anchored comments** (attached to lines, not typed into the PR body), and
- at least **one comment that names a concrete problem** (a bug, a missing test, a design issue,
  an unclear name) **and proposes a specific change**.

A bare "LGTM" is not a review and simply does not count.

**Whether the author acted on your review is not your problem.** A good review that the author
ignores still counts fully for you. The author's silence is scored against the *author*, under
Part 4's review responsiveness criterion.

### The requirement

- **At least 3 substantive reviews** across the project. Your Part 2 review counts toward these.
- **At least one** of your counted reviews **led to a change before merge** (a commit after your
  comment) **or** to a reasoned decline from the author.
- **Cross-team PR reviews count toward the three.** Repositories are public and GitHub accepts
  review comments from outside the team. You will be paired with another team in your own
  section at the Part 2 deadline.
- **One AI review** of your own PR, adjudicated.
- **One cross-team application review** in Part 3.

### If nobody reviews your PR

**Every merge to `main` needs a review.** 
If no teammate reviews within **48 hours**, request a review from your cross-team buddy or from the TA. 
**An unreviewed merge is not held against you when the request is visible on the PR.** You are not required to sit blocked.

### Your review index

For Part 3, submit links to the **three best reviews you wrote**, each with one line: *what I caught*.
Those three are what get graded.

---

## GitHub & Project Tracking

### Branches and Pull Requests

- Branches follow **`<github-username>/feature-name`**, for example `alice-dev/login-scene`.
  Use your GitHub username exactly, because that is what contribution verification keys on.
  Other patterns will not count.
- Every merge to `main` goes through a **Pull Request** that references its issue with
  **`Closes #N`** and is **reviewed** before merging.
- The GitHub measure is **3 or more merged Pull Requests**, not a commit count. Commit however
  often suits the work.

### GitHub Issues are your live tracker

Every slice task, enhancement and scope decision is an Issue, assigned to its owner, labeled
(`slice-1`, `testing`, `enhancement`, `scope:core`, `scope:will-do`, `scope:will-not-do`), and
closed by a PR via `Closes #N`. Write **SMART** issues.

> **Good:** "Create `UserDao` and the `User` entity with insert, update, delete and
> query-by-username; add JUnit tests using in-memory H2. Due end of Sprint 1."
> **Bad:** "Make a database table."

### The repo README is your dashboard

Copy the provided `README.md` template into your repo root and keep it current. It is the first
thing read when grading. It records: the project chosen, the **slice-ownership table**, the
**WILL NOT DO list**, links to Issues and PRs with reviewers, enhancements chosen, AI review
links, the extra-credit log, and build and run instructions. Where the README and the Issues
disagree, **the Issues win**.

---

## The Seven Assignments

| Part | Points | What you do |
| :---- | :----: | :---- |
| **0: Team & Slices** | 10 | Form the team; accept the starter repo and fill in the README; choose your project; assign one slice per member; draft the WILL NOT DO list. |
| **1: The Plan** | 10 | Write your slice spec (2 use cases, ERD fragment, mockup); **merge your table's DDL into `schema.sql`**; open your 3 scope issues plus 3+ SMART task issues; choose your enhancements; write **your slice's** test plan. |
| **2: Data & Scenes** | 15 | **Milestone 1:** merge your slice skeleton (table, DAO CRUD, one scene reachable from the main menu); write your first passing test; give your first substantive PR review. |
| **3: Reviews** | 15 | Submit your review index (3 best reviews); run and adjudicate one AI code review; complete one cross-team application review; **your slice runs in the Part 3 build**. |
| **4: Final Application** | 50 | **Milestone 2:** finish your slice and enhancements; complete the test suite, AI-curated tests and reflection; integrate; record the team video. |
| **Presentation** | 5 | Present your own slice in class. Graded individually. |
| **Teammate Evaluation** | 10 | Evaluate your teammates. |

### About the Part 4 video

The team submits **one video**, 8 minutes or less. Each student narrates their own slice, and
the README lists the **timestamp range** for each student's segment. You are graded on your own
segment.

Your segment shows: your scenes working, your CRUD, your enhancement, a **relaunch demonstrating
persistence**, and your **tests running**.

---

## Helpful Links

- [JavaFX Documentation](https://openjfx.io/openjfx-docs/)
- [IntelliJ IDEA: JavaFX setup](https://www.jetbrains.com/help/idea/javafx.html)
- [H2 Database Engine](https://h2database.com)
- [SQLite JDBC Driver](https://github.com/xerial/sqlite-jdbc)
- [TestFX (JavaFX UI Testing)](https://github.com/TestFX/TestFX)
- [JavaFX concurrency: `Task` and `Service`](https://openjfx.io/javadoc/21/javafx.graphics/javafx/concurrent/package-summary.html)
- [gitignore.io](https://gitignore.io)
- [Writing SMART goals/issues](https://www.atlassian.com/blog/productivity/how-to-write-smart-goals)

---

## Version History

### Oct 2026 V3.0.0
- `14-Oct` Updated language.
- `14-Oct` Point tables replaced by a pointer to Canvas, so the document does not go stale
  when the rubric is re-weighted. Extra-credit gate restated against rubric ratings rather
  than a retired criterion name.
- **Every project now consumes an external REST API**, with four rules defining what counts.
  Moved out of the enhancement menu, where it had been one option among many.
- **Teams select one of ten published projects** rather than inventing their own. Each is built
  around an API verified working in October 2026.
- **A starter repository is provided**, with a Gradle build, `SceneFactory`, `Database.init()`,
  `Session`, and a **development login bypass** so no slice waits on the Accounts slice.
- **Seed-data fixtures required** for the tables your slice reads, so a slice can be graded,
  demonstrated and tested with the other slices absent.
- **Schema first:** your table's DDL is merged during Part 1 as its own reviewed PR.
- **GitHub measure changed from "15+ commits" to "3+ merged Pull Requests."** Commit counts
  break under squash merge and measured the wrong unit.
- **Enhancement menu restocked and tiered**: 23 items in two tiers, choose one Tier B or two
  Tier A. Nine items exist only because every app now has a network boundary.
- **"Substantive review" given an operational definition.** Dropped the request-changes count,
  which was collusion-friendly; replaced with "led to a change before merge". Cross-team PR
  reviews now count, and a 48-hour escape hatch means nobody is blocked by an absent reviewer.
- **Scope accuracy replaces the WILL NOT DO checkbox**, graded on the dated change log rather
  than on the list existing.
- **Extra credit moved to a fixed list** and gated behind a slice score of at least 16/20, so
  under-declaring scope no longer converts ordinary work into extra credit.
- **AI test reflection now requires one test the AI did not write.** Finding the gap is the skill.
- Presentation split out as its own individually-graded assignment.
- Per-student wording fixes: "your slice's test plan", CRUD-or-justified-subset, one video with
  per-student timestamp ranges.

### Jul 2026 V2.0.0
- **Individual-first redesign.** Replaced the open menu, points-target and penalty system with
  **vertical feature slices**, one per student, graded independently.
- 100 points across 5 assignments (10/10/15/15/50), measuring slice (30), testing (15),
  GitHub (20), code review (20), teamwork (15).
- Added 5 candidate slices per app, a required WILL NOT DO list, and beyond-scope items.
- **Testing featured** as its own graded area.
- **AI use required and documented.**
- Retired the project spreadsheet in favor of GitHub Issues plus a repo README dashboard.

### Mar 2026 V1.0.0
- Created the JavaFX version from the Android Application prompt (Android/Room to
  JavaFX/SQLite-H2; Activities/Intents to FXML Scenes/Scene Factory). *(See `__archive/` for the
  V1 menu/claiming version.)*
