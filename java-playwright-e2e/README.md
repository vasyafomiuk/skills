# java-playwright-e2e (Claude Code plugin)

End-to-end test automation with **Java + Playwright + Cucumber (BDD) + REST
Assured**, delivered as seven focused, independently-triggerable Claude Code
skills. Start at the **orchestrator**, which owns the end-to-end workflow and the
shared conventions, and routes to the right specialist skill.

## Install

```text
/plugin marketplace add vasyafomiuk/skills
/plugin install java-playwright-e2e@vasyafomiuk-skills
```

## Skills (invoked as `java-playwright-e2e:<skill>`)

| Skill | Focus |
|-------|-------|
| `orchestrator` | Entry point: AC → passing-test workflow, shared conventions, routing. |
| `create-test-scenarios` | Jira / acceptance criteria → declarative, tagged Gherkin (Jira MCP, else ask). |
| `playwright-page-objects` | Discover stable locators (Playwright MCP → codegen → screenshot → HTML → ask); annotation-driven Page Objects. |
| `cucumber-step-definitions` | Thin step glue → service layer, picocontainer DI, ScenarioContext, hooks & lifecycle. |
| `rest-assured-api-tests` | REST Assured API/hybrid scenarios, three-way validation, `page.route` mocking. |
| `database-validation` | Verify persistence via repository/ORM (no raw SQL); audit fields and the DB leg of three-way validation. |
| `e2e-framework-setup` | Scaffold the Maven multi-module framework: deps, runner (JUnit 5/TestNG), parallelism, config/secrets, CI. |

Skills also trigger automatically when your request matches a skill's
`description`. Version 1.2.
