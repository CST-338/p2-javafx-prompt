# Saving API Data Through a DAO

Follow the steps in order. This guide continues [`API_GUIDE.md`](API_GUIDE.md): you have a
`Pokemon` record built from PokeAPI. Now you save it, so the app works with the network
unplugged. When you finish, this works:

```
run 1 (network):    Imported 2
run 2 (no network): Pokemon[id=6, name=charizard, height=17, weight=905, types=[fire, flying]]
                    Pokemon[id=25, name=pikachu, height=4, weight=60, types=[electric]]
```

**Why.** README API rule 1: a response that is printed and thrown away does not count. Rule 2:
the API is an **import source** and your tables are the **system of record**. Scenes read
from the table, never from the API.

**The flow**

```
Import button → PokemonImporter → PokemonService → ApiClient → PokeAPI
                      │
                      └─ PokemonDao.save(...) → pokemon table
List           ← PokemonDao.findAll() ← pokemon table
```

**What you already have.** `Database` (connections, `schema.sql` on startup, in-memory mode for
tests), `UserDao` and `UserDaoTest` (the pattern to copy), `DataAccessException`.

**Stay on the `practice/pokeapi` branch** from `API_GUIDE.md` if you are following the Pokemon
example: this guide builds on its classes. Copy only the pattern into your slice's branch.

---

## Step 1. Add your table to `schema.sql`

`src/main/resources/schema.sql`, inside **your slice's block**:

```sql
-- ---- Slice 2 (owner: ) ----
CREATE TABLE IF NOT EXISTS pokemon (
    pokemon_id  INT          PRIMARY KEY,
    name        VARCHAR(64)  NOT NULL UNIQUE,
    height      INT          NOT NULL,
    weight      INT          NOT NULL,
    types       VARCHAR(255) NOT NULL,
    imported_at TIMESTAMP    DEFAULT CURRENT_TIMESTAMP NOT NULL
);
```

Decisions in this table, and why:

| Choice | Why |
|---|---|
| `pokemon_id` is the **API's id**, not `AUTO_INCREMENT` | Importing the same Pokemon twice hits the same row, so there are no duplicates |
| `IF NOT EXISTS` | `schema.sql` runs on every startup |
| `NOT NULL` on every column you need | The database rejects a half-filled row instead of storing it |
| `types` is one comma-separated column | Simple. A separate `pokemon_types` table is the normalized design (see Step 8) |
| `imported_at` | Shows how old the data is. Enhancement A5 (cache with a TTL) builds on it |

If your API has **no stable id**, use `AUTO_INCREMENT` for the key and put `UNIQUE` on the
column that identifies a row (a name, a URL, an ISBN).

Run `./gradlew run -Pdev` once. A typo in the SQL stops the app at startup with
`Failed running /schema.sql: ...`.

