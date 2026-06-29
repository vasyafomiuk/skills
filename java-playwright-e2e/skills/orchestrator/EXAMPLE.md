# Worked Example — One Feature Through All Seven Skills

A single end-to-end walk-through of the simplest possible feature: **adding an
internal note to a TaskDetails record**. It threads every sub-skill of the
orchestrator (`java-playwright-e2e:orchestrator`) in workflow order so a reader can
see how the pieces snap together. The snippets are deliberately small and are
mutually consistent — the same `ScenarioContext`, `Config`, page object, service,
and repository the individual skills show. Copy them and they compile.

The shared motif throughout: a `TaskDetails` page; an `InternalNote{title, content,
confidential}` record; the business element names `noteTitle`, `noteContent`,
`saveNote`, `successToast`.

---

## 1. Acceptance criterion — [create-test-scenarios]

Plain text, parsed and confirmed with the user before any Gherkin is written:

```
AC1: A user with the "admin" role can add an internal note (title + content) to a
     TaskDetails record; on save a success toast appears and the note persists.
AC2 (negative): Saving a note with an empty title is rejected with "Title is required."
```

## 2. The .feature file — [create-test-scenarios]

One `@ui @AC1` happy-path scenario and one `@negative` scenario. Declarative — no
selectors, ids, or URLs leak into the spec.

```gherkin
@ui @regression
Feature: Internal notes on a TaskDetails record

  Background:
    Given User is set to "admin"
    And User loads "TaskDetails" page

  @AC1
  Scenario: AC1 - User can add an internal note
    When User adds an internal note
      | title            | content         | confidential |
      | Quarterly review | Reviewed and ok | false        |
    Then Verify "successToast" is visible on "TaskDetails" page
    And Verify note "Quarterly review" persisted for the current record

  @AC2 @negative
  Scenario: AC2 - Empty title is rejected
    When User adds an internal note
      | title | content         | confidential |
      |       | Reviewed and ok | false        |
    Then Verify "titleRequired" is visible on "TaskDetails" page
```

## 3. The page object — [playwright-page-objects]

`TaskDetailsPage extends CommonPage`, locators private and annotation-driven,
discovered from the real page (never guessed). `@VerifyAt` + `@VerifyBy` prove the
right page rendered before acting.

```java
public class TaskDetailsPage extends CommonPage {

    @VerifyAt(UrlType.CONTAINS)
    public static final String URL = "/task-details";

    @VerifyBy
    public static final String ROOT = "[data-testid='task-details']";

    @FindBy(label = "Note Title")            private Locator noteTitle;
    @FindBy(label = "Note Content")          private Locator noteContent;
    @FindBy(role = "button", name = "Save")  private Locator saveNote;
    @FindBy(css = "[role='alert']")          private Locator successToast;

    public TaskDetailsPage(Page page) { super(page); }

    @Override
    public void waitForPageToLoad() {
        assertThat(page).hasURL(Pattern.compile(".*" + Pattern.quote(URL)));  // @VerifyAt
        assertThat(page.locator(ROOT)).isVisible();                           // @VerifyBy
    }

    public void addNote(InternalNote note) {                 // behavior, not raw Locators
        waitForPageToLoad();
        noteTitle.fill(note.title());
        noteContent.fill(note.content());
        if (note.confidential()) page.getByLabel("Confidential").check();
        saveNote.click();
    }

    public Locator successToast() { return successToast; }   // returned only for web-first assertThat
}
```

## 4. Hooks + PageHolder — [cucumber-step-definitions]

Hooks are glue too, so they get DI. `PageHolder` owns the per-scenario
`BrowserContext`/`Page`; `@Before` opens it, `@After` closes it. Every injected
collaborator therefore sees the same scenario's page.

```java
public class Hooks {
    private final PageHolder pages;                 // injected per scenario
    public Hooks(PageHolder pages) { this.pages = pages; }

    @Before("@ui or @e2e")
    public void startBrowser(Scenario s) { pages.open(); }      // new context + page for THIS scenario

    @After("@ui or @e2e")
    public void tearDown(Scenario s) {
        if (s.isFailed() && pages.hasPage()) {
            s.attach(pages.page().screenshot(), "image/png", "failure");
        }
        pages.close();                              // closing context closes its pages — no leaks
    }
}
```

