---
name: java-playwright-e2e
description: >-
  Scaffold and build end-to-end test automation projects using Java +
  Playwright + Cucumber (BDD). Use this skill when the user wants to: bootstrap
  a new test automation framework, turn Jira/acceptance criteria into Gherkin
  feature files, write UI / API / E2E tests, discover element locators with
  Playwright, design Page Objects, apply OOP and dependency injection, or wire
  up runners, hooks, config, and database/API validation. Project-agnostic
  reference — no application-specific assumptions.
metadata:
  version: "1.0.0"
  category: Test Automation
  tags: [java, playwright, cucumber, bdd, gherkin, page-object, rest-assured, dependency-injection, testng, e2e]
---

# Java + Playwright + Cucumber E2E Skill

A reusable blueprint for building maintainable end-to-end automation. The
stack: **Gherkin** for the spec layer, **Cucumber** for binding specs to code,
**Playwright (Java)** for browser control, **REST Assured** for API calls, and
**Maven + TestNG** for build and execution. Feature files are language-neutral,
so the same Gherkin can describe a TypeScript or Java application — the runtime
shown here is Java.

The guiding principle: **specifications (Gherkin) stay business-readable, glue
code (steps) stays thin, and all logic lives in services and page objects.**

---

## WHEN TO USE

Trigger this skill when the user asks to:
- Scaffold / bootstrap a new automation project or module
- Convert acceptance criteria (Jira or otherwise) into feature files
- Write a UI, API, or end-to-end test
- Find or harden element locators with Playwright
- Design Page Objects, step layering, or dependency injection
- Add hooks, runners, tags, parallel execution, config, or DB/API validation

---

## CORE WORKFLOW: Acceptance Criteria → Passing Test

Follow this loop for every feature. Do one story/feature per iteration to keep
context focused.

```
1. Read AC        → extract each acceptance criterion as a discrete, testable statement
2. Write feature  → one Scenario per AC (Gherkin); reuse existing steps first
3. Find locators  → Playwright codegen / inspector / MCP snapshot
4. Build pages    → Page Object with annotated locators
5. Wire steps     → thin glue → service method → page object
6. Verify         → run the scenario; assert UI + API + DB agree
7. Clean up       → remove test data created during the run
```

**Critical rules for the loop:**
- One Scenario per AC. Don't split an AC across scenarios (creates duplicate data and noise). Cover all aspects of the AC in one flow: setup → action → assertions.
- Beyond AC, add negative/edge scenarios to *find bugs* — invalid input, boundary values, unauthorized access, concurrency, error handling.
- Validate truthfully. Never weaken an assertion to make a test pass. A failing test is a signal — report it, investigate, fix the cause (or the AC understanding), not the assertion.
- No conditional assertions. Don't wrap `assert` in `if (data != null)`. Provision required data in setup so the assertion always runs.

---

## PROJECT SKELETON

Multi-module Maven keeps the **reusable framework** separate from **per-suite
tests**. Start with two modules; add more suites as siblings.

```
automation/
├── pom.xml                          # parent (aggregator + dependency mgmt)
├── framework/                       # reusable library — no tests here
│   ├── pom.xml
│   └── src/main/java/com/example/automation/
│       ├── annotation/              # @FindBy, @VerifyAt, @VerifyBy
│       ├── driver/                  # BrowserFactory, BrowserManager, BrowserSupport
│       ├── pageobject/              # CommonPage (base class)
│       ├── context/                 # ScenarioContext (state sharing / DI)
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
        │   └── runners/             # Cucumber + TestNG runner
        └── resources/
            ├── features/            # *.feature (ui/, api/, e2e/)
            └── application-*.properties   # per-environment config
```

**Why split framework vs suite:** the framework (driver, annotations, common
steps, config) is built once and reused by every suite via a Maven dependency.
Suites only contain Gherkin, suite pages, and suite-specific glue.

### Minimal dependencies (suite/framework `pom.xml`)

```xml
<!-- Playwright -->
<dependency><groupId>com.microsoft.playwright</groupId><artifactId>playwright</artifactId><version>1.4x.x</version></dependency>
<!-- Cucumber + TestNG -->
<dependency><groupId>io.cucumber</groupId><artifactId>cucumber-java</artifactId><version>7.x.x</version></dependency>
<dependency><groupId>io.cucumber</groupId><artifactId>cucumber-testng</artifactId><version>7.x.x</version></dependency>
<!-- Dependency injection for step classes (pick ONE) -->
<dependency><groupId>io.cucumber</groupId><artifactId>cucumber-picocontainer</artifactId><version>7.x.x</version></dependency>
<!-- API testing -->
<dependency><groupId>io.rest-assured</groupId><artifactId>rest-assured</artifactId><version>5.x.x</version></dependency>
<!-- Accessibility (optional) -->
<dependency><groupId>com.deque.html.axe-core</groupId><artifactId>playwright</artifactId><version>4.x.x</version></dependency>
```

