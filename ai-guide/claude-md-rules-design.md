# CLAUDE.md Rule Engineering

## Purpose

Design and maintain AI agent behavior-contract files (CLAUDE.md, AGENTS.md, system prompts, lesson logs) so that every rule fires reliably. Cover both the principles and the executable SOP for slimming an existing rule stack.

## When to Use

- Adding a new rule to an agent rule file
- A rule file has crossed ~200 lines and rules are being ignored
- An agent has repeated the same mistake despite a rule existing for it
- Multiple rule files in a stack and rules appear duplicated across them
- Routine maintenance (every 1-3 months)

---

## Core Principles

### PRINCIPLE-001: Rules Must Be Incident-Driven

**Rule**: A rule belongs in the file if and only if it traces to a specific failed agent action that cost time, broke production, or required user correction.

**Test**: Ask "what incident does this prevent?" If the honest answer is "none, but it sounds smart" — do not add it.

**Anti-pattern**: Copying rules from blog posts or viral templates. Generic principles without a local incident history dilute attention and are usually subsumed by the model's defaults.

---

### PRINCIPLE-002: Rules Must Be Operationally Checkable

**Rule**: A reviewer (human or AI) must be able to grade an agent run pass/fail against the rule.

**Examples**:
- Ungradable: "Write clean code", "be thoughtful"
- Gradable: "Run `npm test` and paste output before claiming done", "Before any rsync deploy, run `chmod 755`/`644` reset"

**Test**: If you cannot write a one-line test that detects violation, the rule is aspirational, not behavioral.

---

### PRINCIPLE-003: Single Source of Truth Per Category

**Rule**: Each rule category lives in exactly one file. Other files reference, never copy.

**Category → SSOT mapping**:

| Rule category | SSOT file |
|---------------|-----------|
| Tool dispatch / routing | `rules/routing.md` (or equivalent) |
| Workflow steps | Workflow-managed block (e.g. Spectra block in CLAUDE.md) |
| Personality, tone, communication norms | `soul.md` (or `personality.md`) |
| Incident corrections | `lessons.md` |
| Project-specific lessons | `lessons-<project>.md` |

**Anti-pattern**: Same tool dispatch table copied into 3 files. They drift out of sync within a month and the agent has no way to know which is authoritative.

---

### PRINCIPLE-004: Honor the System Prompt Baseline

**Rule**: Before adding a rule, check the model's system prompt. If the rule restates a built-in default, do not add it.

**Examples of system-prompt-covered defaults** (Claude Code as of 2026-05):
- Do not re-read a file after Edit/Write
- Be concise, avoid filler phrases
- Prefer dedicated tools over Bash when one fits
- Verify before claiming success
- Match response detail to task complexity

**Test**: Read the system prompt yearly. Cross-out every rule of yours that overlaps.

---

### PRINCIPLE-005: Density Cap on Auto-Loaded Surface

**Rule**: The auto-loaded rule surface (system prompt + included files) should stay under ~200 lines of rule content. Past this threshold, compliance falls measurably (per Anthropic-published guidance).

**Mechanism**: Rules compete for the model's attention budget. Crowded files lose enforceability before they lose technical correctness.

**Solution**: Split by load context. Auto-load only cross-cutting rules. Load domain rules on-demand when the conversation enters that domain.

---

### PRINCIPLE-006: Sacred Zones Are Not Optimization Targets

**Rule**: Identity files, communication-norm files, and curated incident logs are never compressed for token budget reasons.

**Sacred zones**:
- "Who I am" / "Who the user is"
- Tone-of-voice prescriptions (e.g. Caveman mode, formality level)
- Decoded-meaning tables ("when user says X, they mean Y")
- Historical iteration log

**Reason**: Removing a personality rule because "be concise" is in the system prompt loses the specific texture the user wants. Sacred zones are signal, not noise, even when redundant on a token analysis.

---

## Common Redundancy Patterns

Catalog of cuts most rule stacks need. Each entry: **Symptom / Root Cause / Cut / Verify**.

### REDUNDANCY-001: System Prompt Duplicate

