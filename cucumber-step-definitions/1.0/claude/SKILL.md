---
name: cucumber-step-definitions
description: >-
  Implements Cucumber-JVM step definitions and the service layer behind them so
  Gherkin lines bind to real, parallel-safe Java automation code. Use this skill
  when the user wants to: write or fix step definition glue, map Gherkin steps to
  Java methods, wire up cucumber-picocontainer dependency injection, share state
  between steps with a ScenarioContext, build typed input objects or records from
  Cucumber DataTables, add @Before/@After hooks, manage the per-scenario browser
  context/page lifecycle, attach screenshots or traces on failure, or refactor fat
  steps into a thin-glue + service-layer design. Also use it when steps are flaky
  under parallel execution, throw PicoContainer "cannot instantiate" errors, or
  leak state across scenarios. The runner and parallel-execution config itself
  live in e2e-framework-setup; this skill owns the hook bodies and per-scenario
  lifecycle. Project-agnostic — no application-specific assumptions.
metadata:
  version: "1.0.0"
  category: Test Automation
  tags: [cucumber, step-definitions, dependency-injection, picocontainer, hooks, scenario-context, service-layer, parallel]
---

# Cucumber Step Definitions & Service Layer

This skill owns the glue that turns a Gherkin line into executing Java. It sits in
the middle of the acceptance-criteria -> passing-test workflow: a feature file
already exists (from `create-test-scenarios`) and page objects exist (from
`playwright-page-objects`); your job is to bind each step to a method, inject the
collaborators it needs, push real logic and assertions down into a service layer,
and manage the per-scenario lifecycle so the whole thing runs parallel-safe. For
cross-cutting rules (stack/versions, locator priority, web-first assertions,
isolation, naming/OOP, test-integrity) defer to the orchestrator
`java-playwright-e2e` — this skill states each in one line and spends its words on
the step/DI/hook how-to.

## WHEN TO USE

Use this skill when the user asks to:
- Write or fix step definitions / glue mapping Gherkin to Java methods.
- Convert a feature's steps into a thin-glue + service-layer design.
- Set up or debug cucumber-picocontainer constructor injection.
- Share data across steps (current user, last response, parsed DTO) via a ScenarioContext.
- Build typed input objects/records from a Cucumber DataTable.
- Add @Before/@After hooks, browser context/page lifecycle, screenshot/trace-on-failure.
- Fix flaky-under-parallel steps, PicoContainer "cannot instantiate" errors, or state bleed between scenarios.

If a prerequisite is missing, hand off rather than invent it: no feature file ->
`create-test-scenarios`; no page object / locators -> `playwright-page-objects`;
no project scaffold, runner, or parallel config -> `e2e-framework-setup`.

## THE THREE-LAYER RULE

Every step obeys exactly this chain. Each layer has one responsibility:

```
Feature (Gherkin) -> Step (glue) -> Service (logic + assertions) -> Page / BrowserSupport / API
```

- **Step**: matches the Gherkin line, parses/coerces params, delegates to a service. No branching, no assertions, no Playwright calls.
- **Service**: orchestrates the action and owns the meaningful assertions. Reusable across steps. This is where logic lives.
- **Page / BrowserSupport / API**: lowest level — locators and Playwright/REST calls. Owned by sibling skills, not here.

The test for a correct step: if you deleted the Gherkin, the step body should read
like a single sentence — "parse this, hand it to that service." If a step has an
`if`, a loop, or an `assertThat`, the logic belongs in a service.

```java
@When("User adds an internal note")
public void userAddsNote(InternalNote note) {   // typed param, see DataTables below
    taskService.addNote(note);                  // delegate; no logic here
}
```

## DEPENDENCY INJECTION (cucumber-picocontainer)

With `cucumber-picocontainer` on the classpath, Cucumber instantiates one instance
of each collaborator **per scenario** and injects them through constructors. Steps,
services, hooks, the page holder, and the ScenarioContext are all plain classes
with constructor parameters — never `new`-ed by you. Because instances are fresh
per scenario, this is inherently parallel-safe (the isolation rule lives in
`java-playwright-e2e`).

```java
public class TaskDetailsSteps {
    private final TaskDetailsService taskService;
    private final ScenarioContext context;

    public TaskDetailsSteps(TaskDetailsService taskService, ScenarioContext context) {
        this.taskService = taskService;
        this.context = context;
    }
    // @When / @Then methods delegate to taskService ...
}
```

