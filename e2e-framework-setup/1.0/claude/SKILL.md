---
name: e2e-framework-setup
description: >-
  Scaffolds a new Java + Playwright + Cucumber end-to-end automation project and
  wires up its execution machinery from scratch. Use this skill when the user
  wants to: bootstrap or initialize a new test automation framework, create the
  Maven project structure or pom.xml, set up a multi-module skeleton (parent +
  framework + suites), add and pin Playwright/Cucumber/REST Assured dependencies,
  install Playwright browsers, configure a Cucumber runner (JUnit 5 platform
  suite or TestNG), enable parallel execution, manage per-environment config and
  secrets, or stand up a CI pipeline (GitHub Actions/GitLab) that runs the suite
  headless on Linux with traces and reports. This is the prerequisite scaffold
  every other test-authoring skill assumes already exists; hook bodies and the
  per-scenario context lifecycle belong to cucumber-step-definitions.
  Project-agnostic — no application-specific assumptions.
metadata:
  version: "1.0.0"
  category: Test Automation
  tags: [maven, scaffolding, junit5, testng, ci, parallel-execution, config, project-setup, playwright-install]
---

# E2E Framework Setup

This skill stands up the empty, runnable skeleton that the rest of the
acceptance-criteria → passing-test workflow builds on: a multi-module Maven
project, pinned dependencies, a working Cucumber runner, parallel execution,
config/secrets, and CI. When you finish, `mvn test` runs zero scenarios green —
the framework module compiles, a suite module exists, and Playwright browsers are
installed. Cross-cutting conventions (locator priority, web-first assertions, DI,
isolation, naming/OOP) live in the orchestrator **java-playwright-e2e** — this
skill owns only the scaffold and the machinery that executes it.

## WHEN TO USE

Use this skill when the user asks to:
- "Bootstrap / set up / initialize a new automation project (or module)."
- "Create the Maven structure / parent pom / multi-module skeleton for E2E tests."
- "Add Playwright + Cucumber + REST Assured dependencies" / "what versions do I pin?"
- "Install Playwright browsers."
- "Set up the Cucumber runner" / "JUnit 5 vs TestNG runner" / "make tests run in parallel."
- "Add per-environment config / `-Denv` selection / keep secrets out of the repo."
- "Wire up CI so the suite runs headless on Linux with traces and an HTML report."

Do NOT use this for authoring tests — once the scaffold runs, hand off (see HANDOFF).

## STACK & VERSIONS (single source surfaced here; the orchestrator references this matrix)

| Component | Version | Notes |
|-----------|---------|-------|
| JDK | **17+** | Playwright Java requires 17 minimum; build fails on 11. |
| Playwright Java | **1.5x – 1.6x** | `com.microsoft.playwright:playwright`; pin and keep current. |
| Cucumber JVM | **7.x** (7.2x+) | `cucumber-java` + one engine (JUnit platform **or** TestNG). |
| Cucumber DI | `cucumber-picocontainer` (7.x) | Lightest container; per-scenario instances make parallel safe. Spring is the heavier alternative. |
| Test engine | `cucumber-junit-platform-engine` + `junit-platform-suite` **or** `cucumber-testng` | Pick exactly one. |
| API | REST Assured **5.x** | `io.rest-assured:rest-assured`; API + hybrid scenarios. |
| Accessibility (optional) | axe-core Playwright (`com.deque.html.axe-core:playwright`) | Drop in only if a11y checks are wanted. |
| Build | Maven 3.9+, `maven-surefire-plugin` 3.5+ | Node.js is pulled in transitively (Playwright drives it internally). |

Keep Playwright current — new versions track the latest browser builds and catch
regressions early. Pin every version in `<properties>` so all modules agree.

## STEP 1 — Multi-module skeleton (framework vs suite)

A multi-module Maven layout keeps the **reusable framework** (driver,
annotations, config, common steps — *no tests*) separate from **per-suite
modules** (Gherkin + suite pages/glue). Build the framework once; every suite
consumes it as a Maven dependency. Add suites as siblings, one per app/area.

