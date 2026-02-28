---
name: bdd
description: Behavior-Driven Development for all UI projects (web, desktop, mobile). Automatically activates during planning for any project with a user interface. Covers scenario authoring, primitive extraction, test environment setup, and performance tracking. Use when planning features, adding UI elements, or modifying user-facing behavior.
---

# BDD Testing Standards

This skill activates automatically when planning or implementing any UI feature — web apps (Playwright), desktop apps (framework test APIs or embedded test hooks), or mobile apps.

BDD tests complement unit/integration tests. They validate user-visible behavior at the UI level.

---

## Three-Phase Workflow

Every UI feature goes through these phases during planning:

### Phase 1: Scenario Authoring

Write strict Gherkin `.feature` files — one per user story.

```gherkin
# tests/bdd/features/user_management.feature
Feature: User Management

  Scenario: Delete a user
    Given a user "test-{uuid}" exists in company "Acme Corp"
    When I navigate to the "Users" page
    And I click "Delete" for user "test-{uuid}"
    And I confirm the deletion dialog
    Then user "test-{uuid}" should not appear in the users list

  Scenario: View user details
    Given I am on the "Users" page
    When I click on user "Bob Fartypants"
    Then the user detail panel shows last name "Fartypants"
    And the user detail panel shows company "Acme Corp"
```

**Rules:**
- Every user action or feature gets its own scenario — be exhaustive
- Use concrete persona names for read-only tests (`Bob Fartypants`, `Acme Corp`)
- Use `test-{uuid}` patterns for any test that creates, modifies, or deletes data
- One `.feature` file per user story, stored in `tests/bdd/features/`
- Claude generates all scenarios; they are updated incrementally with each change

### Phase 2: Primitive Extraction

After writing scenarios, extract reusable parameterized steps.

Review all steps and identify generalizations:
- `Given I am on the "{page_name}" page` — navigation primitive
- `When I click "{button_text}"` — click primitive
- `Then "{element}" should not appear in the {container}` — absence assertion

**Where primitives live:**
- If using a BDD framework (Behave, Cucumber, Godog): step definitions in code are the source of truth
- If using a custom runner: document primitives in `tests/bdd/primitives.md`

**The primitive library is the investment.** The better it is, the less work future tests require. When building a primitive that has general applicability beyond this project, document it in the cross-project reference (see Cross-Project Learning below).

### Phase 3: Primitive Fulfillment

Implement each primitive against real UI automation:

| Platform | Tool | Notes |
|----------|------|-------|
| Web | Playwright | Default for all web projects |
| Desktop (Qt/PySide6) | QTest or framework API | Use built-in test APIs |
| Desktop (Gio/Go) | Embedded test hooks | See Test Hook Enforcement |
| Desktop (other) | Embedded test hooks | Programmatic UI driver |
| Mobile (iOS) | XCUITest | Native UI testing |

---

## BDD Framework Selection

Choose the appropriate BDD framework based on the project language:

| Language | Framework | Feature File Location |
|----------|-----------|----------------------|
| Python | Behave | `tests/bdd/features/` |
| Go | Godog | `tests/bdd/features/` |
| JavaScript/TypeScript | Cucumber.js | `tests/bdd/features/` |
| .NET/C# | SpecFlow or custom | `tests/bdd/features/` |

If no established BDD framework fits, write a lightweight step runner. Keep it minimal — parse `.feature` files, match step patterns, execute bound functions.

---

## Test Data & Personas

### Seed Data

Define a known dataset in `tests/bdd/seed/` — JSON, SQL, or code that populates the test environment:

```
tests/bdd/seed/
  personas.json      # Known users with stable attributes
  companies.json     # Known organizations
  seed.py            # (or seed.go, etc.) — loads seed data into test DB
```

**Example personas.json:**
```json
{
  "personas": [
    {
      "id": "bob",
      "first_name": "Bob",
      "last_name": "Fartypants",
      "email": "bob@acme.test",
      "company": "Acme Corp"
    },
    {
      "id": "alice",
      "first_name": "Alice",
      "last_name": "Wonderland",
      "email": "alice@globex.test",
      "company": "Globex Corporation"
    }
  ],
  "companies": [
    {"id": "acme", "name": "Acme Corp"},
    {"id": "globex", "name": "Globex Corporation"}
  ]
}
```

**Rules:**
- Seed data is loaded once per test run, not per test
- Seed data is read-only — no test may modify or delete seed entities
- Personas serve double duty: testing AND demos

### Parallel-Safe Data Rules

All tests run in parallel in a shared environment. Follow these rules strictly:

1. **Read-only tests** use seed data directly. `Given I am on the "Users" page` and asserting Bob's last name is safe — no mutation.

2. **Tests that create data** generate unique entities: `test-{uuid}` naming. May reference seed data for read-only context (e.g., assign new user to seed company "Acme Corp"). Must clean up created entities after the test.

3. **Tests that modify data** (rename, update) must create their own entity first, modify it, then clean up. Never modify seed data.

4. **Tests that delete data** must create the entity they will delete. Never delete seed data.

**Decision rule:** If your test could cause another concurrent test to see unexpected state, your test must own (create+destroy) the affected data.

---

## Test Environment

### Implicit & Idempotent Setup

The test environment is never a separate command. Running tests automatically:
1. Checks if the test environment exists under `output/bdd/`
2. Creates it if missing (DB, config, server build)
3. Seeds it with persona data if the seed is stale or missing
4. Runs tests
5. Leaves the environment for the next run (idempotent)

```
output/bdd/
  db/              # Test database (SQLite file, or connection config)
  server/          # Test build of the application
  config/          # Test-specific configuration
  perf.json        # Performance tracking (committed — see below)
```

