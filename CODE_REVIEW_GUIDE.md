# Reviewing a Pull Request on GitHub

Follow the steps in order. When you finish, you have left a review that **counts**, and you can
link to it from your Part 3 review index.

**What counts** (from the prompt, "What counts as a substantive review"). Both of these:

- at least **two diff-anchored comments**: attached to lines in **Files changed**, not typed
  into the PR's conversation box, and
- at least **one comment that names a concrete problem** (a bug, a missing test, a design
  issue, an unclear name) **and proposes a specific change**.

"LGTM" alone does not count. Whether the author acts on your review does not change whether it
counts for you.

**What you need across the project:** 3 substantive reviews (cross-team reviews count), at
least one that led to a change or a reasoned decline, one AI review of your own PR
(adjudicated, Step 9), and one cross-team application review in Part 3.

---

## Step 1. Find the PR and get the code

1. On the repository page, click **Pull requests**, then the PR.
2. Read the description. It should say what changed and `Closes #N`. Open that issue: it says
   what the PR is supposed to do.
3. Optional, but the best reviews do it: run the code.

   ```bash
   git fetch
   git switch <branch-name-from-the-PR>
   ./gradlew test
   ./gradlew run -Pdev
   ```

   Return with `git switch main`. With the GitHub CLI, `gh pr checkout <number>` does the
   first two lines.

---

## Step 2. Open "Files changed"

Click the **Files changed** tab. Every comment that counts is made here.

- Green lines (`+`) were added, red lines (`-`) removed.
- The file tree on the left lists every changed file.
- Tick **Viewed** on each file as you finish it, so you know where you stopped.

---

## Step 3. Read with a checklist

Read each file once for what it does, then again with this list. Every item is something
Project 2 grades.

| Look for | Example of a problem |
|---|---|
| Does it do what the issue asked? | The issue says "delete a favorite"; there is no delete |
| A test for each new method, **including a negative case** | `save` is tested, a duplicate `save` is not |
| A failure-path test for API code | Only the 200 case is tested |
| Magic numbers or strings | `new Scene(root, 640, 480)`; a URL typed into a scene |
| Resources closed | A `Connection` or `ResultSet` outside `try (...)` |
| SQL built with `+` and user input | `"... WHERE name = '" + name + "'"` |
| Exceptions swallowed or shown raw | `catch (Exception e) { }`, or `e.printStackTrace()` as the only handling |
| Network call on the JavaFX thread | A fetch inside `setOnAction` with no `Task` (fine, if it is noted) |
| Names that say what things are | `data2`, `doStuff`, `tmp` |
| Javadoc on public methods; Checkstyle clean | `./gradlew checkstyleMain` shows warnings |
| Files that should not be committed | `data/`, `*.mv.db`, `.idea/`, `build/` |
| Edits outside the author's slice | Changes to another slice's table or block |

---

## Step 4. Comment on a line

1. Hover over a line in **Files changed**. A blue comment icon (a `+`) appears beside the line
   number.
2. Click it. A comment box opens under the line.
   - For several lines, click the **+** on the first line and drag to the last.
3. Write the comment (Step 5 says how).
4. Click **Start a review**. (Later comments say **Add review comment**.)

**Start a review**, not **Comment**: a review collects your comments and posts them together
with a verdict in Step 7. **Comment** posts that one comment immediately, on its own.

