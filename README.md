![](banner.jpg)

# BDD Skill for Claude Code

**Stop shipping UI features you can't prove work.**

Every UI project eventually hits the same wall: you build features, manually click through them, ship, and hope. When something breaks, you find out from users — not from your test suite. BDD tests are the fix, but writing and maintaining them is tedious enough that most teams skip it.

This skill makes Claude Code do the tedious part.

## What It Does

When you're planning or building any UI feature — web app, desktop app, mobile — this skill automatically activates and drives a three-phase process:

### 1. Scenario Authoring
Claude examines every user action in your feature and writes exhaustive Gherkin scenarios:

```gherkin
Scenario: Delete a user
  Given a user "test-{uuid}" exists in company "Acme Corp"
  When I navigate to the "Users" page
  And I click "Delete" for user "test-{uuid}"
  And I confirm the deletion dialog
  Then user "test-{uuid}" should not appear in the users list
```

No hand-waving. Every button, every flow, every edge case gets a scenario.

### 2. Primitive Extraction
Claude reviews all the steps and pulls out reusable, parameterized primitives — `Given I am on the "{page}" page`, `When I click "{button}"`, etc. These accumulate into a library. The better the library, the less work future features require.

### 3. Primitive Fulfillment
Claude implements each primitive against real UI automation — Playwright for web, framework test APIs for desktop, or embedded test hooks for native apps that lack test tooling.

## Why This Matters

**Tests that prove the UI works, not just the logic.** Unit tests verify functions. BDD tests verify that a user can actually do the thing you built. Both matter.

**Parallel-safe by design.** Tests run concurrently in a shared environment without stepping on each other. Read-only tests use known seed data (personas). Tests that mutate data create and own their entities. No flaky failures from test interference.

**Performance as a feature.** Every test run tracks scenario durations in a committed `perf.json`. Claude monitors trends and flags real regressions (not noise). In 2026, users shouldn't wait on page actions — the tests prove they don't have to.

**The primitive library compounds.** The first feature is the most work. By the tenth feature, most steps already have working primitives. New scenarios practically write themselves.

**Test hooks are mandatory.** For native apps without built-in test frameworks, every new UI element must have a programmatic accessor. Add a button without a test hook? The commit is blocked. This ensures the app stays testable as it grows.

## Test Data & Personas

Instead of fragile test fixtures, you define personas — known users, companies, datasets that seed the test environment once per run:

```json
{
  "personas": [
    {"id": "bob", "first_name": "Bob", "last_name": "Fartypants", "company": "Acme Corp"},
    {"id": "alice", "first_name": "Alice", "last_name": "Wonderland", "company": "Globex Corporation"}
  ]
}
```

Ten tests can read Bob simultaneously. None of them modify him. A test that deletes a user creates `test-{uuid}` and deletes that instead. Personas also double as demo data — your test environment is always a working demo.

## Environment

The test environment is implicit. Running tests automatically creates it under `output/bdd/` — separate database, separate server, separate config. No manual setup commands. No shared state with your real environment. Idempotent — safe to run repeatedly.

## Integration

- **Pre-commit gate:** All BDD tests must pass before every commit
- **Auto-activates:** No need to remember `/bdd` — it triggers during planning for any UI project
- **Cross-project learning:** Primitives that generalize get documented in language-specific reference files that improve over time

## Supported Platforms

| Platform | Automation |
|----------|-----------|
| Web | Playwright |
| Desktop (Qt/PySide6) | QTest / framework APIs |
| Desktop (Gio/Go) | Embedded test hooks |
| iOS | XCUITest |

## Installation

Copy `SKILL.md` to `~/.claude/skills/bdd/` and add to your Claude Code skill table. The skill activates automatically — no explicit invocation needed.

## License

This project is licensed under [CC BY-NC 4.0](https://darren-static.waft.dev/license) - free to use and modify, but no commercial use without permission.