```
automation/
├── pom.xml                          # parent: aggregator + <dependencyManagement>
├── framework/                       # reusable library — NO tests here
│   ├── pom.xml
│   └── src/main/java/com/example/automation/
│       ├── annotation/              # @FindBy, @VerifyAt, @VerifyBy → playwright-page-objects
│       ├── driver/                  # BrowserFactory, BrowserManager, PageHolder
│       ├── pageobject/  context/    # CommonPage base; ScenarioContext (state sharing)
│       ├── config/                  # Config, Environment, secret access
│       ├── steps/  api/  db/        # common steps+services; REST Assured wrappers; repositories
│       └── util/                    # helpers (dates, strings, files)
└── ui-suite/                        # one test suite (repeat per app/area; depends on framework)
    └── src/test/
        ├── java/com/example/automation/
        │   ├── pages/  steps/  hooks/  runners/   # suite pages, glue, @Before/@After, runner
        └── resources/
            ├── features/                   # *.feature (ui/, api/, e2e/)
            ├── junit-platform.properties   # glue/plugin/parallel config (JUnit 5)
            └── application-*.properties     # per-environment config
```

**Rule:** never put feature files or page objects in the `framework` module, and
never put driver/config/annotation infrastructure in a suite. If a second suite
would copy a class, it belongs in `framework`.

## STEP 2 — Parent POM and pinned dependencies

Parent declares modules, the Java version, and pins all versions in
`<properties>`; `<dependencyManagement>` lets child modules omit versions.

```xml
<project>
  <packaging>pom</packaging>
  <modules>
    <module>framework</module>
    <module>ui-suite</module>
  </modules>
  <properties>
    <maven.compiler.release>17</maven.compiler.release>
    <playwright.version>1.6x.0</playwright.version>
    <cucumber.version>7.2x.0</cucumber.version>
    <junit.platform.version>1.11.x</junit.platform.version>
    <restassured.version>5.x.0</restassured.version>
    <surefire.version>3.5.x</surefire.version>
  </properties>
</project>
```

Dependencies (`framework` for shared libs; the suite adds the engine/runner deps):

```xml
<dependency><groupId>com.microsoft.playwright</groupId><artifactId>playwright</artifactId><version>${playwright.version}</version></dependency>
<dependency><groupId>io.cucumber</groupId><artifactId>cucumber-java</artifactId><version>${cucumber.version}</version></dependency>
<!-- DI for step/page collaborators (per-scenario instances → parallel-safe) -->
<dependency><groupId>io.cucumber</groupId><artifactId>cucumber-picocontainer</artifactId><version>${cucumber.version}</version></dependency>
<dependency><groupId>io.rest-assured</groupId><artifactId>rest-assured</artifactId><version>${restassured.version}</version></dependency>
<!-- Engine: choose JUnit 5 platform suite ... -->
<dependency><groupId>io.cucumber</groupId><artifactId>cucumber-junit-platform-engine</artifactId><version>${cucumber.version}</version><scope>test</scope></dependency>
<dependency><groupId>org.junit.platform</groupId><artifactId>junit-platform-suite</artifactId><version>${junit.platform.version}</version><scope>test</scope></dependency>
<!-- ... OR TestNG: io.cucumber:cucumber-testng (use one engine, never both) -->
```

`maven-surefire-plugin` runs JUnit-platform / TestNG tests on `mvn test`; pin it
to 3.5+ so it discovers the JUnit Platform suite without extra wiring.

## STEP 3 — Install Playwright browsers

Run once after the framework module compiles (it provides the Playwright CLI):

```bash
mvn -q -pl framework exec:java -e \
  -D exec.mainClass=com.microsoft.playwright.CLI \
  -D exec.args="install"
```

On Linux / CI add `--with-deps` to pull system libraries:
`-D exec.args="install --with-deps chromium"`. Install only the browsers you run
(usually `chromium`) to keep CI fast.

## STEP 4 — Runner & parallel execution

**JUnit 5 platform suite (modern default).** The `@Suite` class is just an entry
point; configuration lives in `junit-platform.properties`.

```java
@Suite
@IncludeEngines("cucumber")
@SelectClasspathResource("features")
@ConfigurationParameter(
    key = GLUE_PROPERTY_NAME,
    value = "com.example.automation.steps,com.example.automation.hooks")
public class TestRunner { }
```

`src/test/resources/junit-platform.properties`:
```
cucumber.glue=com.example.automation.steps,com.example.automation.hooks
cucumber.plugin=pretty, html:target/cucumber.html, json:target/cucumber.json
cucumber.execution.parallel.enabled=true
cucumber.execution.parallel.config.strategy=dynamic
```

**TestNG alternative** — extend `AbstractTestNGCucumberTests` and override
`scenarios()` with `@DataProvider(parallel = true)`; configure glue/plugin via
`@CucumberOptions`. Use one engine, never both.

```java
@CucumberOptions(features = "src/test/resources/features",
    glue = {"com.example.automation.steps", "com.example.automation.hooks"},
    plugin = {"pretty", "html:target/cucumber.html"})
public class TestRunner extends AbstractTestNGCucumberTests {
    @Override @DataProvider(parallel = true)
    public Object[][] scenarios() { return super.scenarios(); }
}
```