Reference: [Commenting on a pull request](https://docs.github.com/en/pull-requests/collaborating-with-pull-requests/reviewing-changes-in-pull-requests/commenting-on-a-pull-request)

---

## Step 5. Write a comment that counts

Three parts: **where**, **what is wrong**, **what to do instead**.

| Does not count | Counts |
|---|---|
| "LGTM" | "`findById` returns `null` when nothing matches, and `FavoriteScene` line 40 calls `.name()` on it. Return `Optional<Favorite>` like `UserDao.findById`, and add a test for a missing id." |
| "Nice!" | "There is no test for a 404 from the API. Add one to `PokemonImportFailureTest` with `stub(\"/pokemon/missing\", 404, \"\")` and assert the table is still empty." |
| "Maybe rename this?" | "`list2` holds only the favorites. Rename it `favoriteList` so `reload()` reads clearly." |
| "Magic number" | "`640` and `480` are repeated in three scenes. Move them to `WIDTH` and `HEIGHT` constants." |

Two of your comments must be on lines. At least one must be like the right-hand column.
Questions ("why is this cached?") and praise are welcome **in addition**.

---

## Step 6. Suggest the exact change

For small fixes, give the author a one-click change:

1. In the comment box toolbar, click the suggestion icon (a page with `+` and `-`; hover
   text "Add a suggestion"). It inserts the selected lines. You can also type the block
   yourself:

   ````
   ```suggestion
   private static final int WIDTH = 640;
   ```
   ````

2. Edit the lines inside the block to what they should be.
3. **Start a review** / **Add review comment** as usual.

The author can click **Commit suggestion**, which makes a commit after your comment. That is
one way your review "led to a change before merge".

Reference: [Incorporating feedback](https://docs.github.com/en/pull-requests/collaborating-with-pull-requests/reviewing-changes-in-pull-requests/incorporating-feedback-in-your-pull-request)

---

## Step 7. Submit the review

1. Click **Review changes**, above the changed code at the top right of **Files changed**.
2. Write a one- or two-sentence summary: the most important issue first.
3. Choose one:

| Choice | Use when |
|---|---|
| **Comment** | Questions, small issues, nothing that must block the merge |
| **Approve** | It is ready. Still counts only with the two line comments and a concrete change |
| **Request changes** | Something must be fixed first: a bug, a missing required test |

4. Click **Submit review**.

Reference: [Reviewing proposed changes](https://docs.github.com/en/pull-requests/collaborating-with-pull-requests/reviewing-changes-in-pull-requests/reviewing-proposed-changes-in-a-pull-request)

---

## Step 8. If it is your PR: respond to every comment

Each comment gets one of two answers, as a reply in its thread:

- **Fixed:** push a commit, then reply `Fixed in <short commit hash>`. Click **Resolve
  conversation**.
- **Declined, with a reason:** `Not changing this: the API never returns null for name, and
  PokemonParserTest.missingNameIsAnApiException covers it.` A reasoned decline is a valid
  outcome. Silence is not; it is scored against the author in Part 4.

Then ask for another look: in the **Reviewers** box on the right, click the circular-arrows
icon next to the reviewer's name (**Re-request review**).

**Nobody reviewed in 48 hours?** Request your cross-team buddy or the TA under **Reviewers**.
A visible request means an unreviewed merge is not held against you.

---

## Step 9. The AI review of your own PR

Required once, **in addition to** the human review, on a **substantial** PR: your Milestone 2
PR or one of comparable size. "The AI found nothing" on a two-line diff is not an adjudication.
What is graded is your **decision** on each finding.

1. Get the diff:

   ```bash
   git diff main...HEAD > my-pr.diff
   ```

   (or `gh pr diff <number> > my-pr.diff`)
2. Give it to an AI tool with a specific request, for example: "Review this Java diff for bugs,
   missing negative tests, resource leaks and magic numbers. Cite the file and line for each."
3. For **each** finding, decide: **accept** (commit the fix) or **reject** (say why: it misread
   the code, it is out of scope, it is wrong about the API).
4. Record it **in the PR**, as a comment with three parts, as the prompt requires:

   ```
   AI review. Tool: <name and version>. PR: #<number>.

   Findings:
   1. <finding, file:line>
   2. ...

   Adjudication:
   1. Accepted, fixed in <commit hash>.
   2. Rejected: the ResultSet is already in the try-with-resources on line 88.
   ```

5. Add the comment's link to the "AI review links" in `DESIGN.md`.

A finding you accept without checking is the AI's review, not yours. A good rejection is
evidence you read the code.

---

## Step 10. Link the review in your index (Part 3)

For each of your three best reviews:

1. Open the PR's **Conversation** tab and find your review ("*you* reviewed ...").
2. Click the time stamp next to it ("2 days ago"). The address bar now shows a link ending in
   `#pullrequestreview-<number>`. Copy it.
3. Write one line: *what I caught*. For example: "Caught a missing 404 test; the author added
   one in `a1b2c3d`."

To link one line comment instead, use the **...** menu on the comment, **Copy link**.

---

## Troubleshooting

| Problem | Fix |
|---|---|
| No blue comment icon on hover | You are on **Conversation** or **Commits**. Switch to **Files changed** |
| My comments are not visible to the author | The review is still pending. Click **Review changes** → **Submit review** |
| "Outdated" on a comment | The author changed those lines after you commented. Normal: it shows they acted |
| Cannot see the other team's repository | Ask the instructor for access before the deadline |
| `git switch <branch>` says the branch does not exist | Run `git fetch` first |
| I cannot approve my own PR | GitHub does not allow it. Ask a teammate, your cross-team buddy, or the TA |

---

## Links

- GitHub Docs, reviewing changes in pull requests: <https://docs.github.com/en/pull-requests/collaborating-with-pull-requests/reviewing-changes-in-pull-requests>
- GitHub Docs, about pull request reviews: <https://docs.github.com/en/pull-requests/collaborating-with-pull-requests/reviewing-changes-in-pull-requests/about-pull-request-reviews>
- Google's code review guide, "What to look for": <https://google.github.io/eng-practices/review/reviewer/looking-for.html>
- In the prompt: "Code Review" and "GitHub & Project Tracking".
