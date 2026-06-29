---
name: java-playwright-e2e
description: >-
  Scaffold and build end-to-end test automation projects using Java +
  Playwright + Cucumber (BDD). Use this skill when the user wants to: bootstrap
  a new test automation framework, turn Jira/acceptance criteria into Gherkin
  feature files, write UI / API / E2E tests, discover element locators with
  Playwright, design Page Objects, apply OOP and dependency injection, or wire
  up runners, hooks, config, parallelism, CI, and database/API validation.
  Project-agnostic reference — no application-specific assumptions.
metadata:
  version: "1.1.0"
  category: Test Automation
  tags: [java, playwright, cucumber, bdd, gherkin, page-object, rest-assured, dependency-injection, e2e]
---

# Java + Playwright + Cucumber E2E Skill

A reusable blueprint for building maintainable end-to-end automation. The
stack: **Gherkin** for the spec layer, **Cucumber JVM** for binding specs to
code, **Playwright (Java)** for browser control, **REST Assured** for API calls,
and **Maven + (JUnit 5 platform suite or TestNG)** for build and execution.
Feature files are language-neutral, so the same Gherkin can describe a
TypeScript or Java application — the runtime shown here is Java.

Guiding principles, from Playwright's own best-practices guide:
- **Test user-visible behavior**, not implementation details (function names, CSS classes, DOM shape).
- **Isolate every test** — each scenario runs with its own browser context (fresh cookies/storage).
- **Let Playwright wait.** Locators auto-wait and retry; web-first assertions auto-retry. Never `Thread.sleep`.
- **Keep specs business-readable, glue thin, logic in services/pages.**

---

## WHEN TO USE

Trigger this skill when the user asks to:
- Scaffold / bootstrap a new automation project or module
- Convert acceptance criteria (Jira or otherwise) into feature files
- Write a UI, API, or end-to-end test
- Find or harden element locators with Playwright
- Design Page Objects, step layering, or dependency injection
- Add hooks, runners, tags, parallel execution, CI, config, or DB/API validation

---

## STACK & VERSIONS (verified current as of 2026)

| Component | Version | Notes |
|-----------|---------|-------|
| JDK | **17+** | Playwright Java requires 17 minimum; build fails on 11 |
| Playwright Java | **1.5x – 1.6x** | `com.microsoft.playwright:playwright`; pin and keep current |
| Cucumber JVM | **7.x** (7.2x+) | `cucumber-java` + an engine (JUnit platform or TestNG) |
| Cucumber DI | `cucumber-picocontainer` | Lightest container; Spring is the heavier alternative |
| Test engine | JUnit 5 `junit-platform-suite` **or** `cucumber-testng` | Pick one |
| API | REST Assured **5.x** | API + hybrid scenarios |
| Build | Maven 3.9+, `maven-surefire-plugin` 3.5+ | Node.js present (Playwright uses it internally) |

Keep Playwright current — new versions track the latest browser builds and catch
regressions before they ship to users.

---

## CORE WORKFLOW: Acceptance Criteria → Passing Test

Do one story/feature per iteration to keep context focused.

```
1. Read AC        → extract each acceptance criterion as a discrete, testable statement
2. Write feature  → one Scenario per AC (Gherkin); reuse existing steps first
3. Find locators  → Playwright codegen / inspector / MCP snapshot (role/label first)
4. Build pages    → Page Object with locators + web-first assertions
5. Wire steps     → thin glue → service method → page object
6. Verify         → run the scenario; assert UI + API + DB agree
7. Clean up       → remove test data created during the run
```

Rules for the loop:
- **One Scenario per AC**; cover all of that AC in one flow (setup → action → assertions). Don't split one AC across scenarios (duplicate data, noise).
- Beyond AC, add negative/edge scenarios to *find bugs*: invalid/empty/null input, boundaries, special chars, injection, unauthorized access, concurrency, error handling.
- **Validate truthfully.** Never weaken an assertion to make a test pass. A failing test is a signal — investigate the cause (or the AC understanding), not the assertion.
- **No conditional assertions.** Don't wrap `assert` in `if (data != null)`. Provision the data in setup so the assertion always runs.

---

## PROJECT SKELETON

Multi-module Maven keeps the **reusable framework** separate from **per-suite
tests**. Start with two modules; add suites as siblings.