## 5. Thin steps -> service — [cucumber-step-definitions]

The step parses the `DataTable` into the typed `InternalNote` and delegates; the
service owns the action and the web-first assertion. No `if`, loop, or `assertThat`
ever lives in the step.

```java
public record InternalNote(String title, String content, boolean confidential) {}

public class TaskDetailsSteps {
    private final TaskDetailsService taskService;   // injected collaborators
    private final ScenarioContext context;

    public TaskDetailsSteps(TaskDetailsService taskService, ScenarioContext context) {
        this.taskService = taskService;
        this.context = context;
    }

    @When("User adds an internal note")
    public void userAddsNote(InternalNote note) {   // typed via a registered @DataTableType
        taskService.addNote(note);                  // delegate; no logic here
    }
}
```

```java
import static com.microsoft.playwright.assertions.PlaywrightAssertions.assertThat;

public class TaskDetailsService {
    private final PageHolder pages;                 // injected — shared scenario page
    public TaskDetailsService(PageHolder pages) { this.pages = pages; }

    public void addNote(InternalNote note) {
        TaskDetailsPage page = pages.taskDetails(); // page object from playwright-page-objects
        page.addNote(note);
        assertThat(page.successToast()).isVisible();// web-first; waits and retries
    }
}
```

## 6. The @api three-way check — [rest-assured-api-tests + database-validation]

Same note, proven across all three legs: what we sent (input), what the API
returned, and what the DB stored. The API leg POSTs, stores the `Response` in the
shared `ScenarioContext`, and asserts; the DB leg reads it back through the
repository — a toast is never proof.

```java
// API leg — rest-assured-api-tests
public class ApiSteps {
    private final ApiClient api;
    private final ScenarioContext context;          // injected per scenario
    public ApiSteps(ApiClient api, ScenarioContext context) { this.api = api; this.context = context; }

    @When("User POSTs {string} with body {string}")
    public void userPosts(String endpoint, String fixtureName) {
        context.setResponse(api.post(endpoint, Fixtures.fixture(fixtureName)));   // store once
    }

    @Then("API status code is {int}")
    public void statusIs(int expected) {
        assertEquals(expected, context.getResponse().statusCode());              // assert later
    }
}
```

```java
// DB leg — database-validation. Three pairwise asserts catch a leg that disagrees.
public void assertThreeWay(InternalNote input) {
    NoteResponse api = context.getResponse().as(NoteResponse.class);
    TaskNote saved   = repo.findNoteByTitle(context.recordId(), input.title());  // no raw SQL in tests
    assertNotNull(saved, "note was not persisted");
    assertEquals(input.title(), api.title());        // API reflects what we sent
    assertEquals(api.title(),   saved.title());       // API reflects the DB
    assertEquals(input.content(), saved.content());   // DB persisted what we sent
    assertEquals(context.currentUser(), saved.createdBy());                      // audit field
    context.createdNoteIds().add(saved.id());        // track for @After cleanup
}
```

`ApiClient.post` reads `Config.apiBaseUrl()` / `Config.token()` (static, never an
injected instance); the `@After` cleanup deletes every id stashed in
`context.createdNoteIds()`, leaving the database as it was found.

## 7. Run it green — [e2e-framework-setup]

With the scaffold in place and browsers installed, run just this feature's UI
scenario by tag:

```bash
mvn test -Denv=dev -Dcucumber.filter.tags="@ui and @AC1"
```

Drop the tag filter (`mvn test -Denv=dev`) to run the whole suite, or swap the
filter to `"@api and @AC1"` for the API leg.

---

Pinned example versions used by the pom this assumes — playwright 1.55.0,
cucumber 7.20.1, junit-platform 1.11.4, rest-assured 5.5.0, surefire 3.5.2,
HikariCP 6.2.1, postgresql (JDBC) 42.7.4. Known-good example values — verify the
current release on Maven Central and bump.

> Lost on which step a task belongs to? Go back to the orchestrator
> (`java-playwright-e2e:orchestrator`) and its routing table — it dispatches each
> phase above to the sub-skill that owns the deep how-to.
