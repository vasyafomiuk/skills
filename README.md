# skills

A collection of reusable [Claude](https://claude.com/claude-code) skills.

## Layout

Each skill lives under its own directory, versioned, and namespaced by the
agent/tool it targets:

```
<skill-name>/
└── <version>/
    └── <agent>/
        └── SKILL.md
```

> Note: a skill's identity comes from the `name:` field in its SKILL.md
> frontmatter, not its folder. The orchestrator skill is named
> `java-playwright-e2e` but is versioned under the `java-playwright-cucumber-e2e/`
> directory alongside its earlier monolithic releases.

## Java + Playwright + Cucumber E2E

A test-automation capability for building end-to-end suites with Java, Playwright,
Cucumber (BDD), and REST Assured. It ships in two forms:

- **Monolith** — one comprehensive reference skill (versions 1.0 → 1.1.0).
- **Decomposed set (1.2)** — the orchestrator plus six focused, independently
  triggerable sub-skills, each owning one phase of the AC → passing-test workflow.
  Start at the **orchestrator**, which routes to the right sub-skill and holds the
  shared conventions every sub-skill obeys.

### Orchestrator & monolith

| Skill (`name`) | Version | Path | Role |
|----------------|---------|------|------|
| java-playwright-e2e | **1.2.0** (latest) | [java-playwright-cucumber-e2e/1.2.0](java-playwright-cucumber-e2e/1.2.0/claude/SKILL.md) | Entry point: end-to-end workflow, shared conventions, routing to the six sub-skills below. |
| java-playwright-e2e | 1.1.0 | [java-playwright-cucumber-e2e/1.1.0](java-playwright-cucumber-e2e/1.1.0/claude/SKILL.md) | Comprehensive all-in-one monolith (frozen deep reference). |
| java-playwright-e2e | 1.0 | [java-playwright-cucumber-e2e/1.0](java-playwright-cucumber-e2e/1.0/claude/SKILL.md) | Original monolith. |

### Focused sub-skills (used by the 1.2 orchestrator)

| Skill | Version | Path | Focus |
|-------|---------|------|-------|
| create-test-scenarios | 1.0 | [create-test-scenarios](create-test-scenarios/1.0/claude/SKILL.md) | Jira / acceptance criteria → declarative, tagged Gherkin feature files (Jira MCP, else ask). |
| playwright-page-objects | 1.0 | [playwright-page-objects](playwright-page-objects/1.0/claude/SKILL.md) | Discover stable locators (Playwright MCP → codegen → screenshot → HTML → ask) and build annotation-driven Page Objects. |
| cucumber-step-definitions | 1.0 | [cucumber-step-definitions](cucumber-step-definitions/1.0/claude/SKILL.md) | Thin step glue → service layer, picocontainer DI, ScenarioContext, hooks & per-scenario lifecycle. |
| rest-assured-api-tests | 1.0 | [rest-assured-api-tests](rest-assured-api-tests/1.0/claude/SKILL.md) | REST Assured API/hybrid scenarios, three-way validation, `page.route` mocking. |
| database-validation | 1.0 | [database-validation](database-validation/1.0/claude/SKILL.md) | Verify persistence via repository/ORM (no raw SQL); audit fields and the DB leg of three-way validation. |
| e2e-framework-setup | 1.0 | [e2e-framework-setup](e2e-framework-setup/1.0/claude/SKILL.md) | Scaffold the Maven multi-module framework: deps, runner (JUnit 5/TestNG), parallelism, config/secrets, CI. |