```
automation/
├── pom.xml                          # parent (aggregator + dependency mgmt)
├── framework/                       # reusable library — no tests here
│   ├── pom.xml
│   └── src/main/java/com/example/automation/
│       ├── annotation/              # @FindBy, @VerifyAt, @VerifyBy
│       ├── driver/                  # BrowserFactory, BrowserManager, BrowserSupport
│       ├── pageobject/              # CommonPage (base class)
│       ├── context/                 # ScenarioContext (state sharing)
│       ├── config/                  # Config, Environment, secret access
│       ├── steps/                   # reusable common steps + services (ui/api/db)
│       ├── api/                     # REST Assured wrappers
│       ├── db/                      # repositories / query builders
│       └── util/                    # helpers (dates, strings, files, input)
└── ui-suite/                        # one test suite (repeat per app/area)
    ├── pom.xml
    └── src/test/
        ├── java/com/example/automation/
        │   ├── pages/               # suite Page Objects (extend CommonPage)
        │   ├── steps/               # suite-specific steps + services
        │   ├── hooks/               # @Before / @After
        │   └── runners/             # Cucumber + JUnit/TestNG runner
        └── resources/
            ├── features/            # *.feature (ui/, api/, e2e/)
            ├── junit-platform.properties   # glue/plugin config (JUnit 5)
            └── application-*.properties     # per-environment config
```

**Why split framework vs suite:** the framework (driver, annotations, common
steps, config) is built once and reused by every suite as a Maven dependency.
Suites contain only Gherkin, suite pages, and suite-specific glue.

### Dependencies (suite/framework `pom.xml`)

```xml
<dependency><groupId>com.microsoft.playwright</groupId><artifactId>playwright</artifactId><version>${playwright.version}</version></dependency>
<dependency><groupId>io.cucumber</groupId><artifactId>cucumber-java</artifactId><version>${cucumber.version}</version></dependency>
<!-- DI for step/page collaborators (constructor injection) -->
<dependency><groupId>io.cucumber</groupId><artifactId>cucumber-picocontainer</artifactId><version>${cucumber.version}</version></dependency>
<!-- Engine: choose JUnit 5 platform suite ... -->
<dependency><groupId>io.cucumber</groupId><artifactId>cucumber-junit-platform-engine</artifactId><version>${cucumber.version}</version><scope>test</scope></dependency>
<dependency><groupId>org.junit.platform</groupId><artifactId>junit-platform-suite</artifactId><scope>test</scope></dependency>
<!-- ... or TestNG: cucumber-testng (use one, not both) -->
<dependency><groupId>io.rest-assured</groupId><artifactId>rest-assured</artifactId><version>${restassured.version}</version></dependency>
<dependency><groupId>com.deque.html.axe-core</groupId><artifactId>playwright</artifactId><version>${axe.version}</version></dependency> <!-- optional ally -->
```

Install browsers once:
`mvn exec:java -e -D exec.mainClass=com.microsoft.playwright.CLI -D exec.args="install"`
(use `install --with-deps` on Linux/CI to pull system libraries).

---

## WRITING FEATURE FILES FROM ACCEPTANCE CRITERIA

Gherkin is the contract. Keep it declarative (describe *what*, not *how*).

```gherkin
@ui @regression
Feature: Internal notes on a record

  Background:
    Given User is set to "admin"
    And User loads "TaskDetails" page

  @AC1
  Scenario: AC1 — User can add an internal note
    When User enters following info on "TaskDetails" page:
      | ElementName | Value            |
      | noteTitle   | Quarterly review |
      | noteContent | Reviewed and ok  |
    And User click on "saveNote" button on "TaskDetails" page
    Then Verify "successToast" element is visible on "TaskDetails" page
    And Verify note "Quarterly review" persisted for the current record

  @AC2 @negative
  Scenario Outline: AC2 — Saving is blocked for invalid titles
    When User enters "<title>" on "noteTitle" input on "TaskDetails" page
    And User click on "saveNote" button on "TaskDetails" page
    Then Verify "<error>" element is visible on "TaskDetails" page
    Examples:
      | title | error            |
      | error | titleRequired    |
      | <xss> | titleInvalidChar |
```

