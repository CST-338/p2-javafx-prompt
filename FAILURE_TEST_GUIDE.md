# Testing the API Failure Path

Follow the steps in order. This guide continues [`DAO_GUIDE.md`](DAO_GUIDE.md). When you finish,
one test class proves that every way the API can fail produces a readable message **and
leaves your database untouched**. The test report (`build/reports/tests/test/index.html`)
lists nine passing tests:

| Test | Failure |
|---|---|
| `notFoundIsPermanent` | 404 |
| `serverErrorIsRetryable` | 500 |
| `rateLimitedIsRetryableAndSaysSo` | 429 |
| `htmlInsteadOfJsonIsAReadableMessage` | 200 with HTML |
| `truncatedJsonIsAReadableMessage` | 200 with half the JSON |
| `slowServerTimesOut` | no answer in time |
| `noServerAtAllIsAReadableMessage` | nothing listening |
| `oneFailureInABatchSavesNothing` | one 404 among good answers |
| `theHappyPathStillWorks` | 200, to prove the stub works |

**Why.** README API rule 3: "A timeout, a non-200 and a malformed body each produce a message a
person can read. **At least one test covers a failure path.**" It is the most heavily graded
part of the API requirement.

**Why not test against the real API?** You cannot make PokeAPI return a 500 on demand, and a
test that needs the internet fails for reasons unrelated to your code.

---

## Step 1. Know the three ways to fake the API

| Level | Fake | Tests | Where you have seen it |
|---|---|---|---|
| Parser | A JSON string in the test | Malformed and incomplete bodies | `API_GUIDE.md` Step 6 |
| Service | A lambda `url -> body` | Your logic when a fetch throws | `DAO_GUIDE.md` Step 5 |
| HTTP | A **stub server** on `localhost` | Real status codes, real timeouts, real sockets | `ApiClientTest.java` |

This guide uses the **stub server**, because it is the only level that exercises `ApiClient`
itself: the status-code handling, the timeout and the connection failure. The stub is
`com.sun.net.httpserver.HttpServer`, which ships with the JDK. Nothing to add to `build.gradle`.

A stub server in tests is enhancement **B6**. The required failure-path test needs only one
of these tests; the full set is what B6 asks for.

---

## Step 2. List the failures your API can produce

Write them down before writing tests. For a JSON REST API:

| Failure | What the server does | What your app must do | `ApiException` |
|---|---|---|---|
| Not found | 404 | Say so; do not retry | status 404, not retryable |
| Server error | 500, 502, 503 | Say "try later" | status 5xx, retryable |
| Rate limited | 429 | Say "wait" | status 429, retryable |
| Wrong content | 200 with HTML or garbage | Say the data was unreadable | parser's message |
| Cut-off body | 200 with half the JSON | Same | parser's message |
| Too slow | Never answers in time | Give up after the timeout | status 0, retryable |
| No server | Connection refused | Say "check your connection" | status 0, retryable |
| One bad item in a batch | 200, 200, 404 | Save **nothing** | as above |

Check your API's documentation for others (401 for a bad key is common).

---

## Step 3. Write the test

`src/test/java/edu/csumb/cst338/api/PokemonImportFailureTest.java`

