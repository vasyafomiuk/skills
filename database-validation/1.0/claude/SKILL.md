---
name: database-validation
description: >-
  Verifies that data-changing actions actually persisted to the database, the
  final and most trustworthy leg of end-to-end validation. Use this skill when
  the user wants to: confirm a record was saved after a create/update/delete,
  assert audit fields like createdBy/updatedAt, build a repository or ORM
  abstraction so tests hold no raw SQL, provision required test data in setup
  helpers, complete the DB leg of three-way (input/API/DB) validation, or clean
  up test data created during a run. Reach for it whenever a UI success toast or
  a 200 response is being treated as proof of persistence — it is not. Also use
  it to wire database connection details from config/secret managers by key
  rather than hardcoding connection strings. Project-agnostic — no
  application-specific schema assumptions.
metadata:
  version: "1.0.0"
  category: Test Automation
  tags: [database, persistence, repository, orm, data-provisioning, three-way-validation, audit-fields]
---

# Database Validation

This skill owns the persistence-verification leg of the acceptance-criteria → passing-test
workflow. After a UI flow or an API call reports success, this is where you prove the change
actually reached the database. A green toast means the front end *thinks* it saved; a `200`
means the service *accepted* the request; only a row in the database confirms the data
*persisted* correctly with the right values, owner, and timestamps. This skill provides the
repository/ORM abstraction tests query through, the setup helpers that provision preconditions,
and the assertions that close three-way validation. For cross-cutting rules (stack/versions,
DI, test isolation, test-integrity, naming/OOP) follow the orchestrator `java-playwright-e2e`.

## WHEN TO USE

- After any create/update/delete: "verify the note was saved", "confirm the record persisted".
- "A UI test passes but I'm not sure the data is really in the DB."
- "Assert the audit fields" — `createdBy`, `updatedBy`, `createdAt`, `updatedAt`, status history.
- "We have raw SQL strings scattered in step definitions" — extract into a repository.
- "Provision test data in setup so the assertion never sees null."
- "Complete the three-way validation" — input vs API response vs DB row all agree.
- "Clean up the rows the test created."
- "The DB connection string is hardcoded — read it from config/secrets."

If the endpoints or API response shape you need to compare against don't exist yet, that is
`rest-assured-api-tests`' job — go there first. If there is no feature/page driving the action,
go to `create-test-scenarios` / `playwright-page-objects`. This skill assumes the action already
happens; it verifies the result.

## CORE PRINCIPLE: A toast is not proof

```
UI says "Saved"   →  front end believed it succeeded   (could be optimistic UI)
API returns 200   →  service accepted the request      (could fail downstream / async)
DB row matches    →  data actually persisted           ← only this is proof
```

Always assert the **value** against the source of truth, never just that the toast appeared.
Never weaken a DB assertion to make a test green — a missing or wrong row is a real bug.

## REPOSITORY / ORM ABSTRACTION — no raw SQL in tests

Step definitions and services must never contain SQL strings. Centralize three things:
**entities** (one definition per table), **queries** (reusable finder methods), and
**connection** (built once from config). Two valid approaches:

1. **JDBC + a thin repository** — lightweight; hand-written parameterized SQL lives only inside
   the repository class, mapped to a record/entity.
2. **An ORM (JPA/Hibernate, jOOQ, MyBatis)** — entity classes carry the mapping; the repository
   exposes typed finders. Prefer this when the schema is large or already modeled.

Either way the test layer sees only typed methods like `findNoteByTitle(...)`.

```java
/** One immutable entity per table — defined once, reused everywhere. */
public record TaskNote(
    long id, long recordId, String title, String content,
    String status, String createdBy, Instant createdAt, Instant updatedAt) {}
```

```java
/** Reusable queries live here. No SQL leaks past this class. */
public class TaskNoteRepository {
    private final Database db;                 // injected; owns the connection/pool

    public TaskNoteRepository(Database db) { this.db = db; }

    /** Latest note for a record matching a title, or null if absent. */
    public TaskNote findNoteByTitle(long recordId, String title) {
        return db.queryOne(
            "SELECT * FROM task_note WHERE record_id = ? AND title = ? ORDER BY created_at DESC",
            TaskNoteMapper::map, recordId, title);
    }

    public List<TaskNote> findNotesForRecord(long recordId) {
        return db.queryMany(
            "SELECT * FROM task_note WHERE record_id = ? ORDER BY created_at DESC",
            TaskNoteMapper::map, recordId);
    }

    /** Used by cleanup; returns affected rows so callers can verify. */
    public int deleteById(long id) {
        return db.update("DELETE FROM task_note WHERE id = ?", id);
    }
}
```

Rules:
- **Always parameterize** (`?`) — never string-concatenate values into SQL.
- One repository per aggregate/table area; finders return entities, not `ResultSet`.
- The `Database`/connection helper is a single injectable collaborator (see DI in the orchestrator),
  so every service shares one pool per run.

## CONNECTION FROM CONFIG / SECRETS — never hardcode

```java
public final class Database {
    private final DataSource ds;

    public Database(Config cfg) {
        HikariConfig hc = new HikariConfig();
        hc.setJdbcUrl(cfg.get("db.url"));            // key, not a literal
        hc.setUsername(cfg.get("db.user"));
        hc.setPassword(cfg.secret("db.password"));   // from env / secret manager at runtime
        this.ds = new HikariDataSource(hc);
    }
}
```

- Read URL/user/password **by key** from the per-environment config or secret manager. Never
  commit a connection string, host, or password; never inline them in a step or repository.
- Point at a stable environment you own/control — see config/secrets in `e2e-framework-setup`.

## PROVISION TEST DATA IN SETUP — so assertions never null-guard

