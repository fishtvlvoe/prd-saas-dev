# TDD Pattern: Red-Green-Refactor

## Purpose

Define the Test-Driven Development pattern with failure matrix planning. TDD ensures every implementation has an objective "done" definition, and every test validates requirements -- not implementation details.

## When to Use

Before writing any implementation code. After the spec and design are finalized (SDD Step 4).

---

## Core Logic

```
Phase 1: Define "what is correct" (Spec)
    |
Phase 2: Write tests proving "it is currently wrong" (Red)
    |
Phase 3: Write implementation to make tests pass (Green)
    |
Phase 4: Clean up (Refactor -- change structure, not behavior)
```

**Phase 2 and Phase 3 must not be reversed.**

---

## Why Phase 2 Cannot Be Skipped

### Problem 1: Tests unconsciously conform to implementation

When implementation exists first, test authors tend to write tests that "pass for this implementation" rather than "validate the requirement." Such tests only verify "the code does what it does," not "the code does what it should."

### Problem 2: No clear definition of "done"

Without red tests, there is no objective completion criteria. Developers can keep adding features, keep "optimizing," and never know when to stop.

With red tests: tests go from red to green one by one. "Done" means all reds are green. Clear, objective, unambiguous.

### Problem 3: Tests written after implementation are tightly coupled

When code already exists, tests must understand implementation details to test it. Tightly coupled tests break when implementation changes, making test maintenance expensive. Expensive tests get deleted.

---

## Failure Matrix: The TDD Planning Tool

Before writing any test, produce a failure matrix analysis.

### Format

| Failure Point | Red Test Name | Expected Error/Result |
|---------------|---------------|----------------------|
| Unknown command dispatched | `bridge-unknown-cmd-throws` | `Error: Not implemented: unknown_cmd` |
| State reset restores defaults | `store-reset-restores-defaults` | `items.length === 2` |
| Production env without backend | `bridge-prod-no-backend-throws` | `Error: NotInBackendError` |
| License activation changes state | `license-activate-updates-status` | `status === "valid"` |

### Purpose of the Failure Matrix

1. Every failure scenario has an explicit test name and expected output
2. Stakeholders can verify "are these failure scenarios complete" before implementation begins
3. Agents/developers know exactly which tests to write, not "write some tests"

---

## Steps

### Step 1: Produce the Failure Matrix

1. Read spec.md acceptance criteria
2. Read design.md risks and trade-offs
3. For each acceptance criterion, identify what failure looks like
4. For each risk, identify what failure looks like
5. Write the matrix table

### Step 2: Confirm the Failure Matrix

Get confirmation that the failure matrix is complete. Missing failure points here means missing tests later.

### Step 3: Write Red Tests Only

Dispatch test writing with explicit instruction: **write tests only, do not write implementation.**

```typescript
// Example: red test for environment routing
test('production environment without native backend throws error', async () => {
  // Setup: NODE_ENV=production, no native backend
  // Execute: safeInvoke('list_items')
  // Verify: throws NotInBackendError
})

test('development mock environment dispatches to mockInvoke', async () => {
  // Setup: NODE_ENV=development, no native backend
  // Execute: safeInvoke('list_items')
  // Verify: mockInvoke is called, returns mock data
})

test('native environment uses real invoke', async () => {
  // Setup: native backend present (mock isNativeEnv = true)
  // Execute: safeInvoke('list_items')
  // Verify: real invoke is called
})
```

### Step 4: Run Tests, Confirm All Red

```bash
npm test    # or: pnpm test / cargo test
```

All tests should fail. If any test passes without implementation, either:
- The test is not actually testing the new feature
- The feature already exists (re-evaluate the spec)

### Step 5: Write Implementation (Make Red Turn Green)

Now dispatch implementation work. The goal is narrow: make each failing test pass.

### Step 6: Run Tests, Confirm All Green

```bash
npm test
```

All tests should pass. If any test still fails, fix the implementation (not the test).

### Step 7: Refactor

Change structure without changing behavior. Tests stay green throughout.

---

## Three-Layer Test Architecture

```
Layer 1: Unit Tests
  Speed: milliseconds
  Coverage: individual functions, modules, components
  Tools: Vitest / Jest (frontend), cargo test / pytest (backend)

Layer 2: Integration Tests
  Speed: seconds
  Coverage: multi-module interaction, API communication simulation
  Tools: Vitest + mock backend / Supertest

Layer 3: End-to-End Tests
  Speed: minutes (requires full application)
  Coverage: complete user flows
  Tools: Playwright / Cypress / native E2E
```

The failure matrix maps to Layer 1 and Layer 2:

```
Failure Matrix (design phase)
    |
Layer 1 red tests (unit -- single functions)
Layer 2 red tests (integration -- multi-module interaction)
    |
Write implementation (make reds turn green)
    |
Layer 3 (E2E -- final acceptance)
```

---

## Test Quality Indicators

**Do not chase coverage percentage.** 100% coverage with tests written after implementation may all be conforming to the code rather than validating requirements.

Measure these instead:

| Indicator | Question |
|-----------|----------|
| Failure matrix coverage | Does every failure scenario have a corresponding test? |
| Independence | Can each test run independently (no dependency on execution order)? |
| Readability | Does the test name clearly state what it validates? |
| Speed | Do Layer 1 + Layer 2 complete within 30 seconds? |
| Stability | Is the test deterministic (same code, same result 10/10 times)? |

---

## TDD and SDD Relationship

```
SDD (Spec) defines "what is correct"
    |
TDD Phase 2 (Red) converts "correct" into executable verification
    |
TDD Phase 3 (Green) makes implementation satisfy verification
    |
Analyze confirms spec and implementation have not diverged
```

Three tools serving one goal: **give "done" an objective definition.**

- Without spec: do not know what to build
- Without tests: do not know if it is built correctly
- Without analysis: do not know if what was built matches what was specified

---

## Common Misconceptions

**"TDD makes development slower"**

Surface: extra time writing tests. Reality:
- Fewer hours debugging (bugs caught by tests, not in production)
- Fewer regressions (changing A does not silently break B)
- Less fear of refactoring (tests are the safety net)

**"Only complex code needs tests"**

Simple code can break during refactoring too. Tests exist not because code is hard, but because you want it to stay correct in the future.

**"Higher coverage is always better"**

Coverage written after implementation may all be conforming tests. What matters is whether tests validate requirements. The failure matrix is more meaningful than coverage percentage.
