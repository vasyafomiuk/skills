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

Related skills can be grouped under a **suite** folder
(`<suite>/<skill-name>/<version>/<agent>/SKILL.md`). A skill's identity comes from
the `name:` field in its SKILL.md frontmatter, not its folder path.

## Java + Playwright + Cucumber E2E

A test-automation capability for end-to-end suites with Java, Playwright, Cucumber
(BDD), and REST Assured. It exists in two independent forms:

### 1. Monolith — `java-playwright-cucumber-e2e/` (legacy reference)

One comprehensive all-in-one skill. Frozen; kept as a deep reference.

| Version | Path |
|---------|------|
| 1.1.0 | [java-playwright-cucumber-e2e/1.1.0](java-playwright-cucumber-e2e/1.1.0/claude/SKILL.md) |
| 1.0 | [java-playwright-cucumber-e2e/1.0](java-playwright-cucumber-e2e/1.0/claude/SKILL.md) |

### 2. Modular suite — `java-playwright-e2e/` (current, v1.2)

The monolith decomposed into focused, independently-triggerable skills, all
released together at **v1.2**. **Start at the orchestrator**, which owns the
end-to-end workflow + shared conventions and routes to the six sub-skills.

| Skill (`name`) | Version | Path | Focus |
|----------------|---------|------|-------|
| java-playwright-e2e *(orchestrator)* | 1.2 | [java-playwright-e2e/orchestrator](java-playwright-e2e/orchestrator/1.2/claude/SKILL.md) | Entry point: AC → passing-test workflow, shared conventions, routing. |
| create-test-scenarios | 1.2 | [java-playwright-e2e/create-test-scenarios](java-playwright-e2e/create-test-scenarios/1.2/claude/SKILL.md) | Jira / acceptance criteria → declarative, tagged Gherkin (Jira MCP, else ask). |
| playwright-page-objects | 1.2 | [java-playwright-e2e/playwright-page-objects](java-playwright-e2e/playwright-page-objects/1.2/claude/SKILL.md) | Discover stable locators (Playwright MCP → codegen → screenshot → HTML → ask); annotation-driven Page Objects. |
| cucumber-step-definitions | 1.2 | [java-playwright-e2e/cucumber-step-definitions](java-playwright-e2e/cucumber-step-definitions/1.2/claude/SKILL.md) | Thin step glue → service layer, picocontainer DI, ScenarioContext, hooks & lifecycle. |
| rest-assured-api-tests | 1.2 | [java-playwright-e2e/rest-assured-api-tests](java-playwright-e2e/rest-assured-api-tests/1.2/claude/SKILL.md) | REST Assured API/hybrid scenarios, three-way validation, `page.route` mocking. |
| database-validation | 1.2 | [java-playwright-e2e/database-validation](java-playwright-e2e/database-validation/1.2/claude/SKILL.md) | Verify persistence via repository/ORM (no raw SQL); audit fields and the DB leg of three-way validation. |
| e2e-framework-setup | 1.2 | [java-playwright-e2e/e2e-framework-setup](java-playwright-e2e/e2e-framework-setup/1.2/claude/SKILL.md) | Scaffold the Maven multi-module framework: deps, runner (JUnit 5/TestNG), parallelism, config/secrets, CI. |