Guidelines:
- **One Scenario per AC**; tag with the AC id (`@AC1`) and type (`@ui`/`@api`/`@e2e`).
- `Background` for shared setup; `Scenario Outline` + `Examples` for data-driven variants.
- Reuse existing common steps before writing new ones — search the steps package first.
- Keep steps business-level. `When User click on "saveNote" button` is good; leaking `#btn-1234` into Gherkin is bad.
- Add `@negative` scenarios for empty/null/boundary/special-char/injection/unauthorized cases.

---

## LOCATORS — discovery & strategy

Locators find elements **at the moment of interaction** and come with
auto-waiting + retry, which removes most timing-based flakiness. They are the
single largest source of flaky tests when chosen poorly, so prefer
**user-facing attributes** that survive DOM churn.

### Discover with Playwright tooling (never guess)
- **Codegen** records actions and emits resilient locators, prioritizing role/text/test-id:
  `mvn exec:java -e -D exec.mainClass=com.microsoft.playwright.CLI -D exec.args="codegen <url>"`
- **Inspector / `PWDEBUG=1`** steps through a run and highlights locators live.
- **Playwright MCP** (when available): navigate, `browser_snapshot` for the accessibility tree with refs, then `browser_close`.

### Priority order (XPath last resort)
1. `getByRole(name=...)` — buttons, links, headings (doubles as an a11y check)
2. `getByLabel(...)` — form fields
3. `getByText(...)` — visible static text
4. `getByTestId(...)` — `data-testid`
5. scoped CSS / `[aria-label="..."]`
6. XPath — only when nothing else works

### Chain and filter instead of brittle compound selectors
```java
// scope to the right row, then act within it
page.getByRole(AriaRole.LISTITEM).filter(new Locator.FilterOptions().setHasText("Product 2"))
    .getByRole(AriaRole.BUTTON, new Locator.GetByRoleOptions().setName("Add to cart")).click();
```
Strict mode: if a locator matches multiple elements, an action throws — narrow
with `filter` / `nth` / role+name rather than silently acting on the first match.

---

## PAGE OBJECTS (annotation-driven)

Every page extends a common base and declares locators as annotated fields.
Three annotations carry the metadata generic steps need.

```java
@Retention(RUNTIME) @Target(FIELD)
public @interface FindBy {        // how to locate
    String role() default "";     // preferred
    String label() default "";
    String text() default "";
    String testId() default "";
    String css() default "";
    String xpath() default "";    // discouraged
    boolean all() default false;  // true for a list of matches
}

@Retention(RUNTIME) @Target(FIELD)
public @interface VerifyAt { UrlType value(); }  // page URL (FULL / PARTIAL / CONTAINS)

@Retention(RUNTIME) @Target(FIELD)
public @interface VerifyBy { }                   // selector proving the page rendered
```

```java
public abstract class CommonPage { /* shared locators + helpers */ }

public class TaskDetailsPage extends CommonPage {

    @VerifyAt(UrlType.CONTAINS)
    public static final String URL = "/task-details";

    @VerifyBy
    public static final String ROOT = "[data-testid='task-details']";

    @FindBy(label = "Note Title")            private Locator noteTitle;
    @FindBy(label = "Note Content")          private Locator noteContent;
    @FindBy(role = "button", text = "Save")  private Locator saveNote;
    @FindBy(css = "[role='alert']")          private Locator successToast;
}
```

Rules:
- **All locators live in page objects** — never in feature files or step classes.
- Locators are `private`; no `_` prefix, no getters/setters. Generic steps resolve them by field name.
- `@VerifyAt` + `@VerifyBy` let a single `waitForPageToLoad(page)` confirm the right page rendered before acting.
- For multi-field actions, expose a method that takes a typed input object, not many primitive params.

---

## WEB-FIRST ASSERTIONS (the key reliability practice)

Use Playwright's **auto-retrying** assertions, not boolean getters. They poll
until the condition holds or the timeout expires — this is what removes flaky
"element not ready yet" failures.

```java
import static com.microsoft.playwright.assertions.PlaywrightAssertions.assertThat;

// GOOD — waits & retries until visible
assertThat(page.locator("[role='alert']")).isVisible();
assertThat(saveNote).isEnabled();
assertThat(statusLabel).hasText("Approved");

// BAD — evaluates once, no waiting, flaky
assertTrue(page.locator("[role='alert']").isVisible());   // boolean getter, returns immediately
```

