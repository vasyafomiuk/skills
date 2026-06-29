---
name: create-test-scenarios
description: >-
  Produces ONLY Gherkin *.feature files, not a running or passing test — turning acceptance
  criteria into declarative scenarios — as many per AC as the criterion needs — fully tagged, with negative and edge
  cases designed to find bugs. Use this skill when the user
  wants to: convert a Jira ticket / story / acceptance criteria into feature files, write Gherkin
  or Cucumber scenarios, draft a feature spec from pasted AC or a screenshot of a ticket, add a
  Scenario Outline with Examples for data-driven variants, design negative/boundary test cases,
  or tag scenarios by type and suite (@ui/@api/@e2e, @regression/@smoke, @negative, @ACn).
  This is the spec-authoring step: it produces only feature files, not step definitions or page
  objects. Project-agnostic — no application-specific assumptions.
metadata:
  version: "1.2.0"
  category: Test Automation
  tags: [gherkin, cucumber, bdd, jira, acceptance-criteria, feature-files, test-design, scenario-outline]
---

# Create Test Scenarios (AC → Gherkin)

This skill owns the first leg of the AC → passing-test workflow: it converts acceptance
criteria into declarative Gherkin `*.feature` files. The features you write are the contract
the rest of the stack implements — `cucumber-step-definitions` binds your lines to glue and
`playwright-page-objects` supplies the locators those steps need. You produce **only** feature
files: no step definitions, no locators, no page objects, no runner config. For stack versions,
locator priority, web-first assertions, and naming/OOP rules, the orchestrator
(`java-playwright-e2e:orchestrator`) is the single source of truth — point there rather than
re-deriving them.

## WHEN TO USE

- "Turn this Jira ticket / story into feature files" or "write the Gherkin for AC-123".
- The user pastes acceptance criteria (or attaches a ticket / a screenshot of one) and wants scenarios.
- "Add a Scenario Outline / Examples table for these input variants."
- "Design negative and edge cases for this feature" — empty/null/boundary/injection/unauthorized.
- "Tag these scenarios" by type (@ui/@api/@e2e), suite (@regression/@smoke), @negative, AC ids.
- Any request to author or revise a `.feature` file *before* steps and pages exist.

Do **not** use this skill to implement step definitions, locators, or page objects — hand off
to the siblings below for that.

## STEP 1 — ACQUIRE THE ACCEPTANCE CRITERIA (ordered fallback)

Get the real AC before writing anything. Try each option in order; fall through only on failure.

1. **Jira / Atlassian MCP, if connected.** Detect it: scan the available tools for a Jira or
   Atlassian server (tool names containing `jira` / `atlassian` / `confluence`). If they appear
   only as deferred names, load their schemas with `ToolSearch` (e.g. query `jira atlassian issue`
   or `select:<exact_tool_name>`), then use them to search for the issue key and fetch its
   description + acceptance-criteria field. If no such tool exists, go to step 2.
2. **Ask the user.** Request the AC text directly, or an attached ticket export / a screenshot of
   the ticket. Read a screenshot with the Read tool and transcribe the AC from it.
3. **Parse and CONFIRM.** Distil the AC into a discrete, numbered, testable list (one row per
   criterion). Echo that list back to the user and get confirmation **before** writing scenarios.
   This catches misread or ambiguous AC while it is cheap to fix.

Example confirmation back to the user:

```
I read these acceptance criteria — confirm before I write scenarios:
  AC1: A user with the "admin" role can add an internal note to a record.
  AC2: Saving a note with an empty title is rejected with "Title is required".
  AC3: Notes longer than 2000 chars are truncated to 2000 on save.
Anything to add or correct?
```

**Also ask for existing examples.** Before designing scenarios, ask whether the project already
has `.feature` files or scenarios. If any exist, read them and follow their structure, step
vocabulary, tag conventions, and folder layout — match the house style rather than imposing the
templates below (which are the fallback when no examples exist).

## STEP 2 — DESIGN THE SCENARIO SET

Plan scenarios before typing Gherkin.

- **Cover every AC fully — with as many scenarios as it needs.** A simple AC is usually one
  scenario (setup → action → assertions); an AC with distinct flows, roles, or outcomes can span
  several focused scenarios. Don't merge unrelated AC into one scenario, and don't split just to
  inflate the count. Tag every scenario with its AC id so coverage stays traceable.
- **Then go beyond the AC to find bugs.** For each AC add negative/edge scenarios — that is where
  real defects live, not the happy path. Walk this checklist per feature:
  - **Empty / null / missing** required field.
  - **Boundary**: min, max, max±1, zero, negative, very long input (e.g. 2000 vs 2001 chars).
  - **Special characters & encoding**: unicode, emoji, leading/trailing whitespace.
  - **Injection**: SQL-ish (`'; DROP TABLE`) and XSS (`<script>alert(1)</script>`) payloads —
    assert they are rejected or safely escaped, never executed.
  - **Authorization**: a user *without* the required role/permission is blocked.
  - **Concurrency / duplicates**: same record acted on twice, duplicate unique key.
  - **Error handling**: backend error / timeout surfaces a clear message, not a stack trace.
- One feature file per cohesive capability (a Jira story usually maps to one feature). Group its
  AC scenarios together; keep `api/`, `ui/`, `e2e/` features in their respective folders.

