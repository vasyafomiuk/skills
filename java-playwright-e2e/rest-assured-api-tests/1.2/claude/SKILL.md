---
name: rest-assured-api-tests
description: >-
  Writes API-level and hybrid Cucumber scenarios with REST Assured, and mocks
  third-party dependencies with Playwright page.route. Use this skill when the
  user wants to: write an API test, call a REST endpoint from a step, build a
  request with an auth header and a JSON body from a fixture, store and assert a
  Response, check a status code or response body, validate the exact error
  message from an acceptance criterion, normalize dates before comparing, do
  three-way validation (input vs API vs DB), cover a "UI-only" criterion at the
  API level, or stub/abort/modify an unstable external dependency so a UI test is
  deterministic. Project-agnostic — no application-specific assumptions.
metadata:
  version: "1.2"
  category: Test Automation
  tags: [rest-assured, api-testing, mocking, page-route, three-way-validation, contract, error-validation]
---

# REST Assured API Tests & page.route Mocking

This skill owns the API leg of the AC-to-passing-test workflow: turning an
acceptance criterion into a REST Assured request, storing the `Response` in the
shared `ScenarioContext`, and asserting status, body, and exact error messages.
It also owns mocking unstable third-party dependencies with Playwright
`page.route` so UI scenarios stay deterministic. You are invoked from
`cucumber-step-definitions`; for cross-cutting rules (locator priority,
web-first assertions, DI, isolation, naming) follow the orchestrator
`java-playwright-e2e` — this skill states only the API/mocking how-to.

## WHEN TO USE

Use this skill when the user wants to:
- Write an API or hybrid scenario, or add the API assertions to an E2E flow.
- Build a request: auth header, content type, body loaded from a JSON fixture.
- Store a `Response` in `ScenarioContext` and assert status code / body fields.
- Validate the **exact** error message an AC specifies (not a substring guess).
- Normalize timestamps before comparing so timezone/format drift can't flake.
- Do **three-way validation** — input, API response, and DB all agree.
- Cover a criterion the UI renders by asserting the API that feeds it.
- Mock, modify, abort, or capture a third-party call with `page.route`.

## PREREQUISITES (and where they come from)

This skill has no fallback chain for missing inputs — point the user at the
sibling that produces each one:
- Endpoints / request + response schema, base URL, auth scheme → ask the user, or
  `e2e-framework-setup` for config/secrets wiring.
- An existing `*.feature` with `@api` scenarios → `create-test-scenarios`.
- The DB query for the third leg of three-way validation → `database-validation`.
- Step glue, DI, `ScenarioContext`, hooks → `cucumber-step-definitions`.

## STEP 1 — Build the request (auth, content type, body from fixture)

Keep one REST Assured wrapper in the framework `api/` package. Read base URL and
token from `Config` (never hardcode; see `java-playwright-e2e`), and load bodies
from versioned fixtures, not inline strings, so Gherkin stays declarative.

```java
/** Wraps REST Assured so steps stay thin and auth/base-url live in one place. */
public class ApiClient {
    public Response post(String endpoint, String jsonBody) {
        return RestAssured.given()
                .baseUri(Config.apiBaseUrl())
                .header("Authorization", "Bearer " + Config.token())
                .contentType(ContentType.JSON)
                .body(jsonBody)
            .when().post(endpoint);
    }
}

/** Reads src/test/resources/fixtures/<name>.json from the classpath. */
public final class Fixtures {
    public static String fixture(String name) {
        try (var in = Fixtures.class.getResourceAsStream("/fixtures/" + name + ".json")) {
            return new String(Objects.requireNonNull(in, name).readAllBytes(), UTF_8);
        } catch (IOException e) { throw new UncheckedIOException(e); }
    }
}
```

## STEP 2 — Store the Response in ScenarioContext

Never assert in the same step that fires the call. Store the `Response` once so
later `Then` steps assert against it (and so the API leg of three-way validation
can reuse it). `ScenarioContext` is injected per scenario — see
`cucumber-step-definitions`.

```java
public class ApiSteps {
    private final ApiClient api;
    private final ScenarioContext context;   // injected per scenario

    public ApiSteps(ApiClient api, ScenarioContext context) {
        this.api = api;
        this.context = context;
    }

    @When("User POSTs {string} with body {string}")
    public void userPosts(String endpoint, String fixtureName) {
        context.setResponse(api.post(endpoint, Fixtures.fixture(fixtureName)));
    }
}
```

```gherkin
@api @AC1
Scenario: AC1 — Creating a note returns 201 with the saved note
  When User POSTs "/api/notes" with body "validNote"
  Then API status code is 201
  And API field "title" equals "Quarterly review"
```

## STEP 3 — Assert status, then body fields

Assert the status first (a 500 makes body assertions meaningless), then specific
fields. Extract typed values with REST Assured's `jsonPath()`; assert real values
against the source of truth, never types ("is defined", `instanceof`).

```java
@Then("API status code is {int}")
public void statusIs(int expected) {
    assertEquals(expected, context.getResponse().statusCode());
}

@Then("API field {string} equals {string}")
public void fieldEquals(String path, String expected) {
    assertEquals(expected, context.getResponse().jsonPath().getString(path));
}
```

For richer checks deserialize into a record and assert the object, not a long
list of path strings:

```java
NoteResponse note = context.getResponse().as(NoteResponse.class);
assertEquals("Quarterly review", note.title());
assertEquals(currentUser, note.createdBy());
```