- Per-assertion timeout: `assertThat(locator).isVisible(new LocatorAssertions.IsVisibleOptions().setTimeout(15000))`, or set a global default via `PlaywrightAssertions.setDefaultAssertionTimeout(...)`. Prefer per-assertion over a large global timeout.
- **Soft assertions** when you want several checks to all report before failing the scenario — collect failures, assert at the end.
- Reserve boolean getters (`isVisible()`) for control flow, never for the actual verification.

---

## TEST ISOLATION

Each scenario gets its **own browser context** (fresh cookies, storage, cache —
like a new incognito window). This prevents cascading failures and makes runs
reproducible and parallel-safe.

- Create a new `BrowserContext` + `Page` per scenario in `@Before`; close them in `@After`.
- Share one `Browser` / `Playwright` instance per thread; create a context per scenario.
- **Reuse auth instead of logging in every scenario:** log in once, save `context.storageState(...)`, and seed new contexts with `new Browser.NewContextOptions().setStorageStatePath(...)`. Far faster than UI login per test.
- **Seed data through the API**, not the UI, when a scenario only needs preconditions — quicker and less flaky.

---

## DEPENDENCY INJECTION & STATE SHARING

Two complementary mechanisms keep step classes clean and parallel-safe.

### 1. Constructor injection (PicoContainer) for collaborators

With `cucumber-picocontainer` on the classpath, Cucumber builds and shares one
instance of each collaborator **per scenario** and injects via constructors.
Steps stay thin; pages/services/hooks are injected, not `new`-ed. Because each
scenario gets fresh instances, this is inherently parallel-safe.

```java
public class TaskDetailsSteps {
    private final TaskDetailsService taskService;   // injected
    private final ScenarioContext context;          // shared per scenario

    public TaskDetailsSteps(TaskDetailsService taskService, ScenarioContext context) {
        this.taskService = taskService;
        this.context = context;
    }

    @When("User adds an internal note")
    public void userAddsNote(InternalNote note) {   // typed input from DataTable
        taskService.addNote(note);
    }
}
```

```java
public record InternalNote(String title, String content, boolean confidential) {}
```

A `BrowserContext`/`Page` holder is also a natural injectable: the hook creates
the page and stores it on a per-scenario `PageHolder`, which the services
receive by injection — so every collaborator shares the same scenario's page.

> **Spring alternative:** annotate the runner with `@CucumberContextConfiguration`
> + `@ContextConfiguration`, mark steps/services as beans. Heavier; only worth it
> if you already need the Spring context. Pick one container; don't mix.

### 2. ScenarioContext for cross-step data

A small holder carries data between steps (current user, last API response,
parsed DTOs, DB results). Prefer an **injected, scenario-scoped object** over
global `static` state. If you must support legacy static access in parallel,
back it with `InheritableThreadLocal` and clear it in `@After`.

> **Simplicity note:** the heaviest reference frameworks resolve pages/services
> by name via classpath reflection so one generic step drives every page. It is
> powerful but adds indirection. For new projects prefer **explicit DI of page
> objects**; reach for reflection-driven generic steps only when you have many
> near-identical pages and the payoff is clear.

---

## STEP LAYERING

Three layers, one responsibility each:

```
Feature (Gherkin) → Step (glue) → Service (logic + assertions) → Page / BrowserSupport / API
```

- **Step**: matches the Gherkin line, parses params, delegates. No logic.
- **Service**: orchestrates the action and owns meaningful assertions. Reusable.
- **Page / BrowserSupport / API**: lowest level — locators, Playwright/REST calls.

```java
public class TaskDetailsService {
    private final PageHolder pages;   // injected

    public void addNote(InternalNote note) {
        TaskDetailsPage page = pages.taskDetails();
        page.noteTitle().fill(note.title());
        page.noteContent().fill(note.content());
        if (note.confidential()) page.confidential().check();
        page.saveNote().click();
        assertThat(page.successToast()).isVisible();   // web-first
    }
}
```

---

## BROWSER LIFECYCLE & HOOKS

```java
public final class BrowserFactory {
    public static Browser launch(Playwright pw) {
        boolean headless = Config.HEADLESS;            // headless on CI, headed locally
        return switch (Config.BROWSER) {
            case FIREFOX -> pw.firefox().launch(opts(headless));
            case WEBKIT  -> pw.webkit().launch(opts(headless));
            case EDGE    -> pw.chromium().launch(opts(headless).setChannel("msedge"));
            default      -> pw.chromium().launch(opts(headless).setChannel("chrome"));
        };
    }
}
```