Install Playwright browsers once: `mvn exec:java -e -D exec.mainClass=com.microsoft.playwright.CLI -D exec.args="install"`.

---

## WRITING FEATURE FILES FROM ACCEPTANCE CRITERIA

Gherkin is the contract. Keep it declarative (describe *what*, not *how*) so it
reads like a spec, not a script.

```gherkin
@ui @regression
Feature: Internal notes on a record

  Background:
    Given User is set to "regulator_admin"
    And User loads "TaskDetails" page

  @AC1
  Scenario: AC1 — User can add an internal note
    When User enters following info on "TaskDetails" page:
      | ElementName  | Value            |
      | noteTitle    | Quarterly review |
      | noteContent  | Reviewed and ok  |
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
- **One Scenario per AC**; tag it with the AC id (`@AC1`) and the test type (`@ui`, `@api`, `@e2e`).
- Use `Background` for shared setup; `Scenario Outline` + `Examples` for data-driven variants.
- Reuse existing common steps before writing new ones — search the steps package first.
- Keep steps business-level. `When User click on "saveNote" button` is good; `When User clicks css "#btn-1234"` leaks implementation into the spec.
- Add negative scenarios (`@negative`) for empty, null, boundary, special-char, injection, and unauthorized cases.

---

## PAGE OBJECTS (annotation-driven)

Every page extends a common base and declares locators as annotated fields.
Three annotations carry all the metadata generic steps need.

```java
@Retention(RUNTIME) @Target(FIELD)
public @interface FindBy {        // how to locate an element
    String css() default "";
    String xpath() default "";    // discouraged — prefer role/label/text/css
    String text() default "";
    String role() default "";
    String label() default "";
    boolean all() default false;  // true when the field is a list of matches
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
- Locators are `private` / `readonly`; no `_` prefix, no getters/setters. Generic steps resolve them by field name.
- `@VerifyAt` + `@VerifyBy` let a single `waitForPageToLoad(page)` confirm the right page rendered before acting.
- For complex inputs, add an action method that accepts a typed input object (see DI section) instead of many primitive params.

### Locator priority (never XPath unless unavoidable)

1. `getByRole(...)` — buttons, links, headings
2. `getByLabel(...)` — form fields
3. `getByText(...)` — visible static text
4. `getByTestId(...)` — `data-testid`
5. `[aria-label="..."]` / scoped CSS
6. XPath — last resort only

---

## FINDING LOCATORS WITH PLAYWRIGHT

Discover real, stable locators before writing the page object — never guess.

- **Codegen** records actions and emits suggested locators:
  `mvn exec:java -e -D exec.mainClass=com.microsoft.playwright.CLI -D exec.args="codegen <url>"`
- **Inspector / `PWDEBUG=1`** steps through a run and highlights locators live.
- **Playwright MCP** (when available): navigate, then `browser_snapshot` returns the accessibility tree with element refs; use those to choose role/label-based locators, then `browser_close` when done.
- Prefer the accessibility tree (roles/labels) over DOM structure — it survives markup churn. Scope locators to a parent (dialog, card) to avoid cross-matches. Use exact text matching for known button labels.

---

## DEPENDENCY INJECTION & STATE SHARING

Two complementary mechanisms keep step classes clean and parallel-safe.

### 1. Constructor injection for step/page collaborators

Let the DI container (PicoContainer is the lightest; Spring works too) build and
share objects across step classes within a scenario. Steps stay thin; pages and
services are injected, not `new`-ed.

```java
public class TaskDetailsSteps {
    private final TaskDetailsService taskService;   // injected by the container
    private final ScenarioContext context;

    public TaskDetailsSteps(TaskDetailsService taskService, ScenarioContext context) {
        this.taskService = taskService;
        this.context = context;
    }

    @When("User adds an internal note")
    public void userAddsNote(InternalNote note) {   // typed input object via DataTable
        taskService.addNote(note);
    }
}
```

Use typed input objects for multi-field actions:

```java
public record InternalNote(String title, String content, boolean confidential) {}
```

> If you use **Spring** instead, annotate the runner config with
> `@CucumberContextConfiguration` + `@ContextConfiguration`, and mark steps/
> services as Spring beans. PicoContainer needs no annotations — it just
> resolves constructors. Pick one; don't mix.

### 2. ScenarioContext for cross-step shared state

A thread-local context carries data between steps (current user, session
cookies, last API response, parsed DTOs, DB results) and stays isolated per
parallel thread.

```java
public class ScenarioContext {
    private static final InheritableThreadLocal<String> USER = new InheritableThreadLocal<>();
    private static final InheritableThreadLocal<Response> RESPONSE = new InheritableThreadLocal<>();
    // ... add fields as needed: cookies, baseUrl, pojo, dbResults

    public static void setUser(String u) { USER.set(u); }
    public static String getUser()        { return USER.get(); }
    public static void setResponse(Response r) { RESPONSE.set(r); }
    public static Response getResponse()       { return RESPONSE.get(); }

    public static void clear() { USER.remove(); RESPONSE.remove(); /* ... */ }
}
```

Clear it in an `@After` hook so state never leaks between scenarios.

> **Simplicity note:** the reference framework resolves pages/services by name
> via classpath reflection so one generic step works across all pages. That is
> powerful but heavy. Prefer **explicit DI of page objects** for new projects;
> reach for reflection-driven generic steps only when you have many near-
> identical pages and the indirection clearly pays off.

---

## STEP LAYERING

Three layers, each with one responsibility:

```
Feature (Gherkin) → Step (glue) → Service (logic) → Page / BrowserSupport / API
```

- **Step**: matches the Gherkin line, parses params, delegates. No logic, no assertions beyond trivial ones.
- **Service**: orchestrates the action and owns the meaningful assertions. Reusable across steps.
- **Page / BrowserSupport / API**: the lowest level — locators, Playwright/REST calls.

```java
// Service — logic + assertions live here
public class TaskDetailsService {
    public void addNote(InternalNote note) {
        TaskDetailsPage page = pages.get(TaskDetailsPage.class);
        browser.fill(page, "noteTitle", note.title());
        browser.fill(page, "noteContent", note.content());
        if (note.confidential()) browser.check(page, "confidential");
        browser.click(page, "saveNote");
        browser.assertVisible(page, "successToast");
    }
}
```

---

## BROWSER LIFECYCLE

A factory creates the browser per the active config; a manager holds the
thread-local `Page`; hooks start/stop it.

```java
public final class BrowserFactory {
    public static void launch() {
        Playwright pw = Playwright.create();
        Browser browser = switch (Config.BROWSER) {
            case CHROME   -> pw.chromium().launch(opts("chrome"));
            case EDGE     -> pw.chromium().launch(opts("msedge"));
            case FIREFOX  -> pw.firefox().launch(headless());
            case WEBKIT   -> pw.webkit().launch(headless());
            default       -> pw.chromium().launch(headless());
        };
        BrowserManager.set(pw, browser, browser.newContext().newPage());
    }
}

public class Hooks {
    @Before("@ui or @e2e")
    public void startBrowser(Scenario s) { ScenarioContext.setScenario(s); BrowserFactory.launch(); }

    @After
    public void tearDown(Scenario s) {
        if (s.isFailed() && BrowserManager.hasPage())
            s.attach(BrowserManager.page().screenshot(), "image/png", "failure");
        BrowserManager.close();      // close browser/context to avoid leaks
        ScenarioContext.clear();     // reset shared state
    }
}
```

Always close the browser in `@After` — leaked contexts exhaust resources and
contaminate later scenarios.

---

## RUNNER & EXECUTION

```java
@CucumberOptions(
    features = "classpath:features",
    glue = {"com.example.automation.steps", "com.example.automation.hooks"},
    plugin = {"pretty", "html:target/cucumber.html", "json:target/cucumber.json"})
public class TestRunner extends AbstractTestNGCucumberTests {
    @Override @DataProvider(parallel = true)   // parallel scenarios
    public Object[][] scenarios() { return super.scenarios(); }
}
```

```bash
# Run a suite, selecting environment via system property
mvn test -Denv=dev
# Filter by tag
mvn test -Dcucumber.filter.tags="@ui and @AC1"
```

**Tags** organize execution: `@ui`, `@api`, `@e2e`, `@regression`, `@smoke`,
`@negative`, `@security`, plus AC ids (`@AC1`). Tag every scenario with its type
and the AC it verifies.

---

## API TESTS (REST Assured)

```java
@When("User POST {string} with body {string}")
public void userPosts(String endpoint, String bodyKey) {
    Response r = given().header("Authorization", token())
                    .contentType(JSON).body(load(bodyKey))
                  .when().post(Config.url(endpoint));
    ScenarioContext.setResponse(r);
}

@Then("Verify status code is {int}")
public void verifyStatus(int expected) {
    assertEquals(ScenarioContext.getResponse().statusCode(), expected);
}
```

**Cover every AC, even "UI-only" ones** — the API still returns the data the UI
renders, so assert the response value and confirm it matches the database.

**Three-way validation** for any create/update: input → API response → DB all
agree.

```java
assertEquals(api.status, input.status);          // API reflects what we sent
assertEquals(api.status, dbRecord.getStatus());  // API reflects the DB
assertEquals(dbRecord.getStatus(), input.status);// DB persisted what we sent
```

Validate **exact** error messages from the AC, not just that an error occurred.
Normalize dates before comparing (timezone/format differences cause flakiness).

---

## DATABASE VALIDATION

Verify persistence after any data-changing action — UI success messages are not
proof the data was saved.

- **Avoid raw SQL.** Use an ORM/entity layer or a repository abstraction.
- Keep reusable queries in a repository class; keep entity definitions in one place.
- Provision required test data in `@Before`/setup using helpers so assertions never need null-guards.
- Use past dates for seeded data to avoid "date cannot precede last status" style validation errors.

```java
// repository pattern — no inline SQL in tests
TaskNote saved = taskRepo.findNoteByTitle(recordId, "Quarterly review");
assertNotNull(saved);
assertEquals(saved.getContent(), "Reviewed and ok");
assertEquals(saved.getCreatedBy(), ScenarioContext.getUser());
```

---

## CONFIG, ENVIRONMENTS & SECRETS

- Per-environment `application-<env>.properties`; select with `-Denv=<env>`.
- A single `Config` class loads properties and exposes typed accessors (base URLs, browser, headless, timeouts).
- **Never hardcode secrets.** Read them from environment variables or a secret manager at runtime; reference by key name, never commit values.
- Keep base URLs and credentials out of feature files and page objects — inject via config.

---

## TEST INTEGRITY & OOP PRACTICES

**Test integrity**
- No "green light" tests written only to pass. Tests must genuinely validate the AC.
- No conditional assertions that skip when data is absent — provision the data.
- Don't assert data *types* (`instanceof`, "is defined"); assert real values against the source of truth (API/DB).
- When a test fails, report it and find the root cause. Only change a test if the AC was genuinely misunderstood.

**OOP & clean code**
- **SRP**: a page does locators; a service does logic; a step does binding. Don't blend them.
- **DRY**: reuse common steps/services; extract shared helpers. Search before adding a new step.
- **Encapsulation**: locators private in pages; shared state behind `ScenarioContext`.
- **Composition**: build complex pages from smaller card/section components.
- **Typed inputs**: pass input objects/records, not long primitive parameter lists.
- **Naming**: intention-revealing. Scenario/step/method names describe behavior and expected outcome.
- **No inline comments / emojis**: document public methods with Javadoc. Keep methods small and single-purpose.

---

## QUICK REFERENCE

| Task | Action |
|------|--------|
| New project | Create parent + `framework` + suite modules; add Playwright/Cucumber/TestNG/REST Assured deps; `playwright install` |
| New feature | One Scenario per AC, reuse common steps, add negatives |
| Find locator | Playwright codegen / inspector / MCP snapshot; prefer role/label |
| New page | Extend `CommonPage`; annotate locators (`@FindBy`); add `@VerifyAt`/`@VerifyBy` |
| New step | Thin glue → service → page; inject collaborators via DI |
| UI test | Tag `@ui`; browser via `@Before("@ui")`; assert UI + DB |
| API test | REST Assured; store response in context; three-way validation |
| E2E test | Tag `@e2e`; chain UI + API + DB across the flow |
| DB check | Repository/ORM (no raw SQL); verify persistence + audit fields |
| Run | `mvn test -Denv=<env> -Dcucumber.filter.tags="@..."` |

---

## CHECKLIST (before declaring a test done)

- [ ] Every AC has exactly one scenario, tagged with its AC id
- [ ] Negative/edge scenarios added to hunt for bugs
- [ ] Locators discovered with Playwright, kept in page objects, role/label-based
- [ ] Steps are thin; logic and assertions in services
- [ ] Collaborators injected via DI; shared state via `ScenarioContext`
- [ ] UI assertions confirmed against API and/or DB (no type-only checks)
- [ ] Exact error messages and normalized dates validated
- [ ] Test data provisioned in setup; cleaned up after
- [ ] Browser closed and context cleared in `@After`
- [ ] No hardcoded secrets, URLs, or proprietary references
