# skills

A [Claude Code](https://code.claude.com/docs/en/skills) **plugin marketplace** of
reusable skills.

## Install

Add this repo as a marketplace, then install a plugin:

```text
/plugin marketplace add vasyafomiuk/skills
/plugin install java-playwright-e2e@vasyafomiuk-skills
```

Installed skills are namespaced by plugin (e.g. `java-playwright-e2e:orchestrator`),
so they never collide with other skills, and also trigger automatically when your
request matches a skill's `description`.

## Plugins

### java-playwright-e2e (v1.2)

End-to-end test automation with Java + Playwright + Cucumber (BDD) + REST Assured,
as an orchestrator plus six focused sub-skills. See
[java-playwright-e2e/README.md](java-playwright-e2e/README.md).

| Skill (`java-playwright-e2e:…`) | Path |
|------|------|
| orchestrator | [skills/orchestrator](java-playwright-e2e/skills/orchestrator/SKILL.md) |
| create-test-scenarios | [skills/create-test-scenarios](java-playwright-e2e/skills/create-test-scenarios/SKILL.md) |
| playwright-page-objects | [skills/playwright-page-objects](java-playwright-e2e/skills/playwright-page-objects/SKILL.md) |
| cucumber-step-definitions | [skills/cucumber-step-definitions](java-playwright-e2e/skills/cucumber-step-definitions/SKILL.md) |
| rest-assured-api-tests | [skills/rest-assured-api-tests](java-playwright-e2e/skills/rest-assured-api-tests/SKILL.md) |
| database-validation | [skills/database-validation](java-playwright-e2e/skills/database-validation/SKILL.md) |
| e2e-framework-setup | [skills/e2e-framework-setup](java-playwright-e2e/skills/e2e-framework-setup/SKILL.md) |

## Reference (not a plugin)

`java-playwright-cucumber-e2e/` holds the original **monolith** — one
comprehensive all-in-one SKILL.md, kept as a frozen deep reference. It uses a
versioned-archive layout (`<version>/claude/SKILL.md`) and is **not** installed by
the plugin above.

| Version | Path |
|---------|------|
| 1.1.0 | [java-playwright-cucumber-e2e/1.1.0](java-playwright-cucumber-e2e/1.1.0/claude/SKILL.md) |
| 1.0 | [java-playwright-cucumber-e2e/1.0](java-playwright-cucumber-e2e/1.0/claude/SKILL.md) |

## Layout

```
.claude-plugin/marketplace.json     # marketplace: lists the plugins below
<plugin>/
├── .claude-plugin/plugin.json      # plugin manifest
└── skills/
    └── <skill-name>/SKILL.md       # each skill (folder name = skill name)
```

A skill's identity comes from the `name:` field in its SKILL.md frontmatter.
