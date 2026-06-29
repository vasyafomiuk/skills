# Changelog

All notable changes to the **java-playwright-e2e** plugin are documented here.
Format based on [Keep a Changelog](https://keepachangelog.com); this project
follows [Semantic Versioning](https://semver.org).

## [1.2.0] - 2026-06-29

Decomposed the monolithic skill into a focused, installable plugin.

### Added
- Six focused, independently-triggerable skills plus an `orchestrator` entry
  point: `create-test-scenarios`, `playwright-page-objects`,
  `cucumber-step-definitions`, `rest-assured-api-tests`, `database-validation`,
  `e2e-framework-setup`.
- Packaged as a Claude Code plugin in a marketplace — install with
  `/plugin marketplace add vasyafomiuk/skills` then
  `/plugin install java-playwright-e2e@vasyafomiuk-skills`.
- A worked end-to-end `EXAMPLE.md`, plus Quickstart and Troubleshooting in the
  orchestrator.
- New how-to sections: accessibility (axe-core), trace-viewer debugging,
  retries & flaky-test triage, and `storageState` auth reuse.

### Changed
- Orchestrator skill renamed to `orchestrator` (invoked as
  `java-playwright-e2e:orchestrator`).
- Canonical, mutually consistent `ScenarioContext` and static `Config` across all
  skills so the snippets compile when assembled.
- Pinned concrete, resolvable dependency versions (replacing placeholder tokens);
  added HikariCP + JDBC driver.
- Tightened skill descriptions to remove auto-trigger collisions.
- Relaxed the one-Scenario-per-AC rule — an AC may span multiple scenarios when it
  has distinct flows or roles (each still tagged with its AC id).
- `create-test-scenarios` (and the page-object/step skills) now ask for existing
  feature files / examples first and follow the project's house style.

## [1.1.0]

Monolith reference (`java-playwright-cucumber-e2e/1.1.0`): added web-first
assertions, test isolation, `page.route` mocking, CI guidance, and the JUnit 5
platform-suite runner as the default.

## [1.0.0]

Initial monolithic skill (`java-playwright-cucumber-e2e/1.0`).