**Critical rules:**
- Test environment is ALWAYS separate from production/development: different database, different port, different config
- `output/bdd/` is gitignored EXCEPT `perf.json` which is committed
- The test runner script handles all setup transparently
- Setup is idempotent — safe to run repeatedly, only does work if needed

### Environment Initialization Pattern

```python
# Pseudocode — adapt to project language
def ensure_bdd_environment():
    db_path = OUTPUT_DIR / "bdd" / "db" / "test.db"
    if not db_path.exists():
        create_database(db_path)
    if seed_is_stale(db_path):
        load_seed_data(db_path)
    if not test_server_running():
        start_test_server(db_path)
```

---

## Test Hook Enforcement

For native GUI apps without built-in test frameworks, every UI component MUST have a programmatic test accessor.

**Rule: If you add a UI element, you add a test hook. No exceptions.**

A commit that adds a button without a corresponding test accessor function is a failed task. Claude must refuse to commit.

### What a test hook looks like

```python
# PySide6 example
class UserListWidget(QWidget):
    # Production UI code...

    # Test hooks — programmatic access for BDD primitives
    def test_get_visible_users(self) -> list[str]:
        return [item.text() for item in self.user_list.items()]

    def test_click_delete_for_user(self, username: str) -> None:
        item = self.user_list.find_item(username)
        self.delete_button_for(item).click()
```

```go
// Gio example — exported methods for test driving
func (p *UserListPage) TestGetVisibleUsers() []string { ... }
func (p *UserListPage) TestClickDeleteForUser(name string) { ... }
```

**Naming convention:** Test hooks are prefixed with `test_` (Python) or `Test` (Go) so they are clearly identifiable and greppable.

---

## Performance Tracking

### perf.json

Track test durations in `tests/bdd/perf.json` (committed to repo):

```json
{
  "last_run": "2026-03-01T10:30:00Z",
  "total_duration_ms": 3200,
  "scenarios": {
    "user_management.feature::Delete a user": {
      "duration_ms": 450,
      "history": [420, 430, 450]
    },
    "user_management.feature::View user details": {
      "duration_ms": 180,
      "history": [170, 175, 180]
    }
  }
}
```

**Rules:**
- Updated automatically after each test run
- `history` keeps last 10 runs for trend analysis
- Claude reviews `perf.json` during `/improve` and flags regressions
- The goal is maximum speed — UI-driving tests should still run in seconds
- Performance regressions are not automatic failures — Claude analyzes whether the regression is real (consistent trend) or noise (machine load). Use the history array for this judgment.
- BDD tests also validate that the application itself is fast — if a page action takes perceptibly long during a test, that is a real bug

### Speed Optimization Techniques

- **Browser caching:** For web projects, set `Cache-Control: public, max-age=31536000, immutable` on static assets with content-hash URLs (per `/website` skill). Tests benefit from cached assets across scenarios.
- **Reuse browser context:** Don't launch a new browser per test. Share a browser instance, use separate pages or contexts.
- **Parallel execution:** Run independent scenarios concurrently (the data isolation rules make this safe).
- **Minimize navigation:** If two scenarios test the same page, batch them.
- **Test server startup:** Start once, reuse across all tests. Kill only on full suite completion.

---

## Pre-Commit Gate

BDD tests are mandatory before every commit. They integrate with `/commit`:

1. All `.feature` files must have fulfilled primitives (no unbound steps)
2. All scenarios must pass
3. `perf.json` must be updated
4. Any new UI element must have test hooks (enforced)

If BDD tests fail, the commit is blocked. Fix the code, not the tests.

---

## File Structure Summary

```
tests/bdd/
  features/                  # Gherkin .feature files (one per user story)
    user_management.feature
    checkout.feature
  steps/                     # Step definitions (primitives in code)
    navigation_steps.py      # Given I am on the "{page}" page
    interaction_steps.py     # When I click "{button}"
    assertion_steps.py       # Then "{element}" should be visible
  seed/
    personas.json            # Known test data
    seed.py                  # Seed loader
  primitives.md              # (Only if custom runner) Primitive catalog
  perf.json                  # Performance tracking (COMMITTED)
  conftest.py                # (Python) Shared fixtures, env setup
output/bdd/                  # Gitignored runtime environment
  db/
  server/
  config/
```

---

## Cross-Project Learning

When you build a primitive that would be useful across projects, document it in language-specific reference files within this skill directory:

```
~/.claude/skills/bdd/
  SKILL.md                   # This file
  primitives_web.md          # Reusable Playwright/web primitives
  primitives_python_gui.md   # PySide6/Qt test hook patterns
  primitives_go_gui.md       # Gio test hook patterns
```

These files grow organically. After completing BDD work on any project:
- Check if any new primitive is general enough to be reusable
- If yes, add it to the appropriate file with a concrete example
- Next time Claude works on a similar project, it reads these files first and reuses established patterns

This follows the `/improve` philosophy — gradual, experience-driven improvement rather than speculative up-front design.

---

## Workflow Integration

### During Planning

When planning any UI feature, automatically:
1. Identify all user actions and behaviors
2. Write `.feature` files with complete scenarios
3. Review existing primitives — reuse where possible
4. Identify new primitives needed
5. Plan primitive fulfillment (which tool/framework)

### During Implementation

When implementing UI changes:
1. Add test hooks to every new UI element
2. Implement any new primitives needed
3. Run BDD tests — they must pass
4. Update `perf.json`
5. Commit only when green

### During /improve

Review `perf.json` for:
- Scenarios getting slower (consistent trend, not noise)
- Opportunities to parallelize
- Primitives that could be generalized
- New cross-project patterns to document
