# SDD Workflow (Spec-Driven Development)

## Purpose

Provide a step-by-step execution manual for Spec-Driven Development. SDD ensures every line of code traces back to a validated specification, and every specification can be falsified through tests.

## When to Use

Any feature development, bug fix, or refactor. No exceptions.

---

## Core Principle

**SDD is Popper's falsificationism, not waterfall construction.**

A spec is a hypothesis that can be disproven -- not a blueprint that must be followed rigidly.

- Spec = hypothesis ("this feature should work this way")
- Red tests = falsification experiments ("if the hypothesis is correct, this test should pass")
- Green tests = hypothesis temporarily holds
- Requirement change = new evidence disproves old hypothesis -> update spec, not code

**Never** "build first, then fix the spec to match." That is the waterfall anti-pattern.

---

## Complete Workflow

```
discuss (optional)
    |
propose (required)
    |
analyze (required -- 0 findings before continuing)
    |
[TDD] failure matrix + red tests (required before apply)
    |
apply (dispatch implementation work)
    |
ingest (when requirements change mid-work)
    |
archive (when complete)
```

---

## Step-by-Step Execution

### Step 1: Discuss (Optional)

**When to use**: Requirements are unclear, multiple directions exist, or a major architectural decision is needed.

**Output**: Consensus, captured in discussion notes or reflected in the proposal.

**Skip condition**: Requirements are already clear enough to write a spec directly.

---

### Step 2: Propose (Required)

Produce three artifacts:

#### spec.md -- Functional Specification

```markdown
# Feature Name

## Problem Statement
What problem does this change solve?

## Goals
What does success look like?

## Scope
What is included (list explicitly)

## Non-Goals
What is excluded (equally important)

## Acceptance Criteria
Verifiable conditions -- each one maps to a red test
```

#### design.md -- Design Decisions

```markdown
# Design Decisions

## D1 -- Decision Name
Choice: [selected approach]
Rationale: [why this was chosen]
Rejected: [Alternative B] (reason: [why not])
Rejected: [Alternative C] (reason: [why not])

## D2 -- Next Decision
...
```

**Why record rejected alternatives**: Six months later, if the reason for rejecting Alternative B no longer applies, you can re-evaluate with full context. Without rejection reasons, you only know "we chose A" with no basis for reassessment.

#### tasks.md -- Implementation Task List

```markdown
## Wave 1 (parallelizable)
- [ ] Task 1.1 [Tool: Copilot] -- description
- [ ] Task 1.2 [Tool: Kimi] -- description

## Wave 2 (depends on Wave 1)
- [ ] Task 2.1 [Tool: Sonnet] -- description
```

#### Self-Review After Propose

- [ ] Every design decision in design.md has a corresponding task in tasks.md
- [ ] Every acceptance criterion in spec.md has a corresponding test task
- [ ] Every rejected alternative has an explicit reason

---

### Step 3: Analyze (Required)

Run consistency analysis across all three artifacts.

**Four dimensions checked**:

| Dimension | Question |
|-----------|----------|
| Coverage | Does every spec requirement have a corresponding task? |
| Consistency | Do design.md decisions align with tasks.md implementation? |
| Ambiguity | Are there vague or unclear descriptions? |
| Gaps | Are there missing edge cases? |

**Standard**: 0 findings (all levels -- Critical, Warning, Suggestion) before proceeding.

If analysis finds issues, go back and fix spec/design/tasks, then re-analyze.

---

### Step 4: TDD Failure Matrix (Required Before Implementation)

Before writing any implementation code, produce a failure matrix.

**Format**:

| Failure Point | Red Test Name | Expected Error |
|---------------|---------------|----------------|
| Unknown IPC command | `mock-unknown-cmd-throws` | `Error: Mock not implemented: unknown_cmd` |
| Reset restores defaults | `store-reset-restores-defaults` | -- |
| Production env without native backend | `bridge-prod-throws-error` | `Error: NotInNativeEnvError` |

**Steps**:

1. Read spec.md acceptance criteria
2. Read design.md risks and trade-offs
3. List every scenario that could fail
4. For each failure scenario, define a test name and expected result
5. Get confirmation that the failure matrix is complete
6. Write red tests only (no implementation code)
7. Run tests -- confirm all red (failing)
8. Confirm red test list, then proceed to implementation

---

### Step 5: Apply (Implementation)

**Before starting**:

```bash
git status          # confirm clean working tree
git add openspec/ .claude/ && git commit -m "wip: pre-dispatch checkpoint"
```

**Wave execution loop**:

1. Read tasks.md, list current wave's tasks and tool assignments
2. Confirm assignment table
3. Dispatch tasks in parallel (different files can run in parallel)
4. Wait for all tasks to complete
5. Code review (for diffs > 10 lines)
6. Run tests
7. Git commit (conventional commits format)
8. Update tasks.md (mark completed `[x]`)
9. Next wave

**Wave completion criteria (all must pass)**:

- [ ] Build: 0 errors
- [ ] Code review: no Critical findings
- [ ] Git commit exists
- [ ] tasks.md `[x]` updated

---

### Step 6: Ingest (When Requirements Change)

**When to use**: During apply, you discover the requirements are wrong, or direction changes.

**Do not** deviate from the spec in code. Update the spec first, then continue.

1. Update spec/design/tasks
2. Re-run analysis (0 findings)
3. Continue apply

---

### Step 7: Archive (When Complete)

**When to use**: All tasks marked `[x]`, all tests pass, feature merged to main branch.

Archive the change. It moves from active to archived status.

---

## Traditional Development vs SDD

| Aspect | Traditional | SDD |
|--------|-------------|-----|
| Starting point | Write code directly | Write spec first |
| Documentation | Added after (usually skipped) | Documentation is part of design |
| Requirement change | Code changed, no record of why | Spec updated first, code follows, full context preserved |
| "Why was this done?" | Git blame, guess intent | Read spec.md, explicit decision rationale |
| Code review | Read code, guess intent | Compare code against spec for drift |
| Refactoring | Afraid to break things | Spec + tests as safety net, refactor with confidence |

---

## Why "Spec First" Makes Development Faster

On the surface: writing specs takes time, it is extra work.

In practice: **specs pay the cost of thinking at the cheapest possible moment.**

| When problem is discovered | Cost to fix |
|---------------------------|-------------|
| Spec phase (no code written yet) | Change a few lines of text |
| Mid-development | Change code + tests + possible refactor |
| After code review | Redo the feature |
| After release | Emergency fix + users already impacted |

---

## Common SDD Mistakes

**Mistake 1: Treating specs as immutable**

Specs are hypotheses. Hypotheses should be updated when new evidence appears.

"The spec is finalized, it cannot change" -> This treats SDD as waterfall.

**Mistake 2: Building first, writing spec after**

"Build it first, document later" -> Documentation will always lag behind code. Stale specs are worse than no specs because they mislead.

**Mistake 3: Over-specifying**

Specs are not code comments. They should not describe "what line 42 does."

Spec granularity: feature-level "what to do" and "what not to do", acceptance criteria, key design decisions. Implementation details belong in code.