```java
package edu.csumb.cst338.api;

import static org.junit.jupiter.api.Assertions.assertEquals;
import static org.junit.jupiter.api.Assertions.assertFalse;
import static org.junit.jupiter.api.Assertions.assertThrows;
import static org.junit.jupiter.api.Assertions.assertTrue;

import com.sun.net.httpserver.HttpServer;
import edu.csumb.cst338.dao.PokemonDao;
import edu.csumb.cst338.db.Database;
import java.io.IOException;
import java.io.OutputStream;
import java.net.InetSocketAddress;
import java.nio.charset.StandardCharsets;
import java.time.Duration;
import java.util.List;
import org.junit.jupiter.api.AfterEach;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

/**
 * The import path when things go wrong. A stub server inside the test plays PokeAPI,
 * so every failure is on demand and no test needs the internet.
 */
class PokemonImportFailureTest {

    private static final int ANY_FREE_PORT = 0;
    private static final Duration SHORT_TIMEOUT = Duration.ofMillis(300);
    private static final long SLOWER_THAN_TIMEOUT_MS = 2_000;
    private static final String GOOD_BODY = "{\"id\":25,\"name\":\"pikachu\",\"height\":4,"
            + "\"weight\":60,\"types\":[{\"slot\":1,\"type\":{\"name\":\"electric\",\"url\":\"u\"}}]}";

    private HttpServer server;
    private String baseUrl;
    private PokemonDao dao;

    @BeforeEach
    void setUp() throws IOException {
        Database.useInMemory();
        Database.reset();
        Database.init(false);
        dao = new PokemonDao();

        server = HttpServer.create(new InetSocketAddress(ANY_FREE_PORT), 0);
        stub("/pokemon/pikachu", 200, GOOD_BODY);
        stub("/pokemon/missing", 404, "Not Found");
        stub("/pokemon/broken", 500, "Internal Server Error");
        stub("/pokemon/busy", 429, "Too Many Requests");
        stub("/pokemon/garbage", 200, "<html>this is not json</html>");
        stub("/pokemon/truncated", 200, "{\"id\":25,\"name\":\"pika");
        server.createContext("/pokemon/slow", exchange -> {
            try {
                Thread.sleep(SLOWER_THAN_TIMEOUT_MS);
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
            }
            exchange.close();
        });
        server.start();
        baseUrl = "http://localhost:" + server.getAddress().getPort();
    }

    @AfterEach
    void tearDown() {
        server.stop(0);
    }

    /** Registers a path that answers with this status and body. */
    private void stub(String path, int status, String body) {
        server.createContext(path, exchange -> {
            byte[] bytes = body.getBytes(StandardCharsets.UTF_8);
            exchange.sendResponseHeaders(status, bytes.length);
            try (OutputStream out = exchange.getResponseBody()) {
                out.write(bytes);
            }
        });
    }

    private PokemonImporter importer() {
        ApiClient client = new ApiClient("test", SHORT_TIMEOUT, SHORT_TIMEOUT);
        return new PokemonImporter(new PokemonService(client::getJson, baseUrl), dao);
    }

    /** Runs a failing import and returns the exception, after checking nothing was saved. */
    private ApiException failedImport(String... names) {
        ApiException e = assertThrows(ApiException.class, () -> importer().importAll(List.of(names)));
        assertEquals(0, dao.count(), "a failed import must save nothing");
        return e;
    }

    @Test
    void notFoundIsPermanent() {
        ApiException e = failedImport("missing");
        assertEquals(404, e.getStatusCode());
        assertFalse(e.isRetryable());
    }

    @Test
    void serverErrorIsRetryable() {
        ApiException e = failedImport("broken");
        assertEquals(500, e.getStatusCode());
        assertTrue(e.isRetryable());
    }

    @Test
    void rateLimitedIsRetryableAndSaysSo() {
        ApiException e = failedImport("busy");
        assertTrue(e.isRetryable());
        assertTrue(e.getMessage().contains("rate limiting"), e.getMessage());
    }

    @Test
    void htmlInsteadOfJsonIsAReadableMessage() {
        ApiException e = failedImport("garbage");
        assertEquals("The service sent data this app could not read.", e.getMessage());
    }

    @Test
    void truncatedJsonIsAReadableMessage() {
        failedImport("truncated");
    }

    @Test
    void slowServerTimesOut() {
        ApiException e = failedImport("slow");
        assertEquals(0, e.getStatusCode(), "a timeout never got a status");
        assertTrue(e.isRetryable());
    }

    @Test
    void noServerAtAllIsAReadableMessage() {
        server.stop(0);
        ApiException e = failedImport("pikachu");
        assertTrue(e.getMessage().startsWith("Could not reach"), e.getMessage());
    }

    @Test
    void oneFailureInABatchSavesNothing() {
        failedImport("pikachu", "missing");
    }

    @Test
    void theHappyPathStillWorks() {
        assertEquals(1, importer().importAll(List.of("pikachu")));
        assertEquals(1, dao.count());
    }
}
```

How it works:

