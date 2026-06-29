---
name: playwright-page-objects
description: >-
  Discovers real, stable Playwright (Java) locators for a Playwright+Cucumber suite
  and builds annotation-driven Page Objects from them. Use this skill when the user
  wants to: find or harden Playwright (Java) element locators for a page, decide
  between getByRole / getByLabel / getByText / getByTestId / CSS / XPath, fix
  strict-mode "resolved to N elements" errors, scope a locator to a row/card/dialog
  with filter or nth, snapshot a page with Playwright MCP or codegen to read the
  accessibility tree, read Playwright (Java) locators from a screenshot/HTML for the
  Playwright+Cucumber suite, design a Page Object with @FindBy / @VerifyAt / @VerifyBy
  and a CommonPage base, add axe-core accessibility assertions, or confirm the right
  page rendered before acting. Project-agnostic — no application-specific assumptions.
metadata:
  version: "1.2.0"
  category: Test Automation
  tags: [playwright, locators, page-object, accessibility, selectors, codegen, playwright-mcp, strict-mode]
---

# Playwright Page Objects

This skill owns the **locator layer** of the AC -> passing-test workflow: turning a
live page into resilient, user-facing locators and packaging them into
annotation-driven Page Objects. It sits between scenario authoring and step glue —
`create-test-scenarios` produces the Gherkin that names elements; you produce the
Page Objects that those element names resolve to; `cucumber-step-definitions`
services then consume your pages to drive the browser. The single largest source of
flaky tests is a badly chosen locator, so the whole job here is: discover from the
real page, prefer attributes that survive markup churn, and encapsulate.

Cross-cutting rules (stack/versions, web-first assertions, DI, test isolation,
naming/OOP) live in the orchestrator (`java-playwright-e2e:orchestrator`) — this skill
states each only as a one-line principle and points there.

## WHEN TO USE

Use this skill when the user wants to:
- Find locators for a page, screen, or component ("get me the selectors for the login form").
- Choose between `getByRole`, `getByLabel`, `getByText`, `getByTestId`, CSS, or XPath.
- Fix a strict-mode error ("locator resolved to 3 elements", action throwing on ambiguous match).
- Scope a locator to a specific row / card / dialog, or chain & filter instead of a brittle compound selector.
- Snapshot a page via Playwright MCP, run codegen/inspector, or read locators from a screenshot/HTML.
- Build or refactor a Page Object: `@FindBy` fields, a `CommonPage` base, `@VerifyAt` / `@VerifyBy`.
- Confirm the correct page rendered before interacting with it.

This skill does **not** write feature files (see `create-test-scenarios`) or step
definitions/services (see `cucumber-step-definitions`). It stops at the page object.

## STEP 1 — ACQUIRE THE PAGE (ordered fallback chain)

Never guess locators from memory. Get the real accessibility tree / DOM first.
Try each option in order; fall to the next only when the current one is unavailable.

1. **Playwright MCP (preferred).** Detect it: scan the available tools for Playwright
   MCP entries (e.g. `browser_navigate`, `browser_snapshot`, `browser_close`). If they
   are deferred, load them with `ToolSearch` (query `"playwright browser snapshot"` or
   `"select:browser_snapshot,browser_navigate,browser_close"`). Then:
   `browser_navigate` to the URL -> `browser_snapshot` to read the accessibility tree
   (roles, accessible names, element refs) -> pick role/label/text locators from it ->
   `browser_close`. The snapshot's roles and names map directly onto `getByRole` /
   `getByLabel`, which is exactly the priority you want.
2. **Codegen / inspector** against a live URL the user provides:
   `mvn exec:java -e -D exec.mainClass=com.microsoft.playwright.CLI -D exec.args="codegen <url>"`.
   Codegen records actions and emits role/text/test-id-first locators. For a running
   test, `PWDEBUG=1` opens the inspector and highlights locators live.
3. **Screenshot** the user provides — read visible roles (button/link/heading), field
   labels, and static text directly off the image; translate to `getByRole`/`getByLabel`/`getByText`.
4. **Raw HTML source** the user provides — parse it, but still extract user-facing
   attributes (`role`, `aria-label`, `<label for>`, visible text) rather than `div` nesting.
5. **Ask the user** for the live URL or the page HTML when none of the above is available.

> Always prefer user-facing attributes (role / accessible name / label / visible text)
> over DOM structure. They are what the user perceives and they survive class renames,
> wrapper `<div>` churn, and component-library upgrades.

## STEP 2 — CHOOSE THE LOCATOR (priority order)

Pick the highest option on this list that uniquely identifies the element. XPath is a
last resort. This priority order is shared across the suite — see the orchestrator.