PicoContainer wiring rules — these are the ones that actually bite:
- Declare every collaborator as a **constructor parameter**. No field injection, no setters, no static lookups.
- A class may appear in many constructors; PicoContainer hands out the **same per-scenario instance** to each, so steps and services share one `PageHolder`/`ScenarioContext`.
- **One public constructor per injectable class.** Two constructors -> PicoContainer can't choose -> "cannot instantiate" at runtime.
- **No cyclic dependencies** (A needs B, B needs A) -> PicoContainer throws. Break the cycle by moving shared state into a third holder both depend on.
- Keep injectables in the **glue path** (the packages named in `cucumber.glue`) so Cucumber discovers them.
- Don't put scenario state in `static` fields; let the per-scenario instance carry it (see ScenarioContext).

> **Spring alternative:** annotate the runner with `@CucumberContextConfiguration` +
> `@ContextConfiguration` and mark steps/services as beans. Heavier — only worth it
> if you already run a Spring context. Pick one container; never mix Pico and Spring.

## TYPED INPUTS FROM DATATABLES

Pass an input object or record, never a long primitive parameter list. Register a
`@DataTableType` once so any step receives the typed object directly. Prefer a
`record` for immutable input.

```java
public record InternalNote(String title, String content, boolean confidential) {}
```

```java
public class DataTableTransformers {
    @DataTableType
    public InternalNote internalNote(Map<String, String> row) {
        return new InternalNote(
            row.get("title"),
            row.get("content"),
            Boolean.parseBoolean(row.getOrDefault("confidential", "false")));
    }
}
```

```gherkin
When User adds an internal note
  | title            | content         | confidential |
  | Quarterly review | Reviewed and ok | true         |
```

Notes:
- A **single-row** table maps to one object; a **multi-row** table maps to `List<InternalNote>` automatically once the type is registered.
- Register transformers in a glue-path class so Cucumber finds them; they participate in DI like any other glue.
- For scalar coercions (enums, money, dates) use `@ParameterType` to keep `{string}` out of step signatures and validation in one place.
- Keep parsing in the transformer/step, not the service — the service receives clean typed objects.

## SCENARIOCONTEXT FOR CROSS-STEP STATE

When one step produces data a later step consumes (the logged-in user, the last API
response, a parsed DTO, a DB row), carry it in an **injected, scenario-scoped**
holder — not global statics.

```java
public class ScenarioContext {                  // injected per scenario
    private final Map<String, Object> bag = new HashMap<>();
    public void put(String key, Object value) { bag.put(key, value); }
    @SuppressWarnings("unchecked")
    public <T> T get(String key) { return (T) bag.get(key); }
    public boolean has(String key) { return bag.containsKey(key); }
}
```

- A fresh instance per scenario means **no `@After` cleanup needed** and no bleed between scenarios — that is the whole point of preferring it over statics.
- Prefer **typed accessors** (`setResponse`/`getResponse`) over a raw string-keyed map when the set of shared keys is known; the map form above is the escape hatch.
- If you must keep a legacy `static` API working under parallel execution, back it with `InheritableThreadLocal` and clear it in `@After`. New code: just inject `ScenarioContext`.

```java
// producer step
context.setResponse(response);
// consumer step in a later line
assertThat(context.getResponse().statusCode()).isEqualTo(201);
```

## HOOKS & PER-SCENARIO LIFECYCLE

Hooks are glue too — they get DI. Inject a `PageHolder` (a scenario-scoped object
that owns the `BrowserContext`/`Page`); the hook opens it in `@Before` and closes it
in `@After`, so every injected service sees the same scenario's page.

```java
public class Hooks {
    private final PageHolder pages;                 // injected per scenario
    public Hooks(PageHolder pages) { this.pages = pages; }

    @Before("@ui or @e2e")
    public void startBrowser(Scenario s) {
        pages.open();                               // new context + page for THIS scenario
    }

    @After("@ui or @e2e")
    public void tearDown(Scenario s) {
        if (s.isFailed() && pages.hasPage()) {
            s.attach(pages.page().screenshot(), "image/png", "failure");
        }
        pages.close();                              // close context/page; prevent leaks + state bleed
    }
}
```