## STEP 3 — WRITE DECLARATIVE GHERKIN

Describe **what** the user achieves, never **how** the UI does it. No selectors, CSS, ids, URLs,
or click coordinates ever appear in Gherkin — those belong to the page objects. Every step is a
business-level sentence a product owner could read.

```gherkin
@ui @regression
Feature: Internal notes on a record

  Background:
    Given User is set to "admin"
    And User loads "TaskDetails" page

  @AC1
  Scenario: AC1 - User can add an internal note
    When User enters following info on "TaskDetails" page:
      | ElementName | Value            |
      | noteTitle   | Quarterly review |
      | noteContent | Reviewed and ok  |
    And User clicks "saveNote" on "TaskDetails" page
    Then Verify "successToast" is visible on "TaskDetails" page
    And Verify note "Quarterly review" is saved for the current record

  @AC2 @negative
  Scenario Outline: AC2 - Saving is blocked for invalid titles
    When User enters "<title>" in "noteTitle" on "TaskDetails" page
    And User clicks "saveNote" on "TaskDetails" page
    Then Verify "<error>" is visible on "TaskDetails" page
    Examples:
      | title                       | error            |
      |                             | titleRequired    |
      | <script>alert(1)</script>   | titleInvalidChar |
```

Declarative vs imperative:

```gherkin
# GOOD — business intent; the page object owns the locator
When User clicks "saveNote" on "TaskDetails" page

# BAD — leaks the DOM into the spec; breaks the moment markup changes
When User clicks the element with css "#btn-1234"
```

Authoring rules:

- **Background** holds setup shared by *every* scenario in the file (login, navigation, common
  preconditions). Don't put one-off setup or assertions in `Background`.
- **Scenario Outline + Examples** for the same flow over different data — invalid-input
  variants, boundary tables, role matrices. One Example row per data case; keep the steps
  identical. A blank cell expresses "empty input".
- **Reuse existing steps before inventing new lines.** Search the project's common-steps package
  first and phrase new steps to match the established vocabulary (same verbs, same parameter
  shape). Consistent phrasing lets `cucumber-step-definitions` reuse one glue method across many
  features. If you can't see the steps, propose lines that follow the patterns shown here and
  flag them for the step-definition author to confirm or map.
- **DataTable** for multi-field input; **Examples** for the same flow over varying data. Keep
  tables narrow and named (`ElementName`/`Value`), not positional.
- Keep each scenario short — roughly 3–7 steps after `Background`. If one sprawls, split it into
  focused scenarios (each still tagged with the AC id) rather than forcing one giant scenario.

## STEP 4 — TAG EVERY SCENARIO

Tags drive selection and reporting downstream. Apply all relevant axes:

- **Type** (exactly one): `@ui`, `@api`, or `@e2e`.
- **Suite**: `@smoke` (fast critical path) and/or `@regression` (full coverage).
- **AC id** (exactly one per scenario): `@AC1`, `@AC2`, … — the traceability link back to the ticket.
- **Nature**: `@negative` for failure/rejection paths; `@security` for authz/injection cases.

```gherkin
@api @regression @AC3 @negative
Scenario: AC3 - Rejecting a note over the length limit
```

Put `Feature`-wide tags above `Feature:` (they apply to all scenarios); put scenario-specific
tags directly above the scenario. Every scenario must carry its AC id so coverage stays traceable
to the acceptance criteria (several scenarios may share one `@ACn`).

## QUALITY BAR (before handing off)

- [ ] AC parsed into a numbered list and **confirmed with the user**.
- [ ] Every AC fully covered by one or more scenarios; no unrelated AC merged into one scenario.
- [ ] Negative/edge scenarios added (empty, boundary, special-char, injection, unauthorized,
      concurrency, error handling) to hunt for bugs.
- [ ] Declarative — no selectors/CSS/ids/URLs in any step; every step is business-level.
- [ ] Existing common steps reused; new lines match the established vocabulary.
- [ ] `Background` for shared setup; `Scenario Outline` + `Examples` for data-driven variants.
- [ ] Every scenario tagged: one `@type`, suite tag(s), `@ACn`, plus `@negative`/`@security` where apt.
- [ ] Exact expected error messages from the AC captured in the assertions/Examples.
- [ ] Files placed under `src/test/resources/features/{ui,api,e2e}/`.

## HANDOFF

- **the orchestrator** (`java-playwright-e2e:orchestrator`) — the single source of truth for shared
  conventions (stack/versions, locator priority, web-first assertions, DI, isolation, naming/OOP).
  Follow it; defer to it for any cross-cutting rule rather than restating it here.
- **cucumber-step-definitions** — implements the glue for the steps you wrote (parses params,
  delegates to services, wires DI/ScenarioContext/hooks). Hand off your finished `.feature`
  files plus any new step phrasings that need glue. This is the next step after your features exist.
- **playwright-page-objects** — supplies the locators and page objects the steps reference (e.g.
  the `"TaskDetails"` page and its `noteTitle`/`saveNote` elements named in your Gherkin). The
  business element names you choose become the page-object field names they create.
- **rest-assured-api-tests** — when an `@api` scenario you designed needs request/response and
  mocking glue rather than browser steps.
- **database-validation** — when an AC requires confirming persistence ("…is saved"); your
  `Then` line ("Verify note … is saved") is realised by that skill's repository/ORM checks.
