# Agent Routing — AI-Assisted Development with Multiple Models

## Why Agent Routing Matters

- Human developers using AI tools now have access to multiple models with different strengths
- Routing the right task to the right model saves money and produces better results
- The "PM model" (expensive, good at reasoning) should plan; "worker models" (cheaper, good at execution) should code
- Without routing discipline, expensive model tokens get burned on tasks a cheaper model handles equally well

## The Routing Table

### Code Writing (only these tools)

| Task Type | Route To | Why |
|-----------|----------|-----|
| Business logic / API / UI / simple changes | Copilot CLI (GPT-5.2) | Fast, reliable for standard patterns |
| Mechanical refactoring / batch rename / format | Kimi CLI | Cost-effective for repetitive work |
| Complex integration / E2E tests / multi-module | Sonnet subagent | Best reasoning for complex code |
| Large agentic / sandbox isolation needed | Codex CLI | Sandboxed execution |
| 1-2 line hotfix | Main conversation | Not worth dispatching |

### Non-Code Tasks

| Task Type | Route To |
|-----------|----------|
| Planning, architecture decisions | Main conversation (expensive model) |
| Reading 3+ files / structured summary | Haiku subagent (cheapest) |
| Code Review (diff > 10 lines) | Kimi CLI |
| External research / web queries | Gemini CLI |
| Documentation writing | Haiku subagent |

## The PM Model Role

The expensive model (e.g., Claude Opus) acts as a **Project Manager**, not a coder:

1. **Plan** — Break work into discrete tasks
2. **Route** — Assign each task to the right worker model
3. **Dispatch** — Write clear prompts with constraints
4. **Verify** — Review output, run tests, check diffs
5. **Integrate** — Commit, resolve conflicts, advance to next task

If the PM model is writing more than 5 lines of code, something is wrong with the routing.

## Safety Rules When Dispatching Agents

### 1. Pre-dispatch checkpoint (mandatory)

```bash
# ALWAYS commit important files before dispatching
git add openspec/ .claude/ docs/ && git commit -m "wip: pre-dispatch checkpoint"
```

Agents can run `git clean -fd` or `git restore` and **permanently delete** untracked files. This is not theoretical — it happens routinely with unrestricted agents that "clean up" the working directory before starting.

### 2. Restrict working directory

```bash
# Good: agent can only edit src/
copilot --yolo --add-dir src/ -p @prompt.txt
kimi --print -w src/ -p "refactor X module..."

# Bad: agent has access to everything
copilot --yolo -p @prompt.txt
```

Note: Directory restriction flags only limit **active edits**, not bash commands. An agent can still run `git clean` from a restricted directory.

### 3. Add explicit prohibitions in the prompt

Append to every agent prompt:

```
Only modify files in src/. Do NOT touch openspec/, .claude/, docs/.
Only run: git diff, git status. Do NOT run: git clean, git restore, git checkout, git reset.
```

### 4. Post-dispatch verification

```bash
# After agent completes, verify no unexpected changes
git diff --stat

# If unexpected changes exist, restore protected directories
git checkout -- openspec/ .claude/
```

### 5. Maintain a deny-list

Track tools that are known unstable or produce inconsistent results. Never fall back to denied tools regardless of availability.

## Fallback Order

When the primary tool fails, try the next in line:

```
Copilot CLI → Kimi CLI → Codex CLI → Sonnet subagent → Main conversation
```

When switching, explicitly state which tool failed and why the fallback was chosen. This creates an audit trail and helps refine the routing table over time.

## Wave Execution Pattern

Group non-dependent tasks into **Waves** for parallel execution:

```
Wave 1: [Task A - file1.ts] [Task B - file2.ts] [Task C - file3.ts]
         ↓ parallel dispatch ↓
         ← all complete →
         Code Review (Kimi CLI)
         Build + Test
         git add -A && git commit
         ↓
Wave 2: [Task D - file4.ts] [Task E - file1.ts amended]
         ↓ parallel dispatch ↓
         ...
```

### Wave rules

- **Same Wave** = dispatch in parallel, but only if tasks touch **different files**
- Same-file tasks must be **sequential** (even across Waves)
- Each Wave ends with: code review → build → test → commit
- Every Wave checkpoint: `git add -A && git commit -m "wave N complete"`
- Next Wave starts only after previous Wave passes all checks

### Wave completion checklist

- [ ] `npm run build` — 0 errors
- [ ] Code review — no Critical findings
- [ ] Git commit exists for this Wave
- [ ] Task tracking updated (checkboxes marked)

## Anti-Patterns

| Anti-Pattern | Why It's Bad | What To Do Instead |
|-------------|-------------|-------------------|
| Using expensive model to read files | Wastes tokens on work a cheap model handles | Dispatch Haiku subagent for file reading |
| Dispatching without pre-commit checkpoint | Risk of permanent data loss | Always `git add + commit` first |
| Letting agent access project root | Agent may modify config, SDD artifacts, or docs | Restrict to code directories only |
| Skipping code review between Waves | Errors compound across Waves | Review after every Wave |
| Dispatching same-file tasks in parallel | Agents overwrite each other's changes | Same-file = sequential |
| Trusting agent's "done" report without verification | Agents hallucinate completion | Always `git diff --stat` + test |
| Retrying failed tool with same parameters | Definition of insanity | Switch to fallback, diagnose root cause |

## Parallel vs. Sequential Decision Table

| Task Type | Parallel? | Condition |
|-----------|-----------|-----------|
| Code review (multiple lenses) | Yes | Same diff, different review criteria |
| Code writing | Conditional | Only if tasks touch different files |
| Research queries | Conditional | Only if querying different topics |
| Design decisions | No | Consistency requires single agent |
| File searching | No | Single agent is sufficient |

**Rule of thumb**: Tasks are parallelizable only when they have **zero data dependencies** on each other.
