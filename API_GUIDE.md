# Reading a REST API into Java Objects (Gson)

Follow the steps in order. The example uses [PokeAPI](https://pokeapi.co/), which needs no key.
When you finish, this line prints from your own code:

```
Pokemon[id=25, name=pikachu, height=4, weight=60, types=[electric]]
```

Then swap in your project's API and fields.

**Work on a practice branch you will not merge.** Steps 3 and 8 edit files your team shares
(`api.properties`, `SceneId`, `SceneFactory`). Start with:

```bash
git switch main
git pull
git switch -c practice/pokeapi
```

When you are done, `git switch main` and copy only what you need into your slice's branch.

**What you already have.** `ApiClient` does the HTTP GET and turns timeouts and bad statuses
into an `ApiException`. `ApiConfig` reads URLs from `api.properties`. This guide adds the
missing piece: **JSON text → Java object**.

**Tools used**

| Tool | Version | Docs |
|---|---|---|
| Gson | 2.15.0 | <https://github.com/google/gson/blob/main/UserGuide.md> |
| `java.net.http.HttpClient` (inside `ApiClient`) | JDK 25 | <https://docs.oracle.com/en/java/javase/25/docs/api/java.net.http/java/net/http/HttpClient.html> |
| OkHttp (optional, [last section](#optional-okhttp-instead-of-javanethttp)) | 5.5.1 | <https://github.com/square/okhttp> |

> Why Gson and records? Gson is small and needs no setup. Since Gson 2.10 it fills in Java
> `record`s directly, so each JSON shape is a one-line type. Jackson is the other common choice
> and is more powerful, but it brings more setup than this project needs.

---

## Step 1. Add Gson to `build.gradle`

Add the version next to the other version constants at the top:

```groovy
def GSON_VERSION = '2.15.0'
```

Add the dependency inside `dependencies { ... }`:

```groovy
implementation "com.google.code.gson:gson:${GSON_VERSION}"
```

Run:

```bash
./gradlew build
```

Expect `BUILD SUCCESSFUL`. In IntelliJ, click the Gradle elephant icon ("Load Gradle Changes").

To find the newest version, see <https://github.com/google/gson/releases>.

> This project has no `module-info.java`, so you do **not** need `requires com.google.gson`.

---

## Step 2. Look at the JSON before writing any code

Open this in a browser: <https://pokeapi.co/api/v2/pokemon/pikachu>

Firefox shows it formatted. In Chrome, tick "Pretty-print". The response is long. You need
only these parts:

```json
{
  "id": 25,
  "name": "pikachu",
  "height": 4,
  "weight": 60,
  "types": [
    { "slot": 1, "type": { "name": "electric", "url": "https://pokeapi.co/api/v2/type/13/" } }
  ]
}
```

Write down three things:

1. The **fields** you need, and their JSON types (number, string, object, array).
2. The **nesting**. The type name is at `types[i].type.name`: an array of objects, each
   holding another object.
3. The **exact key spelling**, including any `snake_case`.

Endpoint reference: <https://pokeapi.co/docs/v2> (section "Pokémon")

---

## Step 3. Put the base URL in `api.properties`

`src/main/resources/api.properties`:

```properties
api.baseUrl=https://pokeapi.co/api/v2
```

Never type a URL into a scene (README, API rule 4).

---

## Step 4. Write records that mirror the JSON

`src/main/java/edu/csumb/cst338/api/PokemonJson.java`

```java
package edu.csumb.cst338.api;

import java.util.List;

/**
 * The shape of PokeAPI's /pokemon/{name} response: only the fields we use.
 *
 * <p>Gson matches JSON keys to record component names. Keys we leave out are ignored.
 */
public record PokemonJson(int id, String name, int height, int weight, List<TypeSlot> types) {

    /** One entry of the "types" array. */
    public record TypeSlot(int slot, NamedRef type) {
    }

    /** PokeAPI's common {"name": ..., "url": ...} reference. */
    public record NamedRef(String name, String url) {
    }
}
```

The rules Gson follows:

| JSON | Java |
|---|---|
| `"name": "pikachu"` | `String name` |
| `"id": 25` | `int id` (or `long`, `double`) |
| `{ ... }` | a record |
| `[ ... ]` | `List<SomeRecord>` |
| a key you did not declare | ignored, no error |
| a key you declared that is missing | `null` for objects, `0` for numbers, `false` for booleans, **no error** |

**`snake_case` keys.** Java names are camelCase. Map them with `@SerializedName`:

```java
import com.google.gson.annotations.SerializedName;

public record PokemonJson(
        int id,
        String name,
        @SerializedName("base_experience") int baseExperience) {
}
```

See: [Gson user guide, "JSON Field Naming Support"](https://github.com/google/gson/blob/main/UserGuide.md#json-field-naming-support).

---

## Step 5. Write the domain record, and the parser that fills it

`PokemonJson` is shaped like **PokeAPI**. The rest of your app should use a type shaped like
**your app**, so a change in the API touches one class. Keep them separate.

`src/main/java/edu/csumb/cst338/model/Pokemon.java`

```java
package edu.csumb.cst338.model;

import java.util.List;

/** A Pokemon as this app uses it: flat, and with no API details. */
public record Pokemon(int id, String name, int height, int weight, List<String> types) {
}
```

`src/main/java/edu/csumb/cst338/api/PokemonParser.java`

```java
package edu.csumb.cst338.api;

import com.google.gson.Gson;
import com.google.gson.JsonParseException;
import edu.csumb.cst338.model.Pokemon;
import java.util.List;

/** Turns PokeAPI JSON text into a {@link Pokemon}. */
public class PokemonParser {

    private static final String UNREADABLE = "The service sent data this app could not read.";
    private static final int RESPONSE_WAS_OK = 200;

    private final Gson gson = new Gson();

    /**
     * Parses one /pokemon/{name} response.
     *
     * @param json the response body
     * @return the Pokemon it describes
     * @throws ApiException if the JSON is malformed or a required field is missing
     */
    public Pokemon parse(String json) {
        PokemonJson raw;
        try {
            raw = gson.fromJson(json, PokemonJson.class);
        } catch (JsonParseException e) {
            throw new ApiException(UNREADABLE, RESPONSE_WAS_OK, false, e);
        }
        if (raw == null || raw.name() == null || raw.types() == null
                || raw.types().stream().anyMatch(slot -> slot.type() == null)) {
            throw new ApiException(UNREADABLE, RESPONSE_WAS_OK, false);
        }
        List<String> typeNames = raw.types().stream()
                .map(slot -> slot.type().name())
                .toList();
        return new Pokemon(raw.id(), raw.name(), raw.height(), raw.weight(), typeNames);
    }
}
```

Three things to notice:

1. **`gson.fromJson(text, Type.class)`** is the whole Gson API you need.
2. **Gson does not reject missing fields.** The `if` checks every field you cannot do without.
   An empty body (`""`) returns `null`, which the same check catches.
3. **A bad body becomes an `ApiException`**, the same type `ApiClient` throws, so your scene
   catches one exception type for every API failure.

---

## Step 6. Test the parser with no network

Save a trimmed copy of the real response in the test. A test that needs the internet fails for
reasons unrelated to your code.

`src/test/java/edu/csumb/cst338/api/PokemonParserTest.java`

```java
package edu.csumb.cst338.api;

import static org.junit.jupiter.api.Assertions.assertEquals;
import static org.junit.jupiter.api.Assertions.assertThrows;

import edu.csumb.cst338.model.Pokemon;
import java.util.List;
import org.junit.jupiter.api.Test;

/** Parses saved JSON. No network: the JSON is a string in the test. */
class PokemonParserTest {

    // A trimmed copy of https://pokeapi.co/api/v2/pokemon/pikachu
    private static final String PIKACHU_JSON = """
            {
              "id": 25,
              "name": "pikachu",
              "height": 4,
              "weight": 60,
              "base_experience": 112,
              "types": [
                { "slot": 1, "type": { "name": "electric",
                                       "url": "https://pokeapi.co/api/v2/type/13/" } }
              ]
            }
            """;

    private final PokemonParser parser = new PokemonParser();

    @Test
    void parsesTheFieldsWeAskedFor() {
        Pokemon p = parser.parse(PIKACHU_JSON);
        assertEquals(new Pokemon(25, "pikachu", 4, 60, List.of("electric")), p);
    }

    @Test
    void malformedJsonIsAnApiException() {
        assertThrows(ApiException.class, () -> parser.parse("{ \"id\": 25, "));
    }

    @Test
    void emptyBodyIsAnApiException() {
        assertThrows(ApiException.class, () -> parser.parse(""));
    }

    @Test
    void missingNameIsAnApiException() {
        assertThrows(ApiException.class, () -> parser.parse("{ \"id\": 25, \"types\": [] }"));
    }

    @Test
    void arrayInsteadOfObjectIsAnApiException() {
        assertThrows(ApiException.class, () -> parser.parse("[1, 2, 3]"));
    }
}
```

Run:

```bash
./gradlew test
```

Expect `BUILD SUCCESSFUL`. The four negative tests count toward the required failure-path test.

> `assertEquals` on two records compares every field. Records write `equals` for you.

---

## Step 7. Call the real API

`src/main/java/edu/csumb/cst338/api/PokemonService.java`: the GET, then the parse.

```java
package edu.csumb.cst338.api;

import edu.csumb.cst338.model.Pokemon;
import java.util.function.Function;

/** Fetches a Pokemon from PokeAPI: HTTP GET, then parse. */
public class PokemonService {

    private static final String POKEMON_PATH = "/pokemon/";

    private final Function<String, String> getJson;
    private final String baseUrl;
    private final PokemonParser parser = new PokemonParser();

    /**
     * Creates the service.
     *
     * @param getJson any method that takes a URL and returns the body, such as
     *                {@code new ApiClient(agent)::getJson}
     * @param baseUrl for example {@code https://pokeapi.co/api/v2}
     */
    public PokemonService(Function<String, String> getJson, String baseUrl) {
        this.getJson = getJson;
        this.baseUrl = baseUrl;
    }

    /**
     * Fetches one Pokemon by name or id.
     *
     * @param nameOrId for example {@code pikachu} or {@code 25}
     * @return the Pokemon
     * @throws ApiException on a network failure, a non-2xx status, or unreadable JSON
     */
    public Pokemon fetch(String nameOrId) {
        String json = getJson.apply(baseUrl + POKEMON_PATH + nameOrId);
        return parser.parse(json);
    }
}
```

> The service takes a `Function<String, String>` (URL in, body out) instead of an `ApiClient`.
> Any GET method fits: `ApiClient`, the OkHttp client below, or a fake in a test.

A throwaway `main` to see it work, `src/main/java/edu/csumb/cst338/api/PokemonDemo.java`:

```java
package edu.csumb.cst338.api;

import edu.csumb.cst338.model.Pokemon;

/** Throwaway: run once to see a real Pokemon object. Delete when done. */
public class PokemonDemo {

    public static void main(String[] args) {
        ApiClient client = new ApiClient(ApiConfig.require("api.userAgent"));
        PokemonService service =
                new PokemonService(client::getJson, ApiConfig.require("api.baseUrl"));

        Pokemon pikachu = service.fetch("pikachu");
        System.out.println(pikachu);
        System.out.println(pikachu.name() + " weighs " + pikachu.weight());

        try {
            service.fetch("pikachoo");
        } catch (ApiException e) {
            System.out.println(e.getStatusCode() + " " + e.getMessage());
        }
    }
}
```

Run it with the green arrow beside `main` in IntelliJ. Expected output:

```
Pokemon[id=25, name=pikachu, height=4, weight=60, types=[electric]]
pikachu weighs 60
404 The service rejected the request (404).
```

**You now have a Java object built from a JSON API.** The remaining steps put it on screen.

---

## Step 8. Show it in a JavaFX scene

`src/main/java/edu/csumb/cst338/scenes/PokemonView.java`

```java
package edu.csumb.cst338.scenes;

import edu.csumb.cst338.api.ApiClient;
import edu.csumb.cst338.api.ApiConfig;
import edu.csumb.cst338.api.ApiException;
import edu.csumb.cst338.api.PokemonService;
import edu.csumb.cst338.model.Pokemon;
import javafx.geometry.Insets;
import javafx.scene.Parent;
import javafx.scene.control.Button;
import javafx.scene.control.Label;
import javafx.scene.control.TextField;
import javafx.scene.layout.VBox;

/** Type a name, press Fetch, see the Pokemon. */
public class PokemonView {

    private static final int SPACING = 10;
    private static final int PADDING = 20;

    private final PokemonService service = new PokemonService(
            new ApiClient(ApiConfig.require("api.userAgent"))::getJson,
            ApiConfig.require("api.baseUrl"));
    private final TextField nameField = new TextField("pikachu");
    private final Label result = new Label();

    /**
     * Builds the scene's root.
     *
     * @return the layout
     */
    public Parent getRoot() {
        Button fetch = new Button("Fetch");
        fetch.setOnAction(e -> handleFetch());
        VBox root = new VBox(SPACING, nameField, fetch, result);
        root.setPadding(new Insets(PADDING));
        return root;
    }

    private void handleFetch() {
        // NOTE: this runs on the JavaFX Application Thread, so the window freezes until
        // the network answers. Moving it into a javafx.concurrent.Task is enhancement B1.
        try {
            Pokemon p = service.fetch(nameField.getText().trim().toLowerCase());
            result.setText(p.toString());
        } catch (ApiException ex) {
            result.setText(ex.getMessage());
        }
    }
}
```

Register it inside **your slice's block** in `SceneId` and `SceneFactory.registerAll()`:

```java
// SceneId.java
POKEMON,

// SceneFactory.registerAll()
register(SceneId.POKEMON, () -> new edu.csumb.cst338.scenes.PokemonView().getRoot());
```

Run `./gradlew run -Pdev`. The main menu shows a **Pokemon** button.

> **This step is practice, not your project's design.** Here the scene calls the API every
> time. In Project 2 the API feeds your database, and your scenes read from your tables. See
> Step 10, item 7.

> **Reminder: the UI freezes during the call.** On a slow network the window stops responding
> until the API answers. That is expected at this step. Fixing it with a
> [`javafx.concurrent.Task`](https://openjfx.io/javadoc/25/javafx.graphics/javafx/concurrent/Task.html)
> is enhancement **B1**.

---

## Step 9. Lists and paged responses

Many endpoints return a list wrapped in an object:

```
https://pokeapi.co/api/v2/pokemon?limit=20&offset=0
```

```json
{ "count": 1350, "next": "https://...offset=20...", "previous": null,
  "results": [ { "name": "bulbasaur", "url": "https://pokeapi.co/api/v2/pokemon/1/" }, ... ] }
```

One record per shape, as before:

```java
public record PokemonPage(int count, String next, List<PokemonJson.NamedRef> results) {
}

PokemonPage page = gson.fromJson(json, PokemonPage.class);
```

If the **top level** of a response is an array (`[ ... ]`), ask for an array:

```java
ThingJson[] things = gson.fromJson(json, ThingJson[].class);
List<ThingJson> list = List.of(things);
```

---

## Step 10. Swap in your project's API

1. Change `api.baseUrl` in `api.properties`.
2. Open one real response in the browser (Step 2).
3. Write `XxxJson` records that mirror it (Step 4).
4. Write your domain class and an `XxxParser` that checks the required fields (Step 5).
5. Save a trimmed response in a test and write the negative cases (Step 6).
6. Write `XxxService.fetch(...)` (Step 7).
7. **Persist through your DAO.** README API rule 1: a response that is printed and thrown away
   does not count. Your scene reads from your tables, not from the API.

If your API needs a key, do not commit it. Pass it at run time and read it with `ApiConfig`:

```bash
./gradlew run -Dapi.key=YOUR_KEY
```

```java
String key = ApiConfig.require("api.key");
```

---

## Troubleshooting

| Symptom | Cause | Fix |
|---|---|---|
| Every field is `null` or `0` | Record names do not match the JSON keys | Copy the keys exactly. Use `@SerializedName` for `snake_case` |
| `Expected BEGIN_OBJECT but was BEGIN_ARRAY` | The JSON is `[...]` and you asked for a record | Ask for `Xxx[].class`, or add the wrapper record |
| `Expected BEGIN_ARRAY but was BEGIN_OBJECT` | The list is inside an object (`"results": [...]`) | Parse into a wrapper record that has a `List` field |
| `Expected an int but was ... 6.5` | A number with a decimal point | Use `double` |
| `NullPointerException` after parsing | A nested object was missing | Check it in the parser (Step 5) |
| `ApiException` 404 | Typo in the URL or the name | Open the exact URL in a browser |
| `ApiException` 429 | Too many requests | Wait, and stop calling the API on every scene load |
| `package com.google.gson does not exist` | Gradle was not reloaded | Run `./gradlew build`, then reload Gradle in IntelliJ |
| Window freezes on Fetch | Network call on the JavaFX thread | Expected until B1. See Step 8 |

---

## Optional: OkHttp instead of `java.net.http`

`ApiClient` uses `java.net.http.HttpClient`, which is built into the JDK and is enough for
this project. [OkHttp](https://github.com/square/okhttp) is the most widely used third-party
Java HTTP client. Android apps use it, and it adds interceptors (logging, an auth header on
every call), a response cache and automatic retries on connection failures. Use it if you want
those, or if you want to see how a library client compares. Gson does not care which client
fetched the text.

### O1. Add OkHttp to `build.gradle`

```groovy
def OKHTTP_VERSION = '5.5.1'
```

```groovy
implementation "com.squareup.okhttp3:okhttp:${OKHTTP_VERSION}"
```

Run `./gradlew build`. Newest version: <https://github.com/square/okhttp/blob/master/CHANGELOG.md>

### O2. Write a client with the same contract as `ApiClient`

`src/main/java/edu/csumb/cst338/api/OkHttpApiClient.java`

```java
package edu.csumb.cst338.api;

import java.io.IOException;
import java.io.InterruptedIOException;
import java.time.Duration;
import okhttp3.OkHttpClient;
import okhttp3.Request;
import okhttp3.Response;

/** The same contract as {@link ApiClient#getJson(String)}, using OkHttp. */
public class OkHttpApiClient {

    private static final Duration CONNECT_TIMEOUT = Duration.ofSeconds(10);
    private static final Duration CALL_TIMEOUT = Duration.ofSeconds(15);
    private static final String USER_AGENT_HEADER = "User-Agent";
    private static final int HTTP_TOO_MANY_REQUESTS = 429;
    private static final int HTTP_SERVER_ERROR = 500;
    private static final int NO_RESPONSE = 0;

    private final OkHttpClient http;
    private final String userAgent;

    /**
     * Creates a client. Make one and reuse it: each OkHttpClient holds a connection pool.
     *
     * @param userAgent sent with every request
     */
    public OkHttpApiClient(String userAgent) {
        this.userAgent = userAgent;
        this.http = new OkHttpClient.Builder()
                .connectTimeout(CONNECT_TIMEOUT)
                .callTimeout(CALL_TIMEOUT)
                .build();
    }

    /**
     * GETs {@code url} and returns the body.
     *
     * @param url the full URL
     * @return the response body
     * @throws ApiException on a timeout, a network failure, or any non-2xx status
     */
    public String getJson(String url) {
        Request request = new Request.Builder()
                .url(url)
                .header(USER_AGENT_HEADER, userAgent)
                .build();
        try (Response response = http.newCall(request).execute()) {
            int status = response.code();
            if (!response.isSuccessful()) {
                boolean retryable = status == HTTP_TOO_MANY_REQUESTS || status >= HTTP_SERVER_ERROR;
                throw new ApiException("The service rejected the request (" + status + ").",
                        status, retryable);
            }
            return response.body().string();
        } catch (InterruptedIOException e) {
            throw new ApiException("The service did not respond in time.", NO_RESPONSE, true, e);
        } catch (IOException e) {
            throw new ApiException("Could not reach the service.", NO_RESPONSE, true, e);
        }
    }
}
```

Compared with `ApiClient`:

| | `ApiClient` (`java.net.http`) | `OkHttpApiClient` |
|---|---|---|
| Dependency | none, in the JDK | `okhttp` |
| Build a request | `HttpRequest.newBuilder(URI.create(url))` | `new Request.Builder().url(url)` |
| Send it | `http.send(request, BodyHandlers.ofString())` | `http.newCall(request).execute()` |
| Read the body | `response.body()` | `response.body().string()`, **once** |
| Close | nothing to close | `try (Response response = ...)`, or connections leak |
| Timeout exception | `HttpTimeoutException` | `InterruptedIOException` |

### O3. Use it

Only the method reference changes. The parser and the service are untouched:

```java
PokemonService service = new PokemonService(
        new OkHttpApiClient(ApiConfig.require("api.userAgent"))::getJson,
        ApiConfig.require("api.baseUrl"));
```

Run `PokemonDemo` again. The output is the same.

---

## Links

- Gson user guide: <https://github.com/google/gson/blob/main/UserGuide.md>
- Gson README (record support, maintenance status): <https://github.com/google/gson#readme>
- PokeAPI documentation: <https://pokeapi.co/docs/v2>
- `java.net.http.HttpClient`: <https://docs.oracle.com/en/java/javase/25/docs/api/java.net.http/java/net/http/HttpClient.html>
- OkHttp recipes: <https://github.com/square/okhttp/blob/master/docs/recipes.md>
- JavaFX `Task` (enhancement B1): <https://openjfx.io/javadoc/25/javafx.graphics/javafx/concurrent/Task.html>
- In your team's repo: `README.md` "Consuming your project's API", `ARCHITECTURE.md` "The API import
  path", `ApiClientTest.java` (testing against a stub server with no network)
