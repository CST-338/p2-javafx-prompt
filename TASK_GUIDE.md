# Importing in the Background with `Task` (Enhancement B1)

Follow the steps in order. This guide continues [`DAO_GUIDE.md`](DAO_GUIDE.md), where the
**Import** button freezes the window until the API answers. When you finish:

- the window stays responsive during an import,
- a progress bar fills as each item arrives,
- **Cancel** stops the import and saves nothing, and
- a TestFX test proves all three.

B1, from the prompt: "Background import off the JavaFX Application Thread (`Task`,
`Platform.runLater`), UI responsive, with progress and a working cancel."

---

## Step 1. Know the one rule

JavaFX has one thread that draws the window and runs every event handler: the **JavaFX
Application Thread**. While a handler runs, nothing is drawn and no click is handled.

| Code | Runs on | Allowed to |
|---|---|---|
| `setOnAction(e -> ...)` | JavaFX thread | touch the UI. **Must not** wait on the network |
| `Task.call()` | a background thread | wait on the network, use the DAO. **Must not** touch the UI |
| `setOnSucceeded`, `setOnFailed`, `setOnCancelled` | JavaFX thread | touch the UI |
| `updateProgress`, `updateMessage` (inside `call`) | either | safe: `Task` forwards them to the JavaFX thread |

Touching a `Label` from `call()` either throws `IllegalStateException: Not on FX application
thread` or fails at random. `Task` exists so you never have to.