A conditional assertion (`if (row != null) assertEquals(...)`) silently passes when the data is
missing — that hides bugs. Guarantee the precondition exists in setup via a helper, so the
assertion always runs unconditionally.

```java
/** Seed helper: returns a known-present row the scenario can rely on. */
public TaskNote seedNote(long recordId, String title) {
    long id = db.insert(
        "INSERT INTO task_note(record_id, title, content, status, created_by, created_at) "
        + "VALUES (?,?,?,?,?,?)",
        recordId, title, "seed content", "OPEN", "system",
        Instant.now().minus(7, ChronoUnit.DAYS));   // PAST date — see below
    return repo.findById(id);
}
```

### The past-dates trick

Many domains validate that a new status/timestamp cannot precede the previous one
("date cannot precede last status"). If you seed with `Instant.now()` and the action under test
also writes "now", ordering becomes ambiguous and the app may reject the update. **Seed
historical data with past dates** (e.g. `now().minus(7, DAYS)`) so the action's fresh timestamp
is unambiguously later and domain validation passes.

Then the assertion runs with no null-guard:

```java
TaskNote saved = repo.findNoteByTitle(recordId, "Quarterly review");
assertNotNull(saved, "note was not persisted");     // fail loudly if missing
assertEquals("Reviewed and ok", saved.content());
```

## AUDIT FIELDS — verify who/when, not just what

Persistence is more than the payload. Confirm the system stamped ownership and timing correctly.

```java
TaskNote saved = repo.findNoteByTitle(recordId, "Quarterly review");
assertEquals("Reviewed and ok", saved.content());
assertEquals(context.currentUser(), saved.createdBy());         // correct owner
assertTrue(saved.createdAt().isAfter(scenarioStart));           // stamped during the run
assertNull(saved.updatedAt());                                  // untouched on create
// On a subsequent update, assert updatedAt advanced and updatedBy changed:
assertTrue(updated.updatedAt().isAfter(updated.createdAt()));
```

- Normalize timestamps before comparing — compare `Instant`s in UTC, not formatted strings, to
  avoid timezone/format drift (full date-normalization guidance in `rest-assured-api-tests`).
- Assert **real values** against expectations, never just `instanceof`/"is defined" type checks.

## THE DB LEG OF THREE-WAY VALIDATION

For create/update flows, prove all three agree: what you sent, what the API returned, what the DB
stored. This skill owns the **DB** comparisons; `rest-assured-api-tests` provides the API response.

```java
// input → what the test submitted; api → REST Assured response DTO; dbRow → repository finder
TaskNote dbRow = repo.findNoteByTitle(recordId, input.title());
assertNotNull(dbRow);
assertEquals(input.status(), dbRow.status());   // DB persisted exactly what we sent
assertEquals(api.status(),  dbRow.status());     // API and DB agree
assertEquals(input.content(), dbRow.content());
```

If the API leg isn't built yet, you can still validate input → DB; add the API ↔ DB comparison
once `rest-assured-api-tests` produces the response object. Keep these assertions in a **service**
called from the step (step stays thin) — layering rules live in `cucumber-step-definitions`.

A reusable Gherkin step keeps the DB check business-readable:

```gherkin
Then Verify note "Quarterly review" persisted for the current record
```

```java
@Then("Verify note {string} persisted for the current record")
public void verifyNotePersisted(String title) {
    taskNoteDbService.assertNotePersisted(context.recordId(), title);
}
```

## CLEAN UP TEST DATA CREATED DURING THE RUN

Each scenario must leave the database as it found it, or parallel runs and reruns drift.

- **Track what you create.** Stash inserted ids on the injected `ScenarioContext` (or a per-scenario
  registry) as you seed/act.
- **Delete in an `@After` hook**, reverse dependency order (children before parents) to respect FKs.
- Prefer **targeted deletes by id** over `TRUNCATE`; never truncate shared tables in a shared env.

```java
@After
public void cleanupDb() {
    for (long id : context.createdNoteIds()) {
        repo.deleteById(id);
    }
}
```

- If the platform supports it, a per-scenario **transaction rolled back** in teardown is the
  cleanest isolation — but only when the action-under-test commits within that same transaction.
  Otherwise fall back to explicit id-based deletes.
- Hook ordering and the `@After` mechanism itself belong to `cucumber-step-definitions`; this skill
  supplies *what* to delete.

## RULES OF THUMB

- A toast/200 is never proof — assert the DB value.
- No SQL outside the repository; always parameterize; entities defined once.
- Provision in setup; assert unconditionally; no `if (row != null)` guards.
- Seed with past dates to dodge "date cannot precede last status" validation.
- Verify audit fields (createdBy/updatedAt), comparing UTC instants not strings.
- Connection details by config/secret key — never hardcoded.
- Track and delete what you create; leave the DB as you found it.

## HANDOFF

- **java-playwright-e2e** — orchestrator and single source of truth for stack/versions, DI,
  test isolation, test-integrity, and naming/OOP. Follow it for anything cross-cutting.
- **rest-assured-api-tests** — pairs with this skill for three-way validation; it supplies the API
  response DTO you compare DB rows against, and the input side of create/update flows.
- **cucumber-step-definitions** — calls DB services from thin steps; owns hook ordering (`@After`
  cleanup), `ScenarioContext`, and the service layer your repository plugs into.
- **e2e-framework-setup** — provides the per-environment config and secret-manager wiring the
  `Database` connection reads its keys from, plus the `db/` package location in the module layout.
- **create-test-scenarios** / **playwright-page-objects** — produce the feature/page that triggers
  the action whose persistence you verify; consult them if the action itself doesn't exist yet.
