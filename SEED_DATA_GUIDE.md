# Schema First, and Seed Data

Follow the steps in order. When you finish, your slice:

- has its table in `schema.sql`, merged by its own small Pull Request, and
- runs with `./gradlew run -Pdev` and shows real-looking rows, **even if the teammate whose
  table you read has not finished**.

**Why.** From the prompt: "If your slice reads a table another student owns, you ship fixtures
for it. ... it is the rule that lets your slice be demonstrated and tested whether or not your
teammates finished." And: "Your table's DDL is merged into `schema.sql` during Part 1, as its
own small reviewed Pull Request."

**The example.** Slice 2 owns `pokemon` (see [`DAO_GUIDE.md`](DAO_GUIDE.md)). Slice 3 owns
`favorites`: which user likes which Pokemon. Favorites reads two tables it does not own,
`users` and `pokemon`, so Slice 3 seeds rows for both.

**Files you touch**

| File | Runs | Holds |
|---|---|---|
| `src/main/resources/schema.sql` | every startup, and every test that calls `Database.init` | `CREATE TABLE` |
| `src/main/resources/seed-dev.sql` | only with `-Pdev`, and in tests that call `Database.init(true)` | fixture rows |

---

## Step 1. Turn your ERD box into SQL

| ERD | SQL |
|---|---|
| Entity | `CREATE TABLE IF NOT EXISTS name (...)` |
| Identifier | `name_id INT AUTO_INCREMENT PRIMARY KEY` (or the API's id, see `DAO_GUIDE.md` Step 1) |
| Attribute | a column with a type: `VARCHAR(n)`, `INT`, `BOOLEAN`, `TIMESTAMP`, `DECIMAL(p, s)` |
| Required attribute | `NOT NULL` |
| "No two rows with the same ..." | `UNIQUE (column)` or `UNIQUE (a, b)` |
| Relationship (many-to-one) | `other_id INT NOT NULL REFERENCES other(other_id)` |
| Many-to-many | a link table with two `REFERENCES` and `UNIQUE (a, b)` |

`favorites` is a many-to-many link between `users` and `pokemon`:

```sql
-- ---- Slice 3 (owner: ) ----
CREATE TABLE IF NOT EXISTS favorites (
    favorite_id INT AUTO_INCREMENT PRIMARY KEY,
    user_id     INT NOT NULL REFERENCES users(user_id),
    pokemon_id  INT NOT NULL REFERENCES pokemon(pokemon_id),
    UNIQUE (user_id, pokemon_id)
);
```

H2 reference: [`CREATE TABLE`](https://h2database.com/html/commands.html#create_table),
[data types](https://h2database.com/html/datatypes.html).

---

## Step 2. Put it in your block, in the right order

1. Put your `CREATE TABLE` inside **your slice's block** in `schema.sql`. The blocks are
   pre-cut so two people adding tables do not conflict.
2. **A table must come after every table it `REFERENCES`.** `schema.sql` runs top to bottom.
   If `favorites` came before `pokemon`, startup fails with:

   ```
   Failed running /schema.sql: Table "POKEMON" not found
   ```

   If your block is above the table you reference, agree with that teammate to swap blocks.
3. Never edit a table you do not own. If you need more data about a user, add your own table
   with `user_id INT REFERENCES users(user_id)`.

Run `./gradlew run -Pdev`. If it starts, the SQL is valid.

---

## Step 3. Open the schema Pull Request

This PR is the team's agreement on your table, so keep it small: **only** your `CREATE TABLE`.

```bash
git switch main
git pull
git switch -c <github-username>/favorites-schema
# edit schema.sql
git add src/main/resources/schema.sql
git commit -m "Add favorites table (Slice 3)"
git push -u origin <github-username>/favorites-schema
```

On GitHub: **Compare & pull request**, write `Closes #N` (your schema issue) in the
description, and request a review from the owner of every table you `REFERENCES`. They are the
people who can tell you that `pokemon_id` is the wrong key. See
[`CODE_REVIEW_GUIDE.md`](CODE_REVIEW_GUIDE.md).

---

## Step 4. Seed the rows your slice reads

`src/main/resources/seed-dev.sql`, inside **your slice's block**:

```sql
-- ---- Slice 3 (owner: ) ----
-- Favorites reads pokemon, which Slice 2 owns. Seed the rows we need so this slice works
-- even before Slice 2 is finished. MERGE makes a duplicate fixture harmless.
MERGE INTO pokemon (pokemon_id, name, height, weight, types) KEY(pokemon_id) VALUES
    (25, 'pikachu',   4,  60,  'electric'),
    (6,  'charizard', 17, 905, 'fire,flying');

MERGE INTO favorites (favorite_id, user_id, pokemon_id) KEY(favorite_id) VALUES
    (1, 1, 25),
    (2, 1, 6),
    (3, 2, 25);

ALTER TABLE favorites ALTER COLUMN favorite_id
    RESTART WITH (SELECT GREATEST(1000, MAX(favorite_id) + 1) FROM favorites);
```

The rules, and why:

| Rule | Why |
|---|---|
| `MERGE INTO ... KEY(...)`, never `INSERT` | `-Pdev` runs this file on **every** launch. `INSERT` would fail on the second launch; `MERGE` updates the row that is already there |
| Explicit ids (`1`, `2`, `25`) | Other fixtures point at them: `(1, 1, 25)` means user 1 likes Pokemon 25 |
| Users `1` and `2` | The starter's `dev_admin` and `dev_user`. Reference them; do not add users |
| Seed the other table's rows too | Your slice works before its owner finishes. If they seed the same id, the later `MERGE` wins, so agree on the values |
| Parent rows before child rows | `favorites` rows reference `pokemon` rows, so the `pokemon` `MERGE` comes first |
| The `RESTART WITH (SELECT ...)` line | Explained in Step 5 |

---

## Step 5. Restart the id counter safely

Supplying explicit ids does not move H2's `AUTO_INCREMENT` counter. Without a restart, the
first real insert tries id `1` and fails with a primary key violation.

**Use the computed form:**

```sql
ALTER TABLE favorites ALTER COLUMN favorite_id
    RESTART WITH (SELECT GREATEST(1000, MAX(favorite_id) + 1) FROM favorites);
```

**Not** the fixed form:

```sql
ALTER TABLE favorites ALTER COLUMN favorite_id RESTART WITH 1000;   -- breaks on the 2nd launch
```

The fixed form works once. On the next `-Pdev` launch it resets the counter to 1000 again, and
the first insert collides with the row that already has id 1000. The computed form restarts
above the highest id in the table: 1000 on a fresh database, and past your own rows after that.

> **Fix the `users` line too.** If your team's `seed-dev.sql` still has
> `ALTER TABLE users ALTER COLUMN user_id RESTART WITH 1000;`, it has this bug: after one
> registration, the next `-Pdev` launch cannot add a user. Change it to the computed form, in
> one small PR (the Accounts owner, or whoever gets there first).
> `SeedDataTest.secondLaunchStillInsertsUsers` fails until you do:
>
> ```sql
> ALTER TABLE users ALTER COLUMN user_id
>     RESTART WITH (SELECT GREATEST(1000, MAX(user_id) + 1) FROM users);
> ```

Tables keyed by the API's id (like `pokemon`) have no `AUTO_INCREMENT`, so they need no restart.

---

## Step 6. Test the schema and the fixtures

A test that loads the fixtures the way `-Pdev` does catches a broken seed before your video
does. This uses a small `FavoriteDao`:

`src/main/java/edu/csumb/cst338/dao/FavoriteDao.java`

```java
package edu.csumb.cst338.dao;

import edu.csumb.cst338.db.Database;
import java.sql.Connection;
import java.sql.PreparedStatement;
import java.sql.ResultSet;
import java.sql.SQLException;
import java.sql.Statement;
import java.util.ArrayList;
import java.util.List;

/**
 * Reads and writes the {@code favorites} table, which links a user to a Pokemon.
 *
 * <p>Update is omitted on purpose: a favorite is added or removed, never edited.
 */
public class FavoriteDao {

    private static final String INSERT =
            "INSERT INTO favorites (user_id, pokemon_id) VALUES (?, ?)";
    private static final String NAMES_FOR_USER =
            "SELECT p.name FROM favorites f JOIN pokemon p ON p.pokemon_id = f.pokemon_id "
                    + "WHERE f.user_id = ? ORDER BY p.name";

    /**
     * Marks a Pokemon as one of a user's favorites.
     *
     * @param userId    an existing {@code users.user_id}
     * @param pokemonId an existing {@code pokemon.pokemon_id}
     * @return the new favorite's id
     * @throws DataAccessException if either id does not exist, or it is already a favorite
     */
    public int add(int userId, int pokemonId) {
        try (Connection c = Database.getConnection();
             PreparedStatement ps = c.prepareStatement(INSERT, Statement.RETURN_GENERATED_KEYS)) {
            ps.setInt(1, userId);
            ps.setInt(2, pokemonId);
            ps.executeUpdate();
            try (ResultSet keys = ps.getGeneratedKeys()) {
                keys.next();
                return keys.getInt(1);
            }
        } catch (SQLException e) {
            throw new DataAccessException(
                    "Could not add favorite " + pokemonId + " for user " + userId, e);
        }
    }

    /**
     * Lists the names of a user's favorite Pokemon.
     *
     * @param userId the user
     * @return names in alphabetical order; empty if there are none
     */
    public List<String> namesFor(int userId) {
        List<String> out = new ArrayList<>();
        try (Connection c = Database.getConnection();
             PreparedStatement ps = c.prepareStatement(NAMES_FOR_USER)) {
            ps.setInt(1, userId);
            try (ResultSet rs = ps.executeQuery()) {
                while (rs.next()) {
                    out.add(rs.getString("name"));
                }
            }
            return out;
        } catch (SQLException e) {
            throw new DataAccessException("Could not list favorites for user " + userId, e);
        }
    }
}
```

`src/test/java/edu/csumb/cst338/dao/SeedDataTest.java`

```java
package edu.csumb.cst338.dao;

import static org.junit.jupiter.api.Assertions.assertDoesNotThrow;
import static org.junit.jupiter.api.Assertions.assertEquals;
import static org.junit.jupiter.api.Assertions.assertThrows;
import static org.junit.jupiter.api.Assertions.assertTrue;

import edu.csumb.cst338.db.Database;
import edu.csumb.cst338.model.Pokemon;
import edu.csumb.cst338.model.Role;
import java.util.List;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

/** The schema and the dev fixtures, loaded the way -Pdev loads them. */
class SeedDataTest {

    private static final int DEV_ADMIN_ID = 1;
    private static final int DEV_USER_ID = 2;
    private static final int PIKACHU = 25;
    private static final int CHARIZARD = 6;
    private static final int BULBASAUR = 1;
    private static final int NOT_A_POKEMON = 4242;
    private static final int FIRST_NEW_ID = 1000;

    private FavoriteDao favorites;

    @BeforeEach
    void setUp() {
        Database.useInMemory();
        Database.reset();
        Database.init(true);
        favorites = new FavoriteDao();
    }

    /** Runs the startup scripts again on the same database, as a second -Pdev launch does. */
    private void restartApp() {
        Database.useInMemory();
        Database.init(true);
    }

    @Test
    void fixturesAreLoaded() {
        assertEquals(List.of("charizard", "pikachu"), favorites.namesFor(DEV_ADMIN_ID));
        assertEquals(List.of("pikachu"), favorites.namesFor(DEV_USER_ID));
    }

    @Test
    void newRowsStartAboveTheFixtures() {
        assertEquals(FIRST_NEW_ID, favorites.add(DEV_USER_ID, CHARIZARD));
    }

    @Test
    void secondLaunchStillInserts() {
        favorites.add(DEV_USER_ID, CHARIZARD);
        restartApp();
        new PokemonDao().save(new Pokemon(BULBASAUR, "bulbasaur", 7, 69, List.of("grass")));
        int id = assertDoesNotThrow(() -> favorites.add(DEV_USER_ID, BULBASAUR));
        assertEquals(FIRST_NEW_ID + 1, id);
    }

    @Test
    void secondLaunchStillInsertsUsers() {
        new UserDao().insert("first_run", "pw", Role.USER);
        restartApp();
        assertDoesNotThrow(() -> new UserDao().insert("second_run", "pw", Role.USER));
    }

    @Test
    void relaunchingDoesNotDuplicateFixtures() {
        restartApp();
        assertEquals(List.of("charizard", "pikachu"), favorites.namesFor(DEV_ADMIN_ID));
    }

    @Test
    void unknownPokemonIsRejected() {
        assertThrows(DataAccessException.class, () -> favorites.add(DEV_USER_ID, NOT_A_POKEMON));
    }

    @Test
    void sameFavoriteTwiceIsRejected() {
        assertThrows(DataAccessException.class, () -> favorites.add(DEV_USER_ID, PIKACHU));
    }

    @Test
    void withoutDevModeTheTablesAreEmpty() {
        Database.reset();
        Database.init(false);
        assertTrue(favorites.namesFor(DEV_ADMIN_ID).isEmpty());
    }
}
```

| Test | Catches |
|---|---|
| `fixturesAreLoaded` | A typo or a wrong id in `seed-dev.sql` |
| `newRowsStartAboveTheFixtures` | A missing `RESTART` |
| `secondLaunchStillInserts`, `...Users` | The fixed `RESTART WITH 1000` bug from Step 5 |
| `relaunchingDoesNotDuplicateFixtures` | `INSERT` where `MERGE` belongs |
| `unknownPokemonIsRejected`, `sameFavoriteTwiceIsRejected` | Missing `REFERENCES` or `UNIQUE` |
| `withoutDevModeTheTablesAreEmpty` | Fixtures leaking into the real app |

`restartApp()` re-runs the startup scripts on the same in-memory database, which is what a
second `-Pdev` launch does to `data/app.mv.db`.

Run `./gradlew test`. Expect `BUILD SUCCESSFUL`.

---

## Step 7. See it in the app

```bash
./gradlew run -Pdev
```

Your scene shows the seeded rows. With the fixtures above, user 1 (`dev_admin`, the default
`-Pdev` login) has two favorites: charizard and pikachu. `-Prole=USER` signs in as user 2.

To start over with a clean database, close the app and delete the `data/` folder. `data/` is
in `.gitignore`; never commit it.

---

## Step 8. Do it for your project

1. Turn your ERD box into a `CREATE TABLE` (Step 1 table).
2. Put it in your block, below every table it references (Step 2).
3. Open the schema PR. Request review from the owners of the tables you reference (Step 3).
4. List every table your slice **reads**. Seed rows for each in your block of `seed-dev.sql`.
5. `MERGE ... KEY(...)` for every fixture. The computed `RESTART` for every `AUTO_INCREMENT` table you seed.
6. Copy `SeedDataTest`: fixtures load, second launch still inserts, constraints reject bad rows.

---

## Troubleshooting

| Symptom | Cause | Fix |
|---|---|---|
| `Failed running /schema.sql: Table "X" not found` | Your table is above a table it references | Move it below (Step 2) |
| `Failed running /seed-dev.sql: Referential integrity constraint violation` | A child row before its parent row, or a parent id that does not exist | Seed parents first; check the ids |
| `Unique index or primary key violation` on the 2nd `-Pdev` launch | `INSERT` in the seed, or the fixed `RESTART WITH 1000` | `MERGE ... KEY(...)`, computed `RESTART` |
| First insert in the app fails with a primary key violation | No `RESTART` after explicit ids | Add the computed `RESTART` |
| Seeded rows missing in the app | Ran without `-Pdev` | `./gradlew run -Pdev` |
| Seeded rows appear in the real app | Seed run unconditionally | Only `Database.init(true)` loads seeds; check `Main` |
| Old columns still there after editing `CREATE TABLE` | `IF NOT EXISTS` skipped it: the table already exists in `data/` | Close the app, delete `data/`, restart |

---

## Links

- H2 `MERGE`: <https://h2database.com/html/commands.html#merge_into>
- H2 `ALTER TABLE ... ALTER COLUMN`: <https://h2database.com/html/commands.html#alter_table_alter_column>
- In your team's repo: `schema.sql` and `seed-dev.sql` (the comments explain the blocks),
  `Database.java` (when each script runs).