Schema reference: [H2 `CREATE TABLE`](https://h2database.com/html/commands.html#create_table),
[H2 data types](https://h2database.com/html/datatypes.html).

---

## Step 2. Write the DAO

One method per thing the app needs. Copy the shape of `UserDao`.

`src/main/java/edu/csumb/cst338/dao/PokemonDao.java`

```java
package edu.csumb.cst338.dao;

import edu.csumb.cst338.db.Database;
import edu.csumb.cst338.model.Pokemon;
import java.sql.Connection;
import java.sql.PreparedStatement;
import java.sql.ResultSet;
import java.sql.SQLException;
import java.util.ArrayList;
import java.util.Arrays;
import java.util.List;
import java.util.Optional;

/**
 * Reads and writes the {@code pokemon} table.
 *
 * <p>The primary key is PokeAPI's own id, so importing the same Pokemon twice updates
 * one row instead of adding a second.
 */
public class PokemonDao {

    private static final String TYPE_SEPARATOR = ",";
    private static final String COLUMNS = "pokemon_id, name, height, weight, types";

    private static final String UPSERT =
            "MERGE INTO pokemon (" + COLUMNS + ") KEY(pokemon_id) VALUES (?, ?, ?, ?, ?)";
    private static final String SELECT_BY_ID =
            "SELECT " + COLUMNS + " FROM pokemon WHERE pokemon_id = ?";
    private static final String SELECT_BY_NAME =
            "SELECT " + COLUMNS + " FROM pokemon WHERE name = ?";
    private static final String SELECT_ALL =
            "SELECT " + COLUMNS + " FROM pokemon ORDER BY pokemon_id";
    private static final String COUNT = "SELECT COUNT(*) FROM pokemon";
    private static final String DELETE = "DELETE FROM pokemon WHERE pokemon_id = ?";

    /**
     * Inserts the Pokemon, or updates it if its id is already stored.
     *
     * @param p the Pokemon to save
     * @throws DataAccessException if the database rejects it, for example a duplicate name
     */
    public void save(Pokemon p) {
        try (Connection c = Database.getConnection();
             PreparedStatement ps = c.prepareStatement(UPSERT)) {
            ps.setInt(1, p.id());
            ps.setString(2, p.name());
            ps.setInt(3, p.height());
            ps.setInt(4, p.weight());
            ps.setString(5, String.join(TYPE_SEPARATOR, p.types()));
            ps.executeUpdate();
        } catch (SQLException e) {
            throw new DataAccessException("Could not save " + p.name(), e);
        }
    }

    /**
     * Looks up one Pokemon by id.
     *
     * @param id PokeAPI's id
     * @return the Pokemon, or empty if it is not stored
     */
    public Optional<Pokemon> findById(int id) {
        return queryOne(SELECT_BY_ID, ps -> ps.setInt(1, id));
    }

    /**
     * Looks up one Pokemon by name.
     *
     * @param name for example {@code pikachu}
     * @return the Pokemon, or empty if it is not stored
     */
    public Optional<Pokemon> findByName(String name) {
        return queryOne(SELECT_BY_NAME, ps -> ps.setString(1, name));
    }

    /**
     * Lists every stored Pokemon.
     *
     * @return all rows, ordered by id; empty if there are none
     */
    public List<Pokemon> findAll() {
        List<Pokemon> out = new ArrayList<>();
        try (Connection c = Database.getConnection();
             PreparedStatement ps = c.prepareStatement(SELECT_ALL);
             ResultSet rs = ps.executeQuery()) {
            while (rs.next()) {
                out.add(map(rs));
            }
            return out;
        } catch (SQLException e) {
            throw new DataAccessException("Could not list pokemon", e);
        }
    }

    /**
     * Counts the stored Pokemon.
     *
     * @return the number of rows
     */
    public int count() {
        try (Connection c = Database.getConnection();
             PreparedStatement ps = c.prepareStatement(COUNT);
             ResultSet rs = ps.executeQuery()) {
            rs.next();
            return rs.getInt(1);
        } catch (SQLException e) {
            throw new DataAccessException("Could not count pokemon", e);
        }
    }

    /**
     * Deletes one Pokemon.
     *
     * @param id PokeAPI's id
     * @return true if a row was deleted, false if there was none
     */
    public boolean delete(int id) {
        try (Connection c = Database.getConnection();
             PreparedStatement ps = c.prepareStatement(DELETE)) {
            ps.setInt(1, id);
            return ps.executeUpdate() == 1;
        } catch (SQLException e) {
            throw new DataAccessException("Could not delete pokemon " + id, e);
        }
    }

    private Optional<Pokemon> queryOne(String sql, StatementBinder binder) {
        try (Connection c = Database.getConnection();
             PreparedStatement ps = c.prepareStatement(sql)) {
            binder.bind(ps);
            try (ResultSet rs = ps.executeQuery()) {
                return rs.next() ? Optional.of(map(rs)) : Optional.empty();
            }
        } catch (SQLException e) {
            throw new DataAccessException("Query failed: " + sql, e);
        }
    }

    private static Pokemon map(ResultSet rs) throws SQLException {
        List<String> types = Arrays.asList(rs.getString("types").split(TYPE_SEPARATOR));
        return new Pokemon(rs.getInt("pokemon_id"), rs.getString("name"),
                rs.getInt("height"), rs.getInt("weight"), List.copyOf(types));
    }

    @FunctionalInterface
    private interface StatementBinder {
        void bind(PreparedStatement ps) throws SQLException;
    }
}
```

What each part does:

| Part | Why |
|---|---|
| `MERGE INTO ... KEY(pokemon_id)` | H2's insert-or-update. A second import updates the row |
| `?` placeholders and `ps.setInt(...)` | Prepared statements. **Never** build SQL with `+` and user input |
| `try (Connection c = ...; PreparedStatement ps = ...)` | Closes both, even when an exception is thrown |
| `Optional<Pokemon>` | A lookup that can miss says so in its type, instead of returning `null` |
| `DataAccessException` | One unchecked exception for every database failure, with a readable message |
| `map(ResultSet)` | One place turns a row into a record |

H2 `MERGE` reference: <https://h2database.com/html/commands.html#merge_into>

---

## Step 3. Test the DAO against an in-memory database

The `@BeforeEach` block points every connection at a fresh in-memory database, so tests never
touch your real data or depend on each other.

`src/test/java/edu/csumb/cst338/dao/PokemonDaoTest.java`

```java
package edu.csumb.cst338.dao;

import static org.junit.jupiter.api.Assertions.assertEquals;
import static org.junit.jupiter.api.Assertions.assertFalse;
import static org.junit.jupiter.api.Assertions.assertThrows;
import static org.junit.jupiter.api.Assertions.assertTrue;

import edu.csumb.cst338.db.Database;
import edu.csumb.cst338.model.Pokemon;
import java.util.List;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

/** PokemonDao against an in-memory database. Never touches your real data. */
class PokemonDaoTest {

    private static final Pokemon PIKACHU = new Pokemon(25, "pikachu", 4, 60, List.of("electric"));
    private static final Pokemon CHARIZARD =
            new Pokemon(6, "charizard", 17, 905, List.of("fire", "flying"));

    private PokemonDao dao;

    @BeforeEach
    void setUp() {
        Database.useInMemory();
        Database.reset();
        Database.init(false);
        dao = new PokemonDao();
    }

    @Test
    void saveThenFindByIdReturnsTheSamePokemon() {
        dao.save(CHARIZARD);
        assertEquals(CHARIZARD, dao.findById(6).orElseThrow());
    }

    @Test
    void findByNameFindsIt() {
        dao.save(PIKACHU);
        assertEquals(PIKACHU, dao.findByName("pikachu").orElseThrow());
    }

    @Test
    void savingTheSameIdTwiceUpdatesInsteadOfDuplicating() {
        dao.save(PIKACHU);
        Pokemon heavier = new Pokemon(25, "pikachu", 4, 61, List.of("electric"));
        dao.save(heavier);
        assertEquals(1, dao.count());
        assertEquals(61, dao.findById(25).orElseThrow().weight());
    }

    @Test
    void findAllIsOrderedById() {
        dao.save(PIKACHU);
        dao.save(CHARIZARD);
        assertEquals(List.of(CHARIZARD, PIKACHU), dao.findAll());
    }

    @Test
    void missingIdIsEmptyNotNull() {
        assertTrue(dao.findById(4242).isEmpty());
    }

    @Test
    void emptyTableGivesEmptyList() {
        assertEquals(List.of(), dao.findAll());
        assertEquals(0, dao.count());
    }

    @Test
    void sameNameUnderADifferentIdIsRejected() {
        dao.save(PIKACHU);
        Pokemon impostor = new Pokemon(999, "pikachu", 1, 1, List.of("normal"));
        assertThrows(DataAccessException.class, () -> dao.save(impostor));
    }

    @Test
    void deleteReportsWhetherARowMatched() {
        dao.save(PIKACHU);
        assertTrue(dao.delete(25));
        assertFalse(dao.delete(25));
        assertTrue(dao.findById(25).isEmpty());
    }
}
```

Run `./gradlew test`. Expect `BUILD SUCCESSFUL`. Four of the eight tests are negative or edge
cases (missing id, empty table, duplicate name, delete twice). That is the ratio to aim for.

> **The three lines in `setUp` are required, in that order.** Skip `useInMemory()` and the tests
> write to `data/app.mv.db`, your real database.

---

## Step 4. Write the importer: fetch, then save

`src/main/java/edu/csumb/cst338/api/PokemonImporter.java`

```java
package edu.csumb.cst338.api;

import edu.csumb.cst338.dao.PokemonDao;
import edu.csumb.cst338.model.Pokemon;
import java.util.ArrayList;
import java.util.List;

/** Fetches Pokemon from the API and saves them through the DAO. */
public class PokemonImporter {

    private final PokemonService service;
    private final PokemonDao dao;

    /**
     * Creates the importer.
     *
     * @param service fetches from the API
     * @param dao     saves to the database
     */
    public PokemonImporter(PokemonService service, PokemonDao dao) {
        this.service = service;
        this.dao = dao;
    }

    /**
     * Fetches every name first, then saves them all.
     *
     * <p>Fetching first means a failure on the third name saves nothing, rather than
     * leaving two of three in the table.
     *
     * @param names for example {@code List.of("pikachu", "charizard")}
     * @return how many were saved
     * @throws ApiException if any fetch fails; nothing is saved in that case
     */
    public int importAll(List<String> names) {
        List<Pokemon> fetched = new ArrayList<>();
        for (String name : names) {
            fetched.add(service.fetch(name));
        }
        for (Pokemon p : fetched) {
            dao.save(p);
        }
        return fetched.size();
    }
}
```

> **Fetch everything first, then save.** If the third name fails, nothing is saved. Saving as
> you go would leave the table half-imported. This is not a database transaction; a failure
> *during* the saves can still leave some rows. Making that all-or-nothing is enhancement B9.

---

## Step 5. Test the importer with a fake GET

`PokemonService` takes any URL-to-body function (see `API_GUIDE.md` Step 7), so the test
passes a lambda that answers from a `Map`. No network, no stub server.

`src/test/java/edu/csumb/cst338/api/PokemonImporterTest.java`

```java
package edu.csumb.cst338.api;

import static org.junit.jupiter.api.Assertions.assertEquals;
import static org.junit.jupiter.api.Assertions.assertThrows;

import edu.csumb.cst338.dao.PokemonDao;
import edu.csumb.cst338.db.Database;
import java.util.List;
import java.util.Map;
import java.util.function.Function;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

/** The import path with a fake HTTP GET: no network, real in-memory database. */
class PokemonImporterTest {

    private static final String BASE = "http://fake";
    private static final Map<String, String> RESPONSES = Map.of(
            BASE + "/pokemon/pikachu",
            "{\"id\":25,\"name\":\"pikachu\",\"height\":4,\"weight\":60,"
                    + "\"types\":[{\"slot\":1,\"type\":{\"name\":\"electric\",\"url\":\"u\"}}]}",
            BASE + "/pokemon/charizard",
            "{\"id\":6,\"name\":\"charizard\",\"height\":17,\"weight\":905,\"types\":["
                    + "{\"slot\":1,\"type\":{\"name\":\"fire\",\"url\":\"u\"}},"
                    + "{\"slot\":2,\"type\":{\"name\":\"flying\",\"url\":\"u\"}}]}");

    /** A fake GET: answers from the map, and acts like a 404 for anything else. */
    private static final Function<String, String> FAKE_GET = url -> {
        String body = RESPONSES.get(url);
        if (body == null) {
            throw new ApiException("The service rejected the request (404).", 404, false);
        }
        return body;
    };

    private PokemonDao dao;

    @BeforeEach
    void setUp() {
        Database.useInMemory();
        Database.reset();
        Database.init(false);
        dao = new PokemonDao();
    }

    private PokemonImporter importer(Function<String, String> get) {
        return new PokemonImporter(new PokemonService(get, BASE), dao);
    }

    @Test
    void importSavesEveryPokemon() {
        int saved = importer(FAKE_GET).importAll(List.of("pikachu", "charizard"));
        assertEquals(2, saved);
        assertEquals(List.of("fire", "flying"), dao.findByName("charizard").orElseThrow().types());
    }

    @Test
    void importingTwiceDoesNotDuplicate() {
        importer(FAKE_GET).importAll(List.of("pikachu"));
        importer(FAKE_GET).importAll(List.of("pikachu"));
        assertEquals(1, dao.count());
    }

    @Test
    void dataIsStillThereWithTheNetworkGone() {
        importer(FAKE_GET).importAll(List.of("pikachu"));
        Function<String, String> offline = url -> {
            throw new ApiException("Could not reach the service.", 0, true);
        };
        assertThrows(ApiException.class, () -> importer(offline).importAll(List.of("charizard")));
        assertEquals("pikachu", dao.findById(25).orElseThrow().name());
    }

    @Test
    void oneBadNameSavesNothing() {
        assertThrows(ApiException.class,
                () -> importer(FAKE_GET).importAll(List.of("pikachu", "pikachoo")));
        assertEquals(0, dao.count());
    }
}
```

`dataIsStillThereWithTheNetworkGone` is README rule 2, as a test.

---

## Step 6. Show the table in a scene

The list reads from the DAO. **Import** fetches, saves, then reloads the list from the table.

`src/main/java/edu/csumb/cst338/scenes/PokedexView.java`

```java
package edu.csumb.cst338.scenes;

import edu.csumb.cst338.api.ApiClient;
import edu.csumb.cst338.api.ApiConfig;
import edu.csumb.cst338.api.ApiException;
import edu.csumb.cst338.api.PokemonImporter;
import edu.csumb.cst338.api.PokemonService;
import edu.csumb.cst338.dao.DataAccessException;
import edu.csumb.cst338.dao.PokemonDao;
import edu.csumb.cst338.model.Pokemon;
import java.util.List;
import javafx.geometry.Insets;
import javafx.scene.Parent;
import javafx.scene.control.Button;
import javafx.scene.control.Label;
import javafx.scene.control.ListView;
import javafx.scene.control.TextField;
import javafx.scene.layout.HBox;
import javafx.scene.layout.VBox;

/** The list shows the table. Import fetches from the API, saves, then reloads the list. */
public class PokedexView {

    private static final int SPACING = 10;
    private static final int PADDING = 20;

    private final PokemonImporter importer;
    private final PokemonDao dao;
    private final TextField nameField = new TextField();
    private final ListView<Pokemon> list = new ListView<>();
    private final Label status = new Label();

    /** Uses the real API. This is what SceneFactory calls. */
    public PokedexView() {
        this(new PokemonImporter(
                new PokemonService(new ApiClient(ApiConfig.require("api.userAgent"))::getJson,
                        ApiConfig.require("api.baseUrl")),
                new PokemonDao()), new PokemonDao());
    }

    /**
     * Uses the importer and DAO it is given. Tests pass a fake importer here.
     *
     * @param importer fetches and saves
     * @param dao      reads the table
     */
    public PokedexView(PokemonImporter importer, PokemonDao dao) {
        this.importer = importer;
        this.dao = dao;
    }

    /**
     * Builds the scene's root.
     *
     * @return the layout
     */
    public Parent getRoot() {
        nameField.setId("nameField");
        nameField.setPromptText("pikachu");
        status.setId("status");
        Button importButton = new Button("Import");
        importButton.setOnAction(e -> handleImport());
        reload();
        VBox root = new VBox(SPACING, new HBox(SPACING, nameField, importButton), status, list);
        root.setPadding(new Insets(PADDING));
        return root;
    }

    private void handleImport() {
        // NOTE: blocks the JavaFX Application Thread until the API answers. See TASK_GUIDE.md.
        String name = nameField.getText().trim().toLowerCase();
        try {
            int saved = importer.importAll(List.of(name));
            status.setText("Imported " + saved + ".");
            reload();
        } catch (ApiException | DataAccessException ex) {
            status.setText(ex.getMessage());
        }
    }

    private void reload() {
        list.getItems().setAll(dao.findAll());
    }
}
```

Two constructors: the no-argument one uses the real API and is what `SceneFactory` calls. The
other takes the importer and DAO, so a test can pass fakes.

Register it inside **your slice's block**:

```java
// SceneId.java
POKEDEX,

// SceneFactory.registerAll()
register(SceneId.POKEDEX, () -> new edu.csumb.cst338.scenes.PokedexView().getRoot());
```

> **Reminder: the window freezes during Import.** The fetch runs on the JavaFX Application
> Thread. Moving it to a `Task` is enhancement B1; see [`TASK_GUIDE.md`](TASK_GUIDE.md).

---

## Step 7. Test the scene (TestFX)

The prompt requires one UI test. This one uses a fake API and an in-memory database.

`src/test/java/edu/csumb/cst338/scenes/PokedexViewTest.java`

```java
package edu.csumb.cst338.scenes;

import static org.junit.jupiter.api.Assertions.assertEquals;
import static org.junit.jupiter.api.Assertions.assertTrue;

import edu.csumb.cst338.api.ApiException;
import edu.csumb.cst338.api.PokemonImporter;
import edu.csumb.cst338.api.PokemonService;
import edu.csumb.cst338.dao.PokemonDao;
import edu.csumb.cst338.db.Database;
import edu.csumb.cst338.model.Pokemon;
import java.util.List;
import javafx.scene.Scene;
import javafx.scene.control.Label;
import javafx.scene.control.ListView;
import javafx.scene.control.TextField;
import javafx.stage.Stage;
import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.extension.ExtendWith;
import org.testfx.api.FxRobot;
import org.testfx.framework.junit5.ApplicationExtension;
import org.testfx.framework.junit5.Start;

/** The Pokedex scene with a fake API: the list shows what is in the table. */
@ExtendWith(ApplicationExtension.class)
class PokedexViewTest {

    private static final String PIKACHU_JSON = "{\"id\":25,\"name\":\"pikachu\",\"height\":4,"
            + "\"weight\":60,\"types\":[{\"slot\":1,\"type\":{\"name\":\"electric\",\"url\":\"u\"}}]}";

    private PokemonDao dao;

    @Start
    void start(Stage stage) {
        Database.useInMemory();
        Database.reset();
        Database.init(false);
        dao = new PokemonDao();
        dao.save(new Pokemon(6, "charizard", 17, 905, List.of("fire", "flying")));
        PokemonService fake = new PokemonService(url -> {
            if (url.endsWith("/pikachu")) {
                return PIKACHU_JSON;
            }
            throw new ApiException("The service rejected the request (404).", 404, false);
        }, "http://fake");
        stage.setScene(new Scene(new PokedexView(new PokemonImporter(fake, dao), dao).getRoot()));
        stage.show();
    }

    @SuppressWarnings("unchecked")
    private ListView<Pokemon> list(FxRobot robot) {
        return robot.lookup(".list-view").queryAs(ListView.class);
    }

    /** Sets the text directly. Robot typing is very slow on a headless display. */
    private void type(FxRobot robot, String text) {
        robot.interact(() -> robot.lookup("#nameField").queryAs(TextField.class).setText(text));
    }

    @Test
    void showsWhatIsAlreadyStored(FxRobot robot) {
        assertEquals(1, list(robot).getItems().size());
    }

    @Test
    void importAddsToTheList(FxRobot robot) {
        type(robot, "pikachu");
        robot.clickOn("Import");
        assertEquals(2, list(robot).getItems().size());
        assertEquals("pikachu", list(robot).getItems().get(1).name());
    }

    @Test
    void failedImportShowsTheMessageAndKeepsTheList(FxRobot robot) {
        type(robot, "pikachoo");
        robot.clickOn("Import");
        Label status = robot.lookup("#status").queryAs(Label.class);
        assertTrue(status.getText().contains("404"), status.getText());
        assertEquals(1, list(robot).getItems().size());
    }
}
```

> **Do not use `robot.write(...)` to type.** On the headless test display it can take minutes.
> Set the text with `robot.interact(...)`, as `type(...)` does.

---

## Step 8. Prove it works offline

1. `./gradlew run -Pdev`, open **Pokedex**, import `pikachu`, then `charizard`.
2. Close the app.
3. Turn off Wi-Fi.
4. `./gradlew run -Pdev` again and open **Pokedex**. Both are listed.

That is the Part 4 video's "works after import" moment.

**Going further (not required):**

| Want | Do |
|---|---|
| Types as real rows | A `pokemon_types (pokemon_id, type_name)` table and a `JOIN` (enhancement A3) |
| Show how old the data is | Read `imported_at`; re-fetch only when it is older than a limit (A5) |
| All-or-nothing import | One connection, `setAutoCommit(false)`, `commit()` or `rollback()` (B9) |
| Keep the API's shape out of your tables | Already done: `PokemonJson` vs `Pokemon` (B7) |

---

## Step 9. Do it for your project

1. Add your table to **your block** of `schema.sql`. Pick the key (Step 1 table).
2. Write `XxxDao`: `save`, `findById`, `findAll`, plus what your scenes need.
3. Write `XxxDaoTest` with the three-line `setUp`, at least half negative cases.
4. Write `XxxImporter` that fetches first, then saves.
5. Test it with a fake GET lambda.
6. Point your scene at the DAO, not the API.
7. Fill in `TESTING.md`: DAO insert and read, update or delete, negative case.

---

## Troubleshooting

| Symptom | Cause | Fix |
|---|---|---|
| `Failed running /schema.sql: Syntax error` | SQL typo | The message names the position. Missing comma before the next column is common |
| `Table "POKEMON" not found` | `schema.sql` did not run, or the table is in a comment | Check the `CREATE TABLE` is not inside `--` lines |
| `Unique index or primary key violation` | Same key inserted twice with `INSERT` | Use `MERGE ... KEY(...)`, or check first |
| Tests pass alone, fail together | A test skipped `Database.reset()` | Use the full three-line `setUp` |
| Real data vanished after tests | A test did not call `useInMemory()` | Add it as the first line of `setUp` |
| List empty after Import | Scene did not reload | Call `reload()` after a successful import |
| `NullPointerException` in `map` | A column is `NULL` | Add `NOT NULL` to the column, or check `rs.wasNull()` |

---

## Links

- JDBC tutorial, `PreparedStatement`: <https://docs.oracle.com/javase/tutorial/jdbc/basics/prepared.html>
- try-with-resources: <https://docs.oracle.com/javase/tutorial/essential/exceptions/tryResourceClose.html>
- H2 SQL commands: <https://h2database.com/html/commands.html>
- Next: [`FAILURE_TEST_GUIDE.md`](FAILURE_TEST_GUIDE.md) tests the import when the API fails.
- In your team's repo: `UserDao.java` and `UserDaoTest.java` (the pattern), `ARCHITECTURE.md`
  "The API import path".
