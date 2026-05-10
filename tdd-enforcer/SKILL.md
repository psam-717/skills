---
name: tdd-enforcer
description: >
  Enforces a test-driven development workflow: writes failing tests first, waits
  for user confirmation, then writes the minimal implementation to pass them,
  then refactors. Use this skill when the user types /tdd, says "write tests
  first", "TDD this", "test-driven", or "write tests for X before implementing
  it". Also trigger when the user asks to implement a non-trivial function,
  service, or feature and hasn't mentioned tests yet — proactively suggest the
  TDD cycle. Skips the ceremony for clearly trivial tasks (one-liners, pure
  config, simple data classes with no logic). Adapts to any language and test
  framework — always reads the project first.
---

# TDD Enforcer

Drives a test-first development cycle. Writes failing tests, waits for
confirmation, then writes the minimal passing implementation, then refactors.

---

## Step 0 — Trivial Task Check

Before starting the TDD cycle, ask: *is this task trivial?*

**Skip TDD ceremony if the task is clearly:**
- A one-liner utility (e.g. format a date string, clamp a number)
- Pure configuration or constants
- A simple data class / struct with no logic
- A thin wrapper with no branching

For trivial tasks: write the implementation directly with a brief note —
*"This is straightforward enough to skip TDD — here's the implementation."*

For everything else: proceed with the full cycle below.

---

## Step 1 — Discover the Project's Test Stack

Before writing a single test, read the project to understand:

| Question | Where to look |
|---|---|
| Language & framework | `pyproject.toml`, `package.json`, `go.mod`, `Cargo.toml`, file extensions |
| Test framework | `pytest`, `jest`, `vitest`, `go test`, `unittest`, `rspec`, etc. |
| Test file location | `tests/`, `__tests__/`, co-located `*.test.ts`, `*_test.go` |
| Test naming convention | `test_*`, `*.spec.ts`, `Test*`, etc. |
| Mocking library | `unittest.mock`, `jest.mock`, `mockery`, etc. |
| Async pattern | `pytest-anyio`, `asyncio`, `async/await` in tests |
| Existing test examples | Read 1-2 existing test files to match style exactly |

If no tests exist yet: ask the user which framework they're using before proceeding.

---

## Step 2 — Analyse the Task

Before writing tests, map out what needs to be tested:

1. **What is the unit under test?** (function, class, endpoint, module)
2. **What are the inputs and outputs?**
3. **What are the happy paths?** (normal, expected usage)
4. **What are the edge cases?** (empty, zero, None/null, boundary values, max/min)
5. **What are the failure cases?** (invalid input, missing dependency, permission denied)
6. **What needs to be mocked?** (DB, external APIs, filesystem, time, randomness)

Write this analysis as a brief comment block at the top of the test file.

---

## Step 3 — Write the Failing Tests

Write the complete test suite **before any implementation exists**.

### Rules for the test block:
- Tests must fail if run now (the implementation doesn't exist yet) — state this explicitly
- Cover: happy path(es), edge cases, and at least 2 failure/error cases
- Each test has one clear assertion focus — no multi-concern tests
- Test names must read as documentation: `test_create_order_with_no_items__raises_validation_error`
- Mock all external dependencies — tests must not hit real DBs, APIs, or filesystems
- Include any required fixtures or setup in the same block
- Add a `# FAILING — implementation not yet written` comment at the top

### Output format for Step 3:

```
## 🔴 Tests (will fail — implementation not written yet)

[test code block]

---
**Test plan:**
- ✅ Happy path: [what it tests]
- ✅ Edge case: [what it tests]
- ✅ Edge case: [what it tests]
- ✅ Failure case: [what it tests]
- ✅ Failure case: [what it tests]

**What's mocked:** [list dependencies being mocked and why]

---
▶ Run these tests to confirm they fail, then reply "go" to get the implementation.
```

**Stop here. Do not write any implementation.** Wait for the user to confirm.

---

## Step 4 — Write the Minimal Implementation

Only proceed after the user confirms (e.g. "go", "looks good", "implement it", ✅).

Write the **minimum code needed to make the tests pass**. Nothing more.

### Rules for the implementation block:
- No gold-plating — implement only what the tests require
- No unused parameters, no speculative features
- If a test is wrong or unreasonable, say so before changing either the test or the implementation
- Keep the same code style as the rest of the project (read existing files if needed)

### Output format for Step 4:

```
## 🟢 Implementation (minimal — makes the tests pass)

[implementation code block]

---
**What this does:**
- [brief bullet explaining each meaningful decision]

**What it deliberately doesn't do:**
- [anything intentionally left out / deferred]
```

---

## Step 5 — Refactor Pass

After the implementation, always offer a refactor review:

```
## 🔵 Refactor suggestions (optional — tests still pass after each change)

- [Suggestion 1 — e.g. extract this logic into a helper]
- [Suggestion 2 — e.g. this variable name is ambiguous]
- [Suggestion 3 — e.g. this condition can be simplified]

Apply any of these? Or ship as-is?
```

Only suggest refactors that:
- Don't change behaviour
- Improve readability, reduce duplication, or improve naming
- Can each be applied independently without breaking the others

---

## Anti-Patterns to Actively Avoid

Call these out explicitly if spotted in existing code or if the user tries to shortcut:

| Anti-pattern | Why it's a problem |
|---|---|
| Tests written after implementation | Tests tend to match what the code does, not what it should do — they rubber-stamp bugs |
| Tests with no assertions | Not a test — it's a smoke check at best |
| One giant test covering everything | Failures are hard to diagnose; individual cases are invisible |
| Mocking the thing being tested | Tests the mock, not the code |
| Testing implementation details (private methods, internal state) | Brittle — breaks on refactors that don't change behaviour |
| No edge cases | Most bugs live at the boundaries |
| Hardcoded IDs or data that conflicts across tests | Tests become order-dependent and flaky |
| Testing framework code (e.g. FastAPI routing itself) | Test your logic, not the library |

If the user asks to skip tests entirely for a non-trivial task, push back once:
*"This looks non-trivial — skipping tests here is likely to cost more time than it saves.
Want to at least write the happy path and one failure case?"*
If they still decline, respect it and proceed.

---

## Language-Specific Quick Reference

### Python / pytest
```python
# Async tests
import pytest
pytestmark = pytest.mark.anyio

# Mocking
from unittest.mock import AsyncMock, patch, MagicMock

# Fixtures
@pytest.fixture
def mock_repo():
    return AsyncMock()

# Naming
def test_{unit}_{condition}__{expected_outcome}():
    ...
```

### TypeScript / Jest or Vitest
```typescript
// Mocking
jest.mock('../path/to/module')
vi.mock('../path/to/module')  // vitest

// Async
it('should ...', async () => { ... })

// Naming
describe('UnitName', () => {
  it('does X when Y', () => { ... })
})
```

### Go
```go
func TestUnitName_Condition_ExpectedOutcome(t *testing.T) {
    // table-driven tests preferred
    tests := []struct{ ... }{ ... }
    for _, tt := range tests {
        t.Run(tt.name, func(t *testing.T) { ... })
    }
}
```

---

## Full Cycle Summary

```
/tdd <task description>
        │
        ▼
  Trivial? ──Yes──▶ Write implementation directly
        │
        No
        ▼
  Discover test stack
        │
        ▼
  Analyse: inputs, outputs, edge cases, mocks
        │
        ▼
🔴 Write failing tests ── STOP ── wait for user
        │
      "go"
        ▼
🟢 Write minimal implementation
        │
        ▼
🔵 Offer refactor suggestions
        │
        ▼
      Done
```