```java
public class Hooks {
    private final PageHolder pages;   // injected per scenario
    public Hooks(PageHolder pages) { this.pages = pages; }

    @Before("@ui or @e2e")
    public void startBrowser(Scenario s) {
        pages.open();                  // new context + page for THIS scenario
    }

    @After
    public void tearDown(Scenario s) {
        if (s.isFailed() && pages.hasPage())
            s.attach(pages.page().screenshot(), "image/png", "failure");
        pages.close();                 // close context/page — avoid leaks & state bleed
    }
}
```

- One context per scenario; close it in `@After`. Never share a `Page`/`Browser` across threads.
- Attach a screenshot on failure; for deeper CI debugging, capture a **trace** and open it in the Playwright **trace viewer** (timeline, DOM snapshots, network).

---

## RUNNER & EXECUTION

**JUnit 5 platform suite** (modern default):

```java
@Suite
@IncludeEngines("cucumber")
@SelectClasspathResource("features")
@ConfigurationParameter(key = GLUE_PROPERTY_NAME, value = "com.example.automation.steps,com.example.automation.hooks")
public class TestRunner { }
```

`src/test/resources/junit-platform.properties`:
```
cucumber.glue=com.example.automation.steps,com.example.automation.hooks
cucumber.plugin=pretty, html:target/cucumber.html, json:target/cucumber.json
cucumber.execution.parallel.enabled=true
cucumber.execution.parallel.config.strategy=dynamic
```

**TestNG alternative:** extend `AbstractTestNGCucumberTests` and override
`scenarios()` with `@DataProvider(parallel = true)`.

```bash
mvn test -Denv=dev                              # select environment
mvn test -Dcucumber.filter.tags="@ui and @AC1"  # filter by tags
mvn test -Dbrowser=firefox                       # cross-browser
```

**Parallelism rules:** enable parallel scenarios via the engine, but **each
thread must own its `BrowserContext`/`Page`** — never share them. PicoContainer's
per-scenario instances make this safe by default. For multiple machines, shard
the suite across CI runners.

**Tags** organize execution: `@ui`, `@api`, `@e2e`, `@regression`, `@smoke`,
`@negative`, `@security`, plus AC ids (`@AC1`).

---

## API TESTS & MOCKING

### REST Assured for API-level scenarios
```java
@When("User POST {string} with body {string}")
public void userPosts(String endpoint, String bodyKey) {
    Response r = given().header("Authorization", token())
                    .contentType(JSON).body(load(bodyKey))
                  .when().post(Config.url(endpoint));
    context.setResponse(r);
}

@Then("Verify status code is {int}")
public void verifyStatus(int expected) {
    assertEquals(context.getResponse().statusCode(), expected);
}
```

- **Cover every AC, even "UI-only" ones** — the API still returns the data the UI renders; assert the value and confirm it matches the DB.
- **Three-way validation** for create/update: input → API response → DB all agree.
  ```java
  assertEquals(api.status, input.status);          // API reflects what we sent
  assertEquals(api.status, dbRecord.getStatus());  // API reflects the DB
  assertEquals(dbRecord.getStatus(), input.status);// DB persisted what we sent
  ```
- Validate **exact** error messages from the AC. Normalize dates before comparing (timezone/format drift causes flakiness).

### Mock third-party / unstable dependencies with `page.route`
Don't test systems you don't control. Intercept and fulfill:
```java
page.route("**/api/third-party/**", route ->
    route.fulfill(new Route.FulfillOptions().setStatus(200).setBody(stubJson)));
```
Use the same `route()` API to mock, modify, abort, or capture requests for
deterministic UI scenarios.

---

## DATABASE VALIDATION

Verify persistence after any data-changing action — a UI success toast is not
proof the data was saved.

- **Avoid raw SQL.** Use an ORM/entity layer or a repository abstraction; keep reusable queries in a repository class and entities in one place.
- Provision required test data in setup using helpers so assertions never need null-guards. Use past dates for seeded data to avoid "date cannot precede last status" style validation errors.
- Control the data: test against a stable environment you own.