Reference: [`javafx.concurrent.Task`](https://openjfx.io/javadoc/25/javafx.graphics/javafx/concurrent/Task.html),
[Concurrency in JavaFX](https://docs.oracle.com/javafx/2/threads/jfxpub-threads.htm).

---

## Step 2. Let the importer report progress and stop

`Task` can report progress and be cancelled, but only between steps your code takes. Give
`importAll` a way to say "one more done" and to ask "should I stop?". The original
one-argument method now delegates to the new one, so nothing else changes.

`src/main/java/edu/csumb/cst338/api/PokemonImporter.java`

```java
package edu.csumb.cst338.api;

import edu.csumb.cst338.dao.PokemonDao;
import edu.csumb.cst338.model.Pokemon;
import java.util.ArrayList;
import java.util.List;
import java.util.concurrent.CancellationException;
import java.util.function.BiConsumer;
import java.util.function.BooleanSupplier;

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
        return importAll(names, (done, total) -> { }, () -> false);
    }

    /**
     * Same as {@link #importAll(List)}, reporting progress and checking for a cancel.
     *
     * @param names     the names to import
     * @param progress  called after each fetch with (done, total)
     * @param cancelled checked before each fetch; true stops the import
     * @return how many were saved
     * @throws ApiException          if any fetch fails; nothing is saved
     * @throws CancellationException if cancelled; nothing is saved
     */
    public int importAll(List<String> names, BiConsumer<Integer, Integer> progress,
                         BooleanSupplier cancelled) {
        List<Pokemon> fetched = new ArrayList<>();
        for (String name : names) {
            if (cancelled.getAsBoolean()) {
                throw new CancellationException("Import cancelled");
            }
            fetched.add(service.fetch(name));
            progress.accept(fetched.size(), names.size());
        }
        if (cancelled.getAsBoolean()) {
            throw new CancellationException("Import cancelled");
        }
        for (Pokemon p : fetched) {
            dao.save(p);
        }
        return fetched.size();
    }
}
```

| Parameter | Type | The `Task` passes |
|---|---|---|
| `progress` | `BiConsumer<Integer, Integer>` | `this::updateProgress` |
| `cancelled` | `BooleanSupplier` | `this::isCancelled` |

The importer still knows nothing about JavaFX, so it is still testable without a window.

Add two tests to `PokemonImporterTest`:

```java
@Test
void progressIsReportedAfterEachFetch() {
    List<String> reports = new ArrayList<>();
    importer(FAKE_GET).importAll(List.of("pikachu", "charizard"),
            (done, total) -> reports.add(done + "/" + total), () -> false);
    assertEquals(List.of("1/2", "2/2"), reports);
}

@Test
void cancellingStopsAndSavesNothing() {
    List<String> fetched = new ArrayList<>();
    Function<String, String> counting = url -> {
        fetched.add(url);
        return FAKE_GET.apply(url);
    };
    assertThrows(CancellationException.class, () -> importer(counting).importAll(
            List.of("pikachu", "charizard"), (done, total) -> { }, () -> !fetched.isEmpty()));
    assertEquals(1, fetched.size(), "stopped before the second fetch");
    assertEquals(0, dao.count());
}
```

Imports to add: `java.util.ArrayList`, `java.util.concurrent.CancellationException`.

---

## Step 3. Move the import into a `Task`

Replace `PokedexView` with this version. What changed from `DAO_GUIDE.md`: a progress bar, a
Cancel button, and `handleImport` builds a `Task` instead of calling the importer directly. The
field accepts several names: `pikachu, charizard, bulbasaur`.

`src/main/java/edu/csumb/cst338/scenes/PokedexView.java`

```java
package edu.csumb.cst338.scenes;

import edu.csumb.cst338.api.ApiClient;
import edu.csumb.cst338.api.ApiConfig;
import edu.csumb.cst338.api.PokemonImporter;
import edu.csumb.cst338.api.PokemonService;
import edu.csumb.cst338.dao.PokemonDao;
import edu.csumb.cst338.model.Pokemon;
import java.util.Arrays;
import java.util.List;
import javafx.concurrent.Task;
import javafx.geometry.Insets;
import javafx.scene.Parent;
import javafx.scene.control.Button;
import javafx.scene.control.Label;
import javafx.scene.control.ListView;
import javafx.scene.control.ProgressBar;
import javafx.scene.control.TextField;
import javafx.scene.layout.HBox;
import javafx.scene.layout.VBox;

/** The Pokedex with the import on a background thread, a progress bar and a Cancel button. */
public class PokedexView {

    private static final int SPACING = 10;
    private static final int PADDING = 20;
    private static final String NAME_SEPARATOR = ",";
    private static final String THREAD_NAME = "pokemon-import";

    private final PokemonImporter importer;
    private final PokemonDao dao;
    private final TextField nameField = new TextField();
    private final ListView<Pokemon> list = new ListView<>();
    private final Label status = new Label();
    private final ProgressBar progress = new ProgressBar(0);
    private final Button importButton = new Button("Import");
    private final Button cancelButton = new Button("Cancel");

    private Task<Integer> running;

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
        nameField.setPromptText("pikachu, charizard");
        status.setId("status");
        progress.setId("progress");
        importButton.setId("importButton");
        cancelButton.setId("cancelButton");
        cancelButton.setDisable(true);
        importButton.setOnAction(e -> handleImport());
        cancelButton.setOnAction(e -> handleCancel());
        reload();
        VBox root = new VBox(SPACING,
                new HBox(SPACING, nameField, importButton, cancelButton), progress, status, list);
        root.setPadding(new Insets(PADDING));
        return root;
    }

    private void handleImport() {
        List<String> names = Arrays.stream(nameField.getText().split(NAME_SEPARATOR))
                .map(name -> name.trim().toLowerCase())
                .filter(name -> !name.isEmpty())
                .toList();

        // 1. Describe the work. call() runs on the background thread: no UI code inside it.
        Task<Integer> task = new Task<>() {
            @Override
            protected Integer call() {
                return importer.importAll(names, this::updateProgress, this::isCancelled);
            }
        };

        // 2. Say what happens when it ends. These run on the JavaFX thread: UI code is fine.
        task.setOnSucceeded(e -> {
            status.setText("Imported " + task.getValue() + ".");
            reload();
            finished();
        });
        task.setOnFailed(e -> {
            status.setText(task.getException().getMessage());
            finished();
        });
        task.setOnCancelled(e -> {
            status.setText("Import cancelled.");
            finished();
        });

        // 3. Wire the UI to it, then start it on its own thread.
        progress.progressProperty().bind(task.progressProperty());
        status.setText("Importing...");
        importButton.setDisable(true);
        cancelButton.setDisable(false);
        running = task;
        Thread thread = new Thread(task, THREAD_NAME);
        thread.setDaemon(true);
        thread.start();
    }

    private void handleCancel() {
        if (running != null) {
            running.cancel();
        }
    }

    private void finished() {
        progress.progressProperty().unbind();
        importButton.setDisable(false);
        cancelButton.setDisable(true);
        running = null;
    }

    private void reload() {
        list.getItems().setAll(dao.findAll());
    }
}
```

`handleImport`, in three parts:

| Part | Code | Thread |
|---|---|---|
| 1. Describe the work | `new Task<>() { protected Integer call() {...} }` | `call()` runs in the background |
| 2. Say what happens at the end | `setOnSucceeded`, `setOnFailed`, `setOnCancelled` | JavaFX thread |
| 3. Wire and start | `progressProperty().bind(...)`, `new Thread(task).start()` | JavaFX thread |

Details that matter:

| Detail | Why |
|---|---|
| `thread.setDaemon(true)` | Closing the window ends the app even if an import is running |
| `task.getException().getMessage()` in `setOnFailed` | The `ApiException` from `call()` arrives here, with its readable message |
| Import disabled while running | Prevents two imports at once |
| `progress.progressProperty().unbind()` in `finished()` | A bound property cannot be set; unbinding releases it for the next import |
| `running.cancel()` | Sets `isCancelled()`, and interrupts the thread, which ends a blocked HTTP call |

> **Why does Cancel save nothing?** `importAll` checks `cancelled` before every fetch and once
> more before saving, and throws `CancellationException`. Fetch-then-save means there is
> nothing half-saved to undo.

---

## Step 4. Test it: responsive, progress, cancel

**Replace** `PokedexViewTest` from `DAO_GUIDE.md`. Its tests assert right after the click, which
no longer works: the import now finishes later, on another thread.

The fake API waits on a `CountDownLatch` (a gate), so the test controls exactly when each fetch
finishes. While the fetch is stuck, the test types into the field: if the import were on the
JavaFX thread, that would hang.

`src/test/java/edu/csumb/cst338/scenes/PokedexViewTest.java`

```java
package edu.csumb.cst338.scenes;

import static org.junit.jupiter.api.Assertions.assertEquals;
import static org.junit.jupiter.api.Assertions.assertFalse;
import static org.junit.jupiter.api.Assertions.assertTrue;

import edu.csumb.cst338.api.ApiException;
import edu.csumb.cst338.api.PokemonImporter;
import edu.csumb.cst338.api.PokemonService;
import edu.csumb.cst338.dao.PokemonDao;
import edu.csumb.cst338.db.Database;
import java.util.concurrent.Callable;
import java.util.concurrent.CountDownLatch;
import java.util.concurrent.TimeUnit;
import java.util.concurrent.TimeoutException;
import javafx.scene.Scene;
import javafx.scene.control.Button;
import javafx.scene.control.Label;
import javafx.scene.control.ListView;
import javafx.scene.control.ProgressBar;
import javafx.scene.control.TextField;
import javafx.stage.Stage;
import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.extension.ExtendWith;
import org.testfx.api.FxRobot;
import org.testfx.framework.junit5.ApplicationExtension;
import org.testfx.framework.junit5.Start;
import org.testfx.util.WaitForAsyncUtils;

/** The import runs on a background thread: the window stays usable, and Cancel works. */
@ExtendWith(ApplicationExtension.class)
class PokedexViewTest {

    private static final long WAIT_SECONDS = 10;
    private static final String JSON = "{\"id\":%d,\"name\":\"%s\",\"height\":1,\"weight\":1,"
            + "\"types\":[{\"slot\":1,\"type\":{\"name\":\"electric\",\"url\":\"u\"}}]}";

    /** Every fetch waits here until the test opens the gate. */
    private final CountDownLatch gate = new CountDownLatch(1);
    private PokemonDao dao;

    @Start
    void start(Stage stage) {
        Database.useInMemory();
        Database.reset();
        Database.init(false);
        dao = new PokemonDao();
        PokemonService gated = new PokemonService(url -> {
            try {
                gate.await();
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
                throw new ApiException("The request was interrupted.", 0, true, e);
            }
            String name = url.substring(url.lastIndexOf('/') + 1);
            if (name.equals("pikachoo")) {
                throw new ApiException("The service rejected the request (404).", 404, false);
            }
            return String.format(JSON, name.length(), name);
        }, "http://fake");
        stage.setScene(new Scene(
                new PokedexView(new PokemonImporter(gated, dao), dao).getRoot()));
        stage.show();
    }

    /** Sets the text directly. Robot typing is very slow on a headless display. */
    private void type(FxRobot robot, String text) {
        robot.interact(() -> robot.lookup("#nameField").queryAs(TextField.class).setText(text));
    }

    private Button button(FxRobot robot, String id) {
        return robot.lookup(id).queryAs(Button.class);
    }

    private String status(FxRobot robot) {
        return robot.lookup("#status").queryAs(Label.class).getText();
    }

    private void waitUntil(Callable<Boolean> condition) throws TimeoutException {
        WaitForAsyncUtils.waitFor(WAIT_SECONDS, TimeUnit.SECONDS, condition);
    }

    @Test
    void windowStaysUsableWhileImporting(FxRobot robot) throws TimeoutException {
        type(robot, "pikachu, raichu");
        robot.clickOn("#importButton");
        // The fetch is stuck on the gate, yet the JavaFX thread still answers.
        assertTrue(button(robot, "#importButton").isDisabled());
        assertFalse(button(robot, "#cancelButton").isDisabled());
        type(robot, "the UI still responds");
        assertEquals("the UI still responds",
                robot.lookup("#nameField").queryAs(TextField.class).getText());

        gate.countDown();
        waitUntil(() -> status(robot).startsWith("Imported"));
        assertEquals("Imported 2.", status(robot));
        assertEquals(1.0, robot.lookup("#progress").queryAs(ProgressBar.class).getProgress());
        assertEquals(2, robot.lookup(".list-view").queryAs(ListView.class).getItems().size());
        assertFalse(button(robot, "#importButton").isDisabled());
        assertTrue(button(robot, "#cancelButton").isDisabled());
    }

    @Test
    void failureShowsTheMessage(FxRobot robot) throws TimeoutException {
        type(robot, "pikachoo");
        robot.clickOn("#importButton");
        gate.countDown();
        waitUntil(() -> status(robot).contains("404"));
        assertEquals(0, dao.count());
        assertFalse(button(robot, "#importButton").isDisabled());
    }

    @Test
    void cancelSavesNothing(FxRobot robot) throws TimeoutException {
        type(robot, "pikachu, raichu");
        robot.clickOn("#importButton");
        robot.clickOn("#cancelButton");
        waitUntil(() -> status(robot).equals("Import cancelled."));
        gate.countDown();
        WaitForAsyncUtils.waitForFxEvents();
        assertEquals(0, dao.count());
        assertFalse(button(robot, "#importButton").isDisabled());
        assertTrue(button(robot, "#cancelButton").isDisabled());
    }
}
```

| Tool | What it does |
|---|---|
| `CountDownLatch gate` | Every fake fetch waits until `gate.countDown()` |
| `WaitForAsyncUtils.waitFor(10, SECONDS, condition)` | Waits for the background thread and the JavaFX thread to finish, up to 10 s |
| `robot.interact(...)` | Runs code on the JavaFX thread and waits for it |

Run `./gradlew test`. Expect `BUILD SUCCESSFUL` in a few seconds.

> **Do not use `Thread.sleep` to wait in a test.** Too short and the test fails at random; too
> long and the suite is slow. Wait for a condition, as `waitUntil` does.

---

## Step 5. Try it by hand

1. `./gradlew run -Pdev`, open **Pokedex**.
2. Type `pikachu, charizard, bulbasaur, squirtle, eevee` and click **Import**.
3. While it runs, drag the window and type in the field: both work. The bar fills in steps.
4. Import again and click **Cancel** quickly: "Import cancelled.", and no new rows.
5. Turn off Wi-Fi and Import: the `ApiException` message appears; the list is unchanged.

For your Part 4 video, show step 3 and step 4.

---

## Step 6. Do it for your project

1. Give your importer `progress` and `cancelled` parameters (Step 2), with tests.
2. In your scene: `Task`, three handlers, bind the progress bar, daemon thread (Step 3).
3. Disable the start button while running; enable Cancel only while running.
4. TestFX test with a gated fake: responsive, finishes, fails, cancels (Step 4).
5. In `DESIGN.md`, name B1 as your Tier B enhancement and point to the test.

---

## Troubleshooting

| Symptom | Cause | Fix |
|---|---|---|
| `IllegalStateException: Not on FX application thread` | UI code inside `call()` | Move it to `setOnSucceeded`, or use `updateMessage` |
| Window still freezes | Called `task.run()`, or did the work in the handler | `new Thread(task).start()`; the work goes inside `call()` |
| Progress bar never moves | `call()` never calls `updateProgress`, or the bar is not bound | Pass `this::updateProgress`; bind `progressProperty()` |
| `A bound value cannot be set` | Setting a bound property | `unbind()` first, as `finished()` does |
| Cancel does nothing | `call()` never checks `isCancelled()` | Pass `this::isCancelled` and check it between steps |
| Error is swallowed | No `setOnFailed` | Always set all three handlers |
| App keeps running after the window closes | Non-daemon thread | `thread.setDaemon(true)` |
| UI test fails at random | `Thread.sleep` instead of waiting | `WaitForAsyncUtils.waitFor(...)` with a condition |

---

## Links

- `Task`: <https://openjfx.io/javadoc/25/javafx.graphics/javafx/concurrent/Task.html>
- `Worker` (states, progress): <https://openjfx.io/javadoc/25/javafx.graphics/javafx/concurrent/Worker.html>
- `Platform.runLater`: <https://openjfx.io/javadoc/25/javafx.graphics/javafx/application/Platform.html#runLater(java.lang.Runnable)>
- TestFX `WaitForAsyncUtils`: <https://testfx.github.io/TestFX/docs/javadoc/testfx-core/javadoc/org.testfx/org/testfx/util/WaitForAsyncUtils.html>
- Oracle tutorial, Concurrency in JavaFX: <https://docs.oracle.com/javafx/2/threads/jfxpub-threads.htm>