**Symptom**: A rule restates a baseline behavior the model already follows.

**Root Cause**: Rule was written before the model's defaults included this behavior, never retired.

**Cut**: Delete the rule from your file. Trust the system prompt.

**Verify**: After cut, run a session and trigger the behavior the rule was about. Agent should still follow correctly.

---

### REDUNDANCY-002: Cross-File SSOT Violation

**Symptom**: Same rule appears in 2+ files, sometimes with subtle wording differences.

**Root Cause**: Rule was copied during a refactor instead of being moved.

**Cut**: Pick the file that semantically owns the category (see PRINCIPLE-003). Replace all other copies with a one-line reference: `<category> → <path>`.

**Verify**: `grep -r '<distinctive phrase>' <rule-dir>` returns one file.

---

### REDUNDANCY-003: Tool Section Inside Personality File

**Symptom**: A 10+ line tool dispatch table or workflow checklist embedded in a personality / identity file.

**Root Cause**: Personality file grew organically. Tool guidance was added there because it felt foundational.

**Cut**: Move the table to the tools SSOT file. Replace with one-line reference.

**Verify**: Personality file contains only identity, tone, communication norms, behavioral non-negotiables. No tool-specific tables.

---

### REDUNDANCY-004: Stale Tool / Path Reference

**Symptom**: Rule mentions a tool, path, or product that no longer exists or has been deprecated.

**Root Cause**: Rules accumulate; deletion does not happen automatically.

**Cut**: Update if the rule is still relevant under the new tool. Delete if the underlying need no longer exists.

**Verify**: `grep -r '<tool>\|<path>\|<product>' <rule-dir>` returns no surviving rules.

**Examples**: A rule about Cursor when Cursor has been removed from the workflow. A path `memory/lessons.md` when the actual path is `~/.claude/lessons.md`.

---

### REDUNDANCY-005: Aspirational Rule (No Incident)

**Symptom**: Rule sounds reasonable but no specific failure motivated it.

**Root Cause**: Rule was added from a blog post or general best-practices list, not from a real correction.

**Cut**: Delete. If the rule turns out to matter, the next incident will surface it sharper.

**Verify**: Each surviving rule has a date or incident reference attached.

---

### REDUNDANCY-006: Subsumed Rule

**Symptom**: Rule A is a specific case of rule B; both exist.

**Root Cause**: Both were added at different times; nobody noticed the overlap.

**Cut**: Keep the more general rule (B). Delete the specific rule (A) unless A has unique trigger conditions worth preserving.

**Verify**: Read the file top-to-bottom. No rule paraphrases another.

---

## SOP: Slim an Existing Rule Stack

Run this every 1-3 months or whenever auto-loaded rule files cross ~200 lines.

### Step 1: Inventory

```bash
wc -lwc <rule-dir>/*.md <rule-dir>/**/*.md
```

Note: total lines, total chars. Compute approximate tokens (CJK ≈ chars/3, ASCII ≈ chars/4).

### Step 2: System Prompt Cross-Check

Read the model's current system prompt. For each rule in your files, ask: "Does the system prompt already cover this?"

Apply REDUNDANCY-001 cuts.

### Step 3: Build SSOT Map

List every rule category in your stack. Assign each to exactly one file (per PRINCIPLE-003).

For each cross-file duplicate found:
- Pick the SSOT file
- Apply REDUNDANCY-002 / REDUNDANCY-003 cuts

### Step 4: Stale Reference Scan

```bash
# List every tool, path, product mentioned
grep -rEoh '[a-zA-Z][a-zA-Z0-9-]+(\.[a-zA-Z]+)+|\b[A-Z][a-z]+(CLI|API)\b' <rule-dir> | sort -u
```

For each: verify the tool / path / product still exists and is in active use. Apply REDUNDANCY-004 cuts.

### Step 5: Aspiration Audit

For each rule, ask: "What specific incident does this prevent?"

If no concrete incident comes to mind, mark for deletion. Apply REDUNDANCY-005 cuts.

### Step 6: Subsumption Pass