Rules:
- **Tag hooks** (`@Before("@ui or @e2e")`) so API-only scenarios don't pay to launch a browser; back them with a separate API-setup hook if needed.
- **Order with `order`** when sequencing matters: `@Before(order = 0)` runs before `@Before(order = 10)`; `@After` runs in reverse. Use it to seed data after the context exists.
- Use a **`@BeforeAll`/`@AfterAll`** static hook (or the runner) for once-per-JVM cost like launching `Playwright`/`Browser`; share one `Browser` per thread, one **context per scenario**.
- **Always `pages.close()` in `@After`** even on failure — a leaked context is the most common cause of slow, flaky parallel runs.
- **Attach a screenshot on failure**; for deeper CI debugging, start a `Tracing` recording in `@Before` and `tracing.stop(... setPath(...))` + `s.attach(...)` in `@After` for failed scenarios, then open it in the Playwright trace viewer. Full trace/CI guidance lives in `e2e-framework-setup`.

```java
// PageHolder sketch — owns lifecycle, hands the same Page to every collaborator
public class PageHolder {
    private final Browser browser;                  // injected (one per thread)
    private BrowserContext ctx;
    private Page page;
    public PageHolder(Browser browser) { this.browser = browser; }

    public void open()  { ctx = browser.newContext(); page = ctx.newPage(); }
    public Page page()  { return page; }
    public boolean hasPage() { return page != null; }
    public void close() { if (ctx != null) ctx.close(); }   // closing context closes its pages
}
```

## THE SERVICE LAYER

The service is where the action is orchestrated and where assertions live. It
receives the page via the injected holder and uses **web-first, auto-retrying
assertions** — `assertThat(locator)`, never boolean getters (full rule in
`java-playwright-e2e`).

```java
import static com.microsoft.playwright.assertions.PlaywrightAssertions.assertThat;

public class TaskDetailsService {
    private final PageHolder pages;                 // injected — shared scenario page
    public TaskDetailsService(PageHolder pages) { this.pages = pages; }

    public void addNote(InternalNote note) {
        TaskDetailsPage page = pages.taskDetails(); // page object from playwright-page-objects
        page.noteTitle().fill(note.title());
        page.noteContent().fill(note.content());
        if (note.confidential()) page.confidential().check();
        page.saveNote().click();
        assertThat(page.successToast()).isVisible(); // web-first; waits & retries
    }
}
```

Service rules:
- One service per cohesive area of behavior; reuse it across many steps. **Don't put locators here** — call page-object methods.
- Assertions are **meaningful and unconditional**: never wrap `assertThat` in `if (x != null)` — provision the data so the assertion always runs (integrity rule in `java-playwright-e2e`).
- Use **soft assertions** when several checks should all report before the scenario fails; collect and assert at the end.
- API steps delegate to a REST-Assured service (`rest-assured-api-tests`); persistence checks delegate to a repository service (`database-validation`). The step never calls those APIs directly.

## REUSE BEFORE WRITING

DRY applies hardest to steps. Before adding a `@When`/`@Then`, **search the glue
packages** for an existing step that matches the Gherkin phrasing — duplicate steps
with near-identical regex are a common rot. Reuse a common step; if it's close,
generalize it (e.g. parameterize with `{string}`) rather than cloning. Keep
reusable, app-agnostic steps in the framework's common-steps package and
suite-specific steps in the suite.

## HANDOFF

- **`create-test-scenarios`** — produces the feature files your steps bind to. If a Gherkin line has no matching step, that skill (or this one) authors the step; if the scenario itself is wrong, fix it there.
- **`playwright-page-objects`** — owns locators and page-object methods. Your services call page methods; you never declare locators here. Missing page/locator -> hand off.
- **`rest-assured-api-tests`** — API steps delegate to its REST-Assured services and `page.route` mocking; store the response in `ScenarioContext` for later steps.
- **`database-validation`** — persistence-verification steps delegate to its repository/ORM services for three-way (input/API/DB) validation.
- **`e2e-framework-setup`** — owns the runner, `cucumber.glue`, parallel config, tracing/CI, and where injectable packages must live. Go there for "step not found / not glued," parallelism, or trace-viewer setup.
- **`java-playwright-e2e`** — orchestrator and single source of truth for DI, isolation, web-first assertions, naming/OOP, and test-integrity conventions referenced one-line above.