| # | Locator | Use for |
|---|---------|---------|
| 1 | `getByRole(role, name=...)` | buttons, links, headings, checkboxes — also doubles as an a11y check |
| 2 | `getByLabel(...)` | form fields with a visible/`aria` label |
| 3 | `getByText(...)` | visible static, non-interactive text |
| 4 | `getByPlaceholder(...)` | inputs identified only by placeholder |
| 5 | `getByTestId(...)` | when devs added `data-testid` and nothing user-facing is stable |
| 6 | scoped CSS / `[aria-label="..."]` | structural fallback, kept short and attribute-based |
| 7 | XPath | only when nothing above works; brittle, avoid |

```java
page.getByRole(AriaRole.BUTTON, new Page.GetByRoleOptions().setName("Save"));
page.getByLabel("Note Title");
page.getByText("Reviewed and ok");
page.getByTestId("task-details");
```

Notes:
- `getByRole` matches on the **accessible name**; that is the snapshot's `name`, not the
  DOM text. Prefer it for anything interactive.
- Use `setExact(true)` when a substring name would over-match (`"Save"` vs `"Save and close"`).
- Reach for `getByTestId` only after user-facing options fail — it tests an artifact devs
  added, not what the user sees, but it is far stabler than positional CSS/XPath.

## STEP 3 — SCOPE, CHAIN & FILTER (strict mode)

Playwright locators run in **strict mode**: an action (`click`, `fill`, ...) throws if the
locator resolves to more than one element. Treat that as a signal to narrow, never to
grab the first match. Three tools, in order of preference:

**Scope to a parent**, then locate within it — the cleanest fix for "N matches":
```java
// Act inside the dialog, not the whole page
Locator dialog = page.getByRole(AriaRole.DIALOG, new Page.GetByRoleOptions().setName("Confirm delete"));
dialog.getByRole(AriaRole.BUTTON, new Locator.GetByRoleOptions().setName("Delete")).click();
```

**Filter by text or by a nested locator** to pick the right repeated element (row/card):
```java
// Scope to the row that contains "Product 2", then act within it
page.getByRole(AriaRole.LISTITEM)
    .filter(new Locator.FilterOptions().setHasText("Product 2"))
    .getByRole(AriaRole.BUTTON, new Locator.GetByRoleOptions().setName("Add to cart"))
    .click();

// Filter by a nested control rather than raw text
page.getByRole(AriaRole.LISTITEM)
    .filter(new Locator.FilterOptions().setHas(
        page.getByRole(AriaRole.HEADING, new Page.GetByRoleOptions().setName("Product 2"))));
```

**`nth` / `first` / `last`** only when order is genuinely meaningful and stable — it is
positional, so use it last:
```java
page.getByRole(AriaRole.LISTITEM).nth(2);   // brittle if list order changes
```

Strict-mode triage when an action throws "resolved to N elements":
1. Add a `getByRole(...).setName(...)` to disambiguate by accessible name.
2. Scope to the nearest meaningful parent (dialog / row / card / section).
3. `filter(setHasText / setHas)` to pin the right repeated container.
4. Only then `nth(...)` — and leave a note that order is load-bearing.

Chaining beats a single brittle compound CSS selector: `div.list > li:nth-child(3) .btn`
breaks on any layout change; the scoped/filtered role chain above does not.

## STEP 4 — BUILD THE PAGE OBJECT (annotation-driven)

Every page extends a shared base and declares locators as **private** annotated fields.
Three annotations carry the metadata that generic steps and page-load verification need.
Encapsulation rule (locators private, no leaking getters/setters) is shared — see
the orchestrator.

```java
@Retention(RUNTIME) @Target(FIELD)
public @interface FindBy {        // how to locate — mirrors the priority order
    String role()   default "";   // preferred
    String name()   default "";   // accessible name for an interactive role (supersedes monolith role+text)
    String label()  default "";
    String text()   default "";   // visible static text (non-interactive)
    String testId() default "";
    String css()    default "";
    String xpath()  default "";   // discouraged
    boolean all()   default false;// true -> resolve to a list of matches
}

@Retention(RUNTIME) @Target(FIELD)
public @interface VerifyAt { UrlType value(); }  // expected URL match: FULL / PARTIAL / CONTAINS

@Retention(RUNTIME) @Target(FIELD)
public @interface VerifyBy { }                   // a selector that proves the page rendered
```

```java
/** Shared locators, page-load wait, and helpers for every page. */
public abstract class CommonPage {
    protected final Page page;
    protected CommonPage(Page page) { this.page = page; }

    /** Confirms the right page rendered before any action runs. */
    public abstract void waitForPageToLoad();
}

public class TaskDetailsPage extends CommonPage {

    @VerifyAt(UrlType.CONTAINS)
    public static final String URL = "/task-details";

    @VerifyBy
    public static final String ROOT = "[data-testid='task-details']";

    @FindBy(label = "Note Title")                  private Locator noteTitle;
    @FindBy(label = "Note Content")                private Locator noteContent;
    @FindBy(role = "button", name = "Save")        private Locator saveNote;
    @FindBy(css = "[role='alert']")                private Locator successToast;

    public TaskDetailsPage(Page page) { super(page); }

    // waitForPageToLoad() is implemented in STEP 5 below (asserts @VerifyAt + @VerifyBy)
}
```