Read the file top-to-bottom. For each adjacent pair of rules, check if one implies the other. Apply REDUNDANCY-006 cuts.

### Step 7: Mark Sacred Zones

Before any cut, identify sacred zones (per PRINCIPLE-006). These are excluded from the optimization regardless of token budget.

### Step 8: Split If Still Over Threshold

If auto-loaded surface still exceeds ~200 lines after Steps 2-7:

- Categorize remaining rules by load context (domain-specific vs cross-cutting)
- Move domain rules to `lessons-<domain>.md`
- Auto-load only cross-cutting rules; document on-demand load instructions at the top of the auto-loaded file

### Step 9: Verify

- `wc -lwc` again, compare against Step 1 numbers
- Run a session, exercise rules that were touched, confirm behavior unchanged
- Commit with a message that lists what was cut and why

---

## Anti-Patterns

### ANTI-001: Wish-List CLAUDE.md

**Description**: Every rule sounds reasonable. Rules are phrased as virtues ("be thoughtful", "write clean code"). The author cannot recall enforcing any specific rule. The file grows monotonically; nothing is ever deleted.

**Correction**: Apply Step 5 (Aspiration Audit). Replace wish-list rules with incident-driven entries.

---

### ANTI-002: Marketing Rule Templates

**Description**: Importing a viral rule template (e.g. "12 rules that cut Claude's mistakes to 3%") wholesale.

**Reason it fails**:
- Single-author, no third-party replication
- "Mistake" is undefined operationally
- More rules ≠ higher compliance (often the inverse, per PRINCIPLE-005)
- Generic rules duplicate model defaults (REDUNDANCY-001)

**Correction**: Absorb the principles that overlap with your actual incident history. Discard the rest.

---

### ANTI-003: Personality / Tool Mixing

**Description**: Tool dispatch tables, workflow steps, or procedural checklists embedded inside the identity / personality file.

**Reason it fails**: Updating the routing logic now requires editing the personality file, and the personality file accumulates non-personality content that crowds out identity prescriptions.

**Correction**: Apply REDUNDANCY-003. Personality file contains personality. Procedure files contain procedures.

---

### ANTI-004: Delete-Free Maintenance

**Description**: Rules only get added, never removed. The file is treated as an append-only log.

**Reason it fails**: Density crosses the compliance threshold; rules that mattered most when added get diluted by rules that no longer apply.

**Correction**: Schedule maintenance passes. Rules retire when the tool changes, the incident pattern stops recurring, or the rule is subsumed by a sharper version.

---

## Verification Checklist

After running the SOP, the rule stack should pass:

- [ ] Auto-loaded rule surface ≤ ~200 lines (or split by load context)
- [ ] No rule restates a system-prompt default
- [ ] Each rule category has a single SSOT file
- [ ] No rule references a deprecated tool, path, or product
- [ ] Each surviving rule has an incident or date traceable
- [ ] No rule subsumes another within the same file
- [ ] Sacred zones (identity, tone, decoded-meaning tables, iteration log) are intact

---

## Reference Data

### Example Token Budget

A real personal Claude Code configuration, May 2026, before and after one optimization pass:

| Component | Before | After | Saved |
|-----------|--------|-------|-------|
| CLAUDE.md | 5,789 chars | 4,518 chars | -22% |
| soul.md (personality) | 7,333 chars | 5,875 chars | -20% |
| lessons.md (auto-loaded) | 25,864 chars | 8,684 chars | -67% |
| **Auto-loaded total** | ~39,000 chars | ~19,100 chars | **-51%** |

Approximately 6,800 tokens saved per session. Rule count dropped from 61 lessons + many duplicates to 26 core lessons + 4 on-demand domain files (35 lessons total).

### Source Authority Hierarchy

When two rules conflict, the higher item wins:

1. Built-in safety rules (immutable)
2. System prompt
3. User's `CLAUDE.md` non-negotiables
4. Personality file (`soul.md` / `personality.md`)
5. Routing / workflow procedure files
6. Incident lessons file
7. Project-local `CLAUDE.md`