| Part | What it does |
|---|---|
| `HttpServer.create(new InetSocketAddress(0), 0)` | Port `0` means "any free port", so tests never collide |
| `stub(path, status, body)` | Each path answers with a fixed status and body |
| `/pokemon/slow` | Sleeps longer than the client's timeout, then hangs up |
| `new ApiClient("test", SHORT_TIMEOUT, SHORT_TIMEOUT)` | `ApiClient`'s second constructor. A 300 ms timeout keeps the test fast |
| `failedImport(...)` | Asserts the exception **and** that the table is still empty |
| `server.stop(0)` in a test | Nothing is listening: the "no network" case |
| `theHappyPathStillWorks` | Proves the stub is set up right, so the failures are real failures |

Run:

```bash
./gradlew test --tests '*PokemonImportFailureTest'
```

Expect `BUILD SUCCESSFUL` in a few seconds.

---

## Step 4. Watch one fail on purpose

A test you have never seen fail proves little. **Commit your work first**, then break the code,
run, and undo:

1. In `PokemonParser.parse`, delete `|| raw.name() == null` from the `if`.
2. Run the tests. Nothing should change: this test class never sends JSON without a name.
   That is a gap. `PokemonParserTest.missingNameIsAnApiException` covers it.
3. In `PokemonImporter.importAll`, swap the order: save inside the first loop.
4. Run the tests. `oneFailureInABatchSavesNothing` fails with
   `a failed import must save nothing ==> expected: <0> but was: <1>`.
5. Undo both changes, and only those two files:

   ```bash
   git restore src/main/java/edu/csumb/cst338/api/PokemonParser.java \
               src/main/java/edu/csumb/cst338/api/PokemonImporter.java
   ```

---

## Step 5. Show the message, not the stack trace

The test proves the message is readable. The scene must show it. In your scene's handler:

```java
try {
    int saved = importer.importAll(List.of(name));
    status.setText("Imported " + saved + ".");
    reload();
} catch (ApiException | DataAccessException ex) {
    status.setText(ex.getMessage());
}
```

`PokedexViewTest.failedImportShowsTheMessageAndKeepsTheList` (in `DAO_GUIDE.md` Step 7) tests
this in the UI.

If you want to tell the user whether to try again:

```java
String advice = ex.isRetryable() ? " Try again in a moment." : "";
status.setText(ex.getMessage() + advice);
```

Using `isRetryable()` to retry automatically, with waits that grow, is enhancement **B2**.

---

## Step 6. Do it for your project

1. List your API's failures (Step 2 table). Add any your API documents.
2. Copy the test class. Change the paths and the good body to your API's.
3. One `@Test` per failure, each asserting the exception **and** an untouched table.
4. Keep one happy-path test, so you know the stub works.
5. Fill in the `TESTING.md` row "API failure path" with the test names.

---

## Troubleshooting

| Symptom | Cause | Fix |
|---|---|---|
| `Connection refused` in every test | Server not started | `server.start()` after the `stub(...)` calls, in `@BeforeEach` |
| Timeout test is slow, or does not time out | The client timeout is longer than the stub's sleep | Pass a short timeout to the three-argument constructor; sleep longer than it |
| `Address already in use` | Fixed port number | Use port `0` |
| 404 for a path you stubbed | Path mismatch | `createContext` matches by prefix; print `baseUrl + path` and compare |
| Test passes, but the scene shows a stack trace | Scene catches nothing, or prints `e` | `catch (ApiException ex)` and show `ex.getMessage()` |
| Test hangs | Handler never responds or closes | Every handler calls `sendResponseHeaders` or `exchange.close()` |

---

## Links

- `HttpServer` (the stub): <https://docs.oracle.com/en/java/javase/25/docs/api/jdk.httpserver/com/sun/net/httpserver/HttpServer.html>
- JUnit 5 `assertThrows`: <https://docs.junit.org/5.11.3/user-guide/#writing-tests-assertions>
- HTTP status codes: <https://developer.mozilla.org/en-US/docs/Web/HTTP/Status>
- In your team's repo: `ApiClientTest.java` (the stub pattern), `ApiClient.java` (which status
  is retryable).