## STEP 4 — Validate the EXACT error message from the AC

When an AC names an error message, assert it verbatim — exact text and exact
status. A substring or "contains" check lets a reworded or wrong-field error pass.

```gherkin
@api @negative @AC2
Scenario: AC2 — Empty title is rejected with the documented message
  When User POSTs "/api/notes" with body "emptyTitle"
  Then API status code is 422
  And API error message is "Title is required."
```

```java
@Then("API error message is {string}")
public void errorMessageIs(String expected) {
    assertEquals(expected, context.getResponse().jsonPath().getString("message"));
}
```

Drive variants with a `Scenario Outline` so each invalid input maps to its exact
documented error. Add negatives beyond the AC to hunt bugs: empty/null,
boundaries, special chars, injection strings, unauthorized (401/403), wrong
content type. Never weaken an assertion to make one pass — a real failure is a bug
signal (rule owned by `java-playwright-e2e`).

## STEP 5 — Normalize dates before comparing

Timezone and format drift between the request, the API, and the DB is a top cause
of flaky API tests. Parse both sides to a common `Instant`/`LocalDate` and compare
those — never compare raw timestamp strings.

```java
/** Compares two ISO-8601 timestamps as instants, ignoring format/zone noise. */
private static void assertSameInstant(String expected, String actual) {
    assertEquals(Instant.parse(expected), Instant.parse(actual));
}

// Date-only fields: truncate before comparing.
assertEquals(LocalDate.parse("2026-06-29"),
             OffsetDateTime.parse(note.createdAt()).toLocalDate());
```

When seeding data for a scenario, use past dates so "date cannot precede last
status" style server validation doesn't reject the setup.

## STEP 6 — Three-way validation (input → API → DB agree)

For any create/update, verify all three legs agree. You own the **input** and
**API** legs; defer the DB query to `database-validation` and assert against what
its repository returns. Three pairwise asserts catch a leg that silently disagrees
(e.g. the API echoes the input but the DB never persisted it).

```java
// input  : what the test sent
// api    : context.getResponse().as(NoteResponse.class)
// saved  : repository result from the database-validation skill
assertEquals(input.status(), api.status());          // API reflects what we sent
assertEquals(api.status(),   saved.getStatus());      // API reflects the DB
assertEquals(saved.getStatus(), input.status());      // DB persisted what we sent
```

```gherkin
@api @e2e @AC3
Scenario: AC3 — Approving a record agrees across input, API, and DB
  When User PATCHes "/api/records/42" with body "approve"
  Then API status code is 200
  And API field "status" equals "APPROVED"
  And record 42 in the database has status "APPROVED"
```

## STEP 7 — Cover every AC at the API level, even "UI-only" ones

The UI renders data the API returns, so a "the screen shows X" criterion is also
checkable at the API: assert the value in the response that the UI binds to. This
gives a fast, stable check that complements the UI assertion and pins down which
layer broke when something fails. Pair the UI assertion (owned by
`playwright-page-objects` / `cucumber-step-definitions`) with an `@api` assertion
of the same field.

## STEP 8 — Mock third-party / unstable dependencies with page.route

Don't test systems you don't control. For UI scenarios that hit a flaky or
external service, intercept the call with Playwright `page.route` so the run is
deterministic. The handler can **fulfill** (stub), **modify** (rewrite upstream),
**abort** (simulate failure), or **capture** (assert the request was made).
Register routes before the navigation that triggers them; the `page` is injected
per scenario — see `cucumber-step-definitions`.

```java
// FULFILL — return a canned body, never hitting the real service
page.route("**/api/third-party/**", route ->
    route.fulfill(new Route.FulfillOptions()
        .setStatus(200)
        .setContentType("application/json")
        .setBody(Fixtures.fixture("thirdPartyOk"))));

// ABORT — simulate an outage to test the UI's error state
page.route("**/api/third-party/**", route -> route.abort());

// MODIFY — let the call go through, then rewrite the upstream response
page.route("**/api/rates/**", route -> {
    APIResponse upstream = route.fetch();
    // setResponse seeds status + headers from upstream; setBody then replaces the body
    route.fulfill(new Route.FulfillOptions().setResponse(upstream)
        .setBody(overrideRate(upstream.text(), "1.10")));
});

// CAPTURE — assert the app actually called the dependency with the right payload
List<String> sent = new ArrayList<>();
page.route("**/api/audit/**", route -> { sent.add(route.request().postData()); route.resume(); });
```

Rules:
- Glob match the URL (`**/api/...`); narrow enough to avoid catching unrelated calls.
- Mock only what you don't own. Real, controlled endpoints should be tested for real.
- Keep stub bodies in fixtures next to request fixtures, so they version together.
- Routing is scenario-scoped; the per-scenario context teardown clears it (isolation rule owned by `java-playwright-e2e`).

## HANDOFF

- **database-validation** — owns the DB leg of three-way validation: provides the
  repository/ORM query whose result you compare against the input and API legs.
- **cucumber-step-definitions** — calls this skill; owns step glue, the injected
  `ScenarioContext`/`page`, hooks, and the service layer your assertions live in.
- **create-test-scenarios** — produces the `@api` Gherkin scenarios you implement.
- **e2e-framework-setup** — provides config/secrets (base URL, token) and the
  REST Assured dependency on the classpath.
- **java-playwright-e2e** — orchestrator and single source of truth for shared
  conventions (web-first assertions, DI, isolation, test integrity, naming).