Rules:
- **All locators live in page objects** — never in feature files or step classes.
- Locators are `private`; no `_` prefix, no getters/setters. Generic steps resolve fields
  by name; explicit-DI projects expose **behavioral methods** (`addNote(...)`), not the raw `Locator`.
  A page may expose one read-only `Locator` accessor (e.g. `successToast()`) for the **service's**
  web-first assertion — never expose a locator to a step or feature file.
- One `@FindBy` per field, expressing a single chosen strategy from STEP 2. Set `all = true`
  for a list-valued locator the caller will iterate or count.
- **Compose**, don't inflate: model a repeated card/row as its own small component page and
  build the screen from those, rather than one page with a hundred fields.
- For a multi-field action, expose a method that takes a typed input object (a `record`),
  not a long primitive parameter list.
- **Match existing page objects.** If the project already has page objects, ask for / read one
  first and follow its base class, annotation usage, and naming rather than introducing a new style.

## STEP 5 — VERIFY THE PAGE BEFORE ACTING

Confirm you are on the intended page before driving it; acting on a half-rendered or wrong
page is a top flakiness source. `@VerifyAt` + `@VerifyBy` give a single check both pieces
of evidence — the URL **and** a rendered anchor element:

```java
@Override
public void waitForPageToLoad() {
    assertThat(page).hasURL(Pattern.compile(".*" + Pattern.quote(URL)));   // @VerifyAt
    assertThat(page.locator(ROOT)).isVisible();                            // @VerifyBy
}
```

Use Playwright's **auto-retrying** `assertThat(...)` here (it polls until the condition
holds), never a one-shot boolean getter. Web-first assertions are a shared convention —
full rule in the orchestrator. Call `waitForPageToLoad()` at the top of any service
flow that targets this page, before the first `fill`/`click`.

## ACCESSIBILITY (axe-core)

Locators built from roles and accessible names already lean on the a11y tree; axe-core
turns that into an explicit assertion. Run a scan against the live `page` and fail the
test on any violation, using the same web-first style as page verification. The axe-core
Playwright dependency is pinned in **e2e-framework-setup**.

```java
import com.deque.html.axecore.playwright.AxeBuilder;
import com.deque.html.axecore.results.AxeResults;

AxeResults results = new AxeBuilder(page).analyze();
assertTrue(results.getViolations().isEmpty(), results.getViolations().toString());
```

- **Scope** the scan to the region under test rather than the whole document:
  ```java
  AxeResults results = new AxeBuilder(page).include(List.of("main")).analyze();
  ```
- **Suppress** known, triaged issues instead of asserting on a noisy full result —
  keep the suppression list short and reviewed:
  ```java
  AxeResults results = new AxeBuilder(page).disableRules(List.of("color-contrast")).analyze();
  ```
- **Tag** accessibility scenarios with `@a11y` so they can be run (or skipped) as a slice.
- **Reuse** the scan behind one generic step backed by a service, so feature files stay
  declarative and the AxeBuilder wiring lives in one place:
  ```gherkin
  Then User sees no accessibility violations on "TaskDetails" page
  ```
  The step resolves the named page and delegates to a service method that runs the
  `AxeBuilder(page).analyze()` assertion above — no axe wiring in the step or feature file.

## ANTI-PATTERNS

- Guessing selectors without acquiring the page (STEP 1) — leads to non-existent or ambiguous locators.
- Auto-generated absolute XPath (`/html/body/div[2]/div[3]/...`) — breaks on the first layout change.
- Compound positional CSS (`.col > div:nth-child(4) > button`) — chain + filter instead.
- Swallowing strict-mode by appending `.first()` — narrow with role+name / scope / filter first.
- `public Locator getSaveNote()` leaking a locator to steps — expose behavior, keep locators private.
- Locator strings hardcoded in step definitions or `.feature` files — they belong in the page object only.

## HANDOFF

- **the orchestrator (`java-playwright-e2e:orchestrator`)** — the single source of truth for
  locator-priority, encapsulation, web-first assertions, DI, and isolation. Defer there for any
  cross-cutting rule.
- **create-test-scenarios** — upstream. Its Gherkin names elements (`"saveNote"`,
  `"successToast"`); your `@FindBy` field names should match those names so steps resolve cleanly.
- **cucumber-step-definitions** — downstream consumer. Its services receive your page objects by
  DI and call the behavioral methods you expose; hand it pages whose locators are already scoped and verified.
- **rest-assured-api-tests** — when a scenario needs `page.route` mocking or API seeding rather
  than UI locators, that work lives there, not here.
- **e2e-framework-setup** — owns where the `annotation/` and `pageobject/` packages sit in the
  Maven module layout your page objects compile into.