```java
TaskNote saved = taskRepo.findNoteByTitle(recordId, "Quarterly review");
assertNotNull(saved);
assertEquals(saved.getContent(), "Reviewed and ok");
assertEquals(saved.getCreatedBy(), context.getUser());
```

---

## CONFIG, ENVIRONMENTS & SECRETS

- Per-environment `application-<env>.properties`; select with `-Denv=<env>`.
- One `Config` class loads properties and exposes typed accessors (base URLs, browser, headless, timeouts); reference by key name, never commit values.
- **Never hardcode secrets.** Read from environment variables or a secret manager at runtime; reference by key name, never commit values.
- Keep base URLs and credentials out of feature files and page objects — inject via config.

---

## CI

- Run on every commit/PR; use **Linux** runners (cheaper, consistent).
- Install only needed browsers; `playwright install chromium --with-deps`.
- Run **headless** in CI; headed only for local development.
- Cache `~/.m2`; upload the Cucumber HTML report and traces as artifacts with `if: always()` so they exist when tests fail.
- Capture **traces on first retry** (not always-on — it is heavy) and debug failures in the trace viewer.
- Shard across runners for large suites.

---

## TEST INTEGRITY & OOP PRACTICES

**Integrity**
- No "green light" tests written only to pass — tests must genuinely validate the AC.
- No conditional assertions that skip when data is absent — provision the data.
- Don't assert data *types* ("is defined", `instanceof`); assert real values against the source of truth (API/DB).
- When a test fails, report and find the root cause. Only change a test if the AC was genuinely misunderstood.

**OOP & clean code**
- **SRP**: a page does locators; a service does logic; a step does binding. Don't blend them.
- **DRY**: reuse common steps/services; extract shared helpers. Search before adding a new step.
- **Encapsulation**: locators private in pages; shared state behind an injected `ScenarioContext`.
- **Composition**: build complex pages from smaller card/section components.
- **Typed inputs**: pass input objects/records, not long primitive parameter lists.
- **Naming**: intention-revealing; scenario/step/method names describe behavior and expected outcome.
- **No inline comments / emojis**: Javadoc public methods; keep methods small and single-purpose.

---

## QUICK REFERENCE

| Task | Action |
|------|--------|
| New project | Parent + `framework` + suite modules; add Playwright/Cucumber/engine/REST Assured; `playwright install` |
| New feature | One Scenario per AC, tag `@type @ACn`, reuse common steps, add negatives |
| Find locator | Playwright codegen / inspector / MCP; prefer role/label; chain+filter |
| New page | Extend `CommonPage`; annotate locators; add `@VerifyAt`/`@VerifyBy` |
| New step | Thin glue → service → page; inject collaborators (PicoContainer) |
| Assert UI | `assertThat(locator)` web-first (auto-retry); never boolean `isVisible()` |
| Isolate | New `BrowserContext` per scenario; reuse auth via `storageState` |
| UI test | Tag `@ui`; context in `@Before("@ui")`; assert UI + DB |
| API test | REST Assured; store response in context; three-way validation |
| Mock dep | `page.route("**/api/...", route -> route.fulfill(...))` |
| DB check | Repository/ORM (no raw SQL); verify persistence + audit fields |
| Run | `mvn test -Denv=<env> -Dcucumber.filter.tags="@..." -Dbrowser=<b>` |
| CI | Linux, headless, `install --with-deps`, traces on retry, upload report |

---

## CHECKLIST (before declaring a test done)

- [ ] Every AC has exactly one scenario, tagged with its AC id
- [ ] Negative/edge scenarios added to hunt for bugs
- [ ] Locators discovered with Playwright, kept in page objects, role/label-based
- [ ] Verifications use web-first `assertThat(locator)` — no boolean getters, no `Thread.sleep`
- [ ] Steps are thin; logic and assertions in services
- [ ] Collaborators injected (PicoContainer); state via injected `ScenarioContext`
- [ ] New browser context per scenario; auth reused via `storageState` where possible
- [ ] Third-party/unstable deps mocked via `page.route`
- [ ] UI assertions confirmed against API and/or DB (no type-only checks)
- [ ] Exact error messages and normalized dates validated
- [ ] Test data provisioned in setup; cleaned up after
- [ ] Context closed in `@After`; screenshot/trace on failure
- [ ] Parallel-safe (no shared Page/Browser); headless on CI
- [ ] No hardcoded secrets, URLs, or proprietary references
