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

## Skills

| Skill | Version | Agent | Description |
|-------|---------|-------|-------------|
| [java-playwright-cucumber-e2e](java-playwright-cucumber-e2e/1.1.0/claude/SKILL.md) | 1.1.0 (latest) | claude | Scaffold and build end-to-end test automation projects using Java + Playwright + Cucumber (BDD). |
| [java-playwright-cucumber-e2e](java-playwright-cucumber-e2e/1.0/claude/SKILL.md) | 1.0 | claude | Scaffold and build end-to-end test automation projects using Java + Playwright + Cucumber (BDD). |
