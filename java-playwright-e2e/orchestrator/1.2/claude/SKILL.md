---
name: java-playwright-e2e
description: >-
  Entry point and orchestrator for building end-to-end test automation with Java
  + Playwright + Cucumber (BDD), routing each request to a focused sub-skill. Use
  this skill when the user wants to: bootstrap or build an end-to-end test
  automation framework with Java + Playwright + Cucumber, stand up a new BDD
  automation project, turn acceptance criteria or Jira tickets into a passing
  automated test, or understand the overall workflow and shared conventions for
  this stack. It owns the end-to-end workflow, the cross-cutting conventions every
  sub-skill obeys, and a routing table that sends you to the right specialist skill
  for writing scenarios, page objects, step definitions, API tests, database
  validation, or framework setup. Start here when the intent is broad ("set up the
  framework", "automate this story") and let it dispatch the deep how-to.
  Project-agnostic — no application-specific assumptions.
metadata:
  version: "1.2"
  category: Test Automation
  tags: [java, playwright, cucumber, bdd, gherkin, page-object, rest-assured, dependency-injection, e2e, orchestrator]
---

# Java + Playwright + Cucumber E2E — Orchestrator

This is the entry point and single source of truth for building maintainable
end-to-end automation on the stack of **Gherkin** (spec layer), **Cucumber JVM**
(binds specs to code), **Playwright Java** (browser control), **REST Assured**
(API calls), and **Maven + JUnit 5 platform suite or TestNG** (build and run).
The guiding principle across every phase: keep specs **business-readable**, keep
glue **thin**, and put all real **logic and assertions in services and page
objects**. This orchestrator owns the workflow, the shared conventions, and the
routing — the deep, step-by-step how-to for each phase lives in the six focused
sub-skills named below.

## WHEN TO USE

Use this skill as the starting point when the user wants to:
- Bootstrap or build an end-to-end test automation framework with Java + Playwright + Cucumber.
- Stand up a new BDD automation project or add a suite to an existing one.
- Take an acceptance criterion / Jira story all the way to a passing automated test.
- Understand the overall AC -> passing-test workflow or the cross-cutting conventions.
- Decide *which* part of the stack a task belongs to (scenarios vs. pages vs. steps vs. API vs. DB vs. setup).

If the request is already narrow ("write the page object for the cart", "add the
DB check"), you may jump straight to the matching sub-skill — but the conventions
below still govern that work.

## CORE WORKFLOW: Acceptance Criteria -> Passing Test

Do one story/feature per iteration to keep context focused. Each step names the
sub-skill that owns its detailed how-to.

```
1. Read AC        extract each acceptance criterion as a discrete, testable statement   [create-test-scenarios]
2. Write feature  one Scenario per AC in Gherkin; reuse existing steps first            [create-test-scenarios]
3. Find locators  Playwright codegen / inspector / MCP snapshot (role/label first)      [playwright-page-objects]
4. Build pages    Page Object with locators + web-first assertions                      [playwright-page-objects]
5. Wire steps     thin glue -> service method -> page object; inject collaborators       [cucumber-step-definitions]
6. Verify         run the scenario; assert UI + API + DB agree                          [rest-assured-api-tests / database-validation]
7. Clean up       remove test data created during the run; confirm context closed       [cucumber-step-definitions]
```

Framework scaffolding (modules, dependencies, runner, hooks, config, CI) is a
prerequisite for step 1 on a fresh project and is owned by **e2e-framework-setup**.

Rules for the loop:
- **One Scenario per AC.** Cover that AC in one flow (setup -> action -> assertions); don't split one AC across scenarios.
- Beyond the AC, add **negative/edge** scenarios to find bugs: invalid/empty/null input, boundaries, special chars, injection, unauthorized access, concurrency.
- **Validate truthfully.** A failing test is a signal — investigate the cause, never weaken the assertion to force green.
- **Validate against a source of truth.** A UI success toast is not proof; confirm the value via API and/or DB.

## SUB-SKILLS / ROUTING

The detailed how-to lives in these six sibling skills. Match the user's intent to
a row and hand off — see **## HANDOFF** for the connections.

| User intent | Sub-skill | Purpose |
|-------------|-----------|---------|
| "Turn this AC/Jira into feature files", write or refine Gherkin, add negative scenarios | **create-test-scenarios** | Convert acceptance criteria into declarative, well-tagged Gherkin scenarios and outlines. |
| "Find the locators", "build the page object", harden flaky selectors | **playwright-page-objects** | Discover resilient locators and design annotation-driven Page Objects with web-first assertions. |
| "Wire up the steps", glue Gherkin to code, set up DI / hooks | **cucumber-step-definitions** | Author thin step glue, the service layer, hooks, and DI (picocontainer). |
| "Add API tests", validate responses, mock a third-party dependency | **rest-assured-api-tests** | Write REST Assured API and hybrid scenarios, three-way validation, and `page.route` mocking. |
| "Verify it persisted", check the database, validate audit fields | **database-validation** | Confirm persistence via repository/ORM (no raw SQL) and reconcile DB with API/UI. |
| "Bootstrap the project", set up Maven modules, dependencies, the runner, config, CI, parallelism | **e2e-framework-setup** | Scaffold the multi-module Maven framework, pin versions, and wire the runner, config, CI, and parallel execution. |

## SHARED CONVENTIONS (single source of truth)

Every sub-skill obeys these. They are stated here once; deeper detail is in the
named sub-skill.

### Stack & versions
JDK 17+, Playwright Java 1.5x–1.6x, Cucumber JVM 7.x with `cucumber-picocontainer`,
one engine (JUnit 5 `junit-platform-suite` *or* `cucumber-testng`), REST Assured 5.x,
Maven 3.9+ with `maven-surefire-plugin` 3.5+. Pin Playwright and keep it current —
new versions track the latest browser builds. The **full version matrix and
dependency declarations live in e2e-framework-setup.**

### Locator priority
Prefer user-facing attributes that survive DOM churn; XPath is the last resort.
Discover with Playwright tooling — never guess.

1. `getByRole(name=...)` — buttons, links, headings (doubles as an a11y check)
2. `getByLabel(...)` — form fields
3. `getByText(...)` — visible static text
4. `getByTestId(...)` — `data-testid`
5. scoped CSS / `[aria-label="..."]`
6. XPath — only when nothing else works

Chain and filter instead of brittle compound selectors; respect strict mode.
**Full discovery and Page Object detail in playwright-page-objects.**

```java
page.getByRole(AriaRole.LISTITEM).filter(new Locator.FilterOptions().setHasText("Product 2"))
    .getByRole(AriaRole.BUTTON, new Locator.GetByRoleOptions().setName("Add to cart")).click();
```

### Web-first assertions
Use Playwright's **auto-retrying** assertions. They poll until the condition holds
or the timeout expires, which removes "element not ready" flakiness. Never use a
boolean getter for verification, and **never `Thread.sleep`**.

```java
import static com.microsoft.playwright.assertions.PlaywrightAssertions.assertThat;

assertThat(saveNote).isEnabled();              // GOOD — waits and retries
assertThat(statusLabel).hasText("Approved");

assertTrue(page.locator("[role='alert']").isVisible());  // BAD — evaluates once, no waiting
```

Set a per-assertion timeout via the assertion's options rather than a large global
one. Reserve boolean getters (`isVisible()`) for control flow only.

### Plain (JUnit) assertions for API/DB checks
Non-UI checks (API responses, DB rows) use JUnit Jupiter assertions, and the
examples follow Jupiter's **`(expected, actual)`** argument order. If your suite
runs on TestNG instead, flip to `(actual, expected)` — equality passes or fails
the same either way; only the failure message differs.

### Dependency injection & state sharing
With `cucumber-picocontainer` on the classpath, Cucumber builds and shares one
instance of each collaborator **per scenario** and injects via constructors —
steps stay thin and runs are parallel-safe. Cross-step data (current user, last
API response, DB results) travels in an **injected, scenario-scoped
`ScenarioContext`**, never global `static` state.

```java
public class TaskDetailsSteps {
    private final TaskDetailsService taskService;   // injected collaborator
    private final ScenarioContext context;          // injected, scenario-scoped

    public TaskDetailsSteps(TaskDetailsService taskService, ScenarioContext context) {
        this.taskService = taskService;
        this.context = context;
    }
}
```

Pick one container — don't mix picocontainer and Spring. **Wiring detail in
cucumber-step-definitions.**

### Test isolation
Each scenario gets its **own `BrowserContext` + `Page`** (fresh cookies, storage,
cache — like a new incognito window), created in `@Before` and closed in `@After`.
Share one `Browser`/`Playwright` per thread; never share a `Page`/`Browser` across
threads. **Reuse auth instead of logging in every scenario:** log in once, save
`context.storageState(...)`, then seed new contexts via
`new Browser.NewContextOptions().setStorageStatePath(...)`. Seed preconditions
through the API, not the UI, when a scenario only needs setup data.

### Test integrity
- No "green light" tests written only to pass — every test must genuinely validate the AC.
- **No conditional assertions.** Don't wrap `assert` in `if (data != null)`; provision the data in setup so the assertion always runs.
- Don't assert data *types* ("is defined", `instanceof`); assert real values against the **source of truth** (API/DB).
- On failure, find the root cause. Only change a test if the AC was genuinely misunderstood.

### OOP & naming
- **SRP:** a page does locators, a service does logic and assertions, a step does binding. Don't blend them.
- **DRY:** reuse common steps/services and shared helpers; search before adding a new step.
- **Encapsulation:** locators are private in page objects; shared state sits behind the injected `ScenarioContext`.
- **Typed inputs:** pass input objects/records, not long primitive parameter lists.
- **Intention-revealing names:** scenario, step, and method names describe behavior and expected outcome.
- **No inline-comment or emoji noise:** Javadoc public methods only where useful; keep methods small and single-purpose.

## HANDOFF

This orchestrator dispatches to and connects the six sub-skills:

- **e2e-framework-setup** — go here first on a fresh project (modules, dependencies, runner, hooks, config, CI, parallelism). Returns here once the skeleton compiles, ready for step 1.
- **create-test-scenarios** — owns steps 1–2 (AC -> Gherkin). Hands off to playwright-page-objects (UI) or rest-assured-api-tests (API) once scenarios exist.
- **playwright-page-objects** — owns steps 3–4 (locators + Page Objects). Hands the page methods to cucumber-step-definitions for wiring.
- **cucumber-step-definitions** — owns step 5 and step 7 (thin glue, services, hooks, cleanup). Calls into page objects for UI and into rest-assured-api-tests / database-validation for verification.
- **rest-assured-api-tests** — owns API scenarios and the API half of step 6 (three-way validation, mocking). Pairs with database-validation.
- **database-validation** — owns the DB half of step 6 (persistence and audit-field checks). Confirms the source of truth behind every UI/API assertion.

The frozen **1.1.0** monolith remains available as the deep all-in-one reference
at `java-playwright-cucumber-e2e/1.1.0/claude/SKILL.md` when you want every phase
in a single document.