**Parallel rule (the one that prevents 90% of flakiness):** parallel runs **one
scenario per thread**, and **each thread must own its own `BrowserContext` +
`Page`** — never share them across threads. PicoContainer hands each scenario
fresh collaborator instances, so a per-scenario `PageHolder` created in `@Before`
and closed in `@After` is automatically thread-safe. The framework only has to
guarantee instances are per-scenario; the hook/lifecycle details belong to
**cucumber-step-definitions**. For more capacity, shard the suite across CI
runners rather than over-threading one machine.

Common invocations:
```bash
mvn test -Denv=dev                                # select environment
mvn test -Dcucumber.filter.tags="@ui and @AC1"    # filter by tags
mvn test -Dbrowser=firefox                         # cross-browser
```

## STEP 5 — Config, environments & secrets

- One `application-<env>.properties` per environment (`application-dev.properties`,
  `application-staging.properties`); select with `-Denv=<env>`.
- A single typed `Config` class loads the active file and exposes typed accessors
  (base URLs, browser, headless, timeouts) — callers reference keys, never inline
  literals.
- **Never commit secret values.** Read credentials/tokens from environment
  variables or a secret manager at runtime; the properties file holds only the
  *key name* or a `${ENV_VAR}` reference.

```java
public final class Config {
    private static final Properties P = load(System.getProperty("env", "dev"));
    public static String baseUrl()  { return P.getProperty("base.url"); }
    public static boolean headless() { return Boolean.parseBoolean(P.getProperty("headless", "true")); }
    public static String secret(String key) { return System.getenv(key); } // never from the file
}
```

Keep base URLs and credentials out of feature files and page objects entirely —
they flow in through `Config`.

## STEP 6 — CI pipeline

Run the suite on every commit/PR. Linux runners are cheaper and consistent;
headless is the CI default (headed only locally).

```yaml
jobs:
  e2e:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-java@v4
        with: { distribution: temurin, java-version: '17', cache: maven }
      - name: Install browsers + system deps
        run: mvn -q -pl framework exec:java
             -D exec.mainClass=com.microsoft.playwright.CLI
             -D exec.args="install --with-deps chromium"
      - name: Run E2E suite
        run: mvn test -Denv=ci -Dheadless=true
      - name: Upload report + traces
        if: always()                         # so artifacts exist on failure
        uses: actions/upload-artifact@v4
        with:
          name: e2e-report
          path: |
            **/target/cucumber.html
            **/target/**/trace.zip
```

CI essentials:
- `cache: maven` (or cache `~/.m2`) to skip re-downloading dependencies.
- `install --with-deps` — the system libraries are required on bare Linux runners.
- Capture **traces on first retry**, not always-on (traces are heavy); open them
  in the Playwright trace viewer to debug failures.
- Upload the Cucumber HTML report and traces with `if: always()`.
- **Shard** large suites across runners (matrix on a tag/feature subset) instead
  of pushing thread count on one box.

## INPUT-ACQUISITION FALLBACK

This skill has no external-input chain — it produces inputs for others. If a
prerequisite for *running* something is missing (an endpoint base URL, a DB
connection string, an existing feature or page), it is not this skill's job to
invent it: ask the user, or point them to the sibling that produces it
(endpoints → **rest-assured-api-tests**, schema/connection → **database-validation**,
features → **create-test-scenarios**, pages → **playwright-page-objects**).

## HANDOFF

Once `mvn test` runs green with zero scenarios and browsers are installed, the
scaffold is done. Hand off:
- **java-playwright-e2e** — the orchestrator; single source of truth for all
  cross-cutting conventions (locator priority, web-first assertions, DI,
  isolation, test-integrity, naming/OOP). It references the STACK & VERSIONS
  matrix above. Return here when a new suite module is needed.
- **create-test-scenarios** — turn Jira/AC into Gherkin feature files that drop
  into a suite's `resources/features/`.
- **playwright-page-objects** — build the `@FindBy`/`@VerifyAt`/`@VerifyBy`
  annotations and Page Objects this skeleton reserves packages for.
- **cucumber-step-definitions** — the step glue → service layer, plus the
  per-scenario `@Before`/`@After` hooks that create and close the `BrowserContext`
  this skill made parallel-safe.
- **rest-assured-api-tests** — REST Assured scenarios and `page.route` mocking,
  using the pinned REST Assured dependency added here.
- **database-validation** — persistence verification via the `db/` repository
  packages reserved in the framework module.
