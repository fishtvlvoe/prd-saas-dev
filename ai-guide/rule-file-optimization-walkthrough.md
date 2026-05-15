# Rule File Optimization Walkthrough

A worked example of slimming down a real CLAUDE.md / soul.md / lessons.md stack using the principles in `claude-md-rules-design.md`. This documents both *what* changed and *how the diagnosis was done*, so the same method can be replayed on any rule file.

## Purpose

The companion document `claude-md-rules-design.md` lays out the principles for designing AI agent behavior contracts. This document shows the principles applied — the actual diagnostic moves, the cuts, the parts that were deliberately left alone, and the SSOT logic that decided where each rule lives.

## When to Use

- You have a CLAUDE.md / AGENTS.md / system prompt that has grown past 500 lines and feels noisy
- Multiple rule files in your stack and you're not sure which one owns which rule
- You want to slim down without losing the rules you actually need
- You suspect parts of your rule file are redundant with the model's built-in defaults

---

## The Starting Stack

A real personal Claude Code configuration, May 2026:

| File | Lines | Chars | ~Tokens (CJK) | Role |
|------|------|-------|---------|------|
| CLAUDE.md | 125 | 5,789 | ~1,900 | Entry point: workflow, non-negotiables, completion criteria |
| soul.md | 140 | 7,333 | ~2,400 | Identity, hard rules, communication preferences |
| lessons.md | 72 (61 rules) | 25,864 | ~8,600 | Incident-driven corrections |
| rules/ × 7 | 356 | ~10,000 | ~3,500 | Routing, triggers, SSOT, pipeline, skills |
| **Auto-loaded total** | ~722 | ~50KB | **~16,700** | Every session start |

Each new session pays ~16,700 tokens just to load the rules before any actual work begins. The lessons.md file in particular packs 61 rules into 72 lines (each rule on one very long line), which crosses the density threshold where Anthropic's published guidance says compliance starts to drop.

---

## The Two-Pass Method

**Pass 1: Split lessons.md by load context.** Domain-specific rules don't need to ride in every session; they can be loaded on-demand when the conversation enters that domain. Already covered in detail by `claude-md-rules-design.md` and the project's commit `ee1aabd`. Result: auto-loaded surface 25,864 → 8,684 chars.

**Pass 2: De-duplicate across files.** Once the lessons split exposed the load-context dimension, the question for CLAUDE.md and soul.md became: are these rules earning their place, or are they shadowing rules that live elsewhere?

This document is about Pass 2.

---

## Diagnostic Move 1: Compare Against System Prompt

The Claude Code system prompt already enforces a baseline. If your rule restates a system-prompt rule, you've spent attention budget twice.

Examples found in the starting CLAUDE.md:

```
# CLAUDE.md said:
- Edit/Write 完成後不要 Read 回來驗證，信任自己剛寫的內容
```

The system prompt already contains: *"Do NOT re-read a file you just edited to verify — Edit/Write would have errored if the change failed."* — verbatim coverage. Delete the CLAUDE.md line.

```
# CLAUDE.md said:
- 除錯先查基礎項（權限、路徑、是否存在），再猜外部原因
```

Lessons.md L021 already says: *"Bug 流程：蒐集線索→列原因→工具逐一排除→確定根因→一次修復→自驗→才告知"* — wider coverage of the same instinct. Delete the CLAUDE.md line.

**Rule of thumb:** If you can't recall a recent incident where the model failed *because* it didn't have this rule (i.e., the system prompt or another rule wasn't enough), the rule isn't earning its place.

---

## Diagnostic Move 2: Find SSOT Violations

A piece of behavioral guidance should live in exactly one file. When the same guidance appears in three files, two of them are dead weight — and worse, they drift out of sync over time.

The starting stack had four SSOT violations:

1. **Tool dispatch matrix** appeared in soul.md (lines 11-22) *and* in rules/routing.md. The routing.md version was clearly the SSOT — newer, more detailed, the file other rules referenced. The soul.md version was a 13-line copy. → Collapse soul.md to "派工速查表 → routing.md" and delete the table.

2. **Spectra workflow steps** appeared in soul.md (lines 31-48) *and* in the Spectra-managed block at the top of CLAUDE.md. The Spectra block is auto-managed by the Spectra CLI — it's the SSOT by definition. → Collapse soul.md to the trigger sentence and the "skippable cases" list.

3. **SDD apply pre-dispatch checklist** appeared in soul.md (lines 50-67) *and* in lessons-spectra.md L027. → Soul keeps the 4-step protocol (the personality-level "this is what I always do"), routes detail to lessons-spectra.md.

4. **"先白話再技術"** appeared in CLAUDE.md Defaults *and* in soul.md iteration log (2026-04-27). Picked CLAUDE.md as SSOT because Defaults is exactly the right semantic home; left the iteration log untouched as historical record.

The SSOT principle isn't about which file is "better" — it's about which file has the canonical right to define this rule going forward. Other files should reference, not copy.

---

## Diagnostic Move 3: Hunt for Stale Rules

Stale rules are worse than missing rules. A missing rule fails silently. A stale rule misdirects.

Found in the starting soul.md:

```
# soul.md line 128 said:
超過 5 行程式碼 → 派 cursor-agent 或 Sonnet。
```

Cursor was deprecated months earlier — lessons.md has multiple entries about routing *away* from cursor-agent. This line was a landmine: it could send the model to dispatch work to a tool that lessons explicitly forbid.

Fixed to: *"派 Copilot CLI / Kimi CLI / Codex CLI / Sonnet 子代理（Cursor 全面禁用）"*.

Found in the starting CLAUDE.md:

```
# CLAUDE.md said:
- 判斷日誌格式 → memory/judgment-system.md
- lessons 格式 → memory/lessons.md
- today.md → memory/today.md
```

A `memory/` subdirectory existed in an earlier configuration but had been replaced — lessons now lives at `~/.claude/lessons.md`, today.md is maintained by claude-mem. Three pointers to non-existent paths.

Fixed to: *"全域記憶檔路徑：`~/.claude/lessons*.md`、`~/.claude/soul.md`、`~/.claude/today.md`（claude-mem 維護）"* — one accurate line replacing three stale ones.

**Rule of thumb:** Every six months, grep your rule files for every tool, path, and product name mentioned. Anything that no longer exists, anything that's been renamed, anything that's been deprecated — those rules are actively hurting you.

---

## Sacred Zones — What Not to Touch

When optimizing a personality file, some sections are off-limits regardless of how "redundant" they look on a token budget audit. In soul.md these were:

- **「我是誰」/「Fish 是誰」/「解讀表」** — identity and communication norms. Even if a tone-of-voice rule could be inferred from prior conversations, having it stated in soul.md is what makes it durable across cache misses.
- **「Caveman 模式」** — a deliberately stylized output rule. Cutting it because "be concise" exists elsewhere would lose the specific texture the user wants.
- **「迭代記錄」table** — historical log of corrections. Removing it might save 200 chars but loses the audit trail.

In CLAUDE.md the sacred zone was:

- **Non-Negotiables** — already a curated list, every line earned through real incidents. Touched exactly one entry (the system-prompt duplicate).

**Rule of thumb:** Personality, history, and curated lists are sacred. Tool tables and process specs are fair game.

---

## The Cuts (Final Numbers)

| File | Before | After | Saved |
|------|--------|-------|-------|
| CLAUDE.md | 5,789 | ~4,400 | -24% |
| soul.md | 7,333 | ~5,500 | -25% |
| lessons.md (auto-loaded) | 25,864 | 8,684 | -67% |
| **Total auto-loaded** | ~39,000 chars | ~18,600 chars | **~52% reduction** |

Roughly 6,800 tokens saved per session at the auto-loaded surface. More importantly: rule density dropped under the compliance threshold, so the rules that remain should fire more reliably than the larger set did.

---

## Why This Worked

The exercise didn't invent any new rules. It surfaced what was already true:

- Every rule has an SSOT — if it's not where it should be, it's noise everywhere else
- The model's defaults are part of the rule stack; treating them as zero costs you twice
- Personality data and tool data live in different files for a reason; never let them mix
- Stale rules are worse than missing rules

A good rule file isn't the one with the most rules. It's the one where every rule fires.

---

## How to Replay This on Your Own Stack

1. **Inventory.** `wc -lwc` on every auto-loaded rule file. Note total tokens.
2. **Read the system prompt of the model you're using.** Grep for any rule of yours that restates a baseline behavior. Delete those.
3. **Pick an SSOT for each rule category.** Tool routing → one file. Workflow → one file. Personality → one file. Incidents → one file. Migrate copies into references.
4. **Grep for stale references.** Every tool name, every path, every deprecated product. Update or delete.
5. **Identify sacred zones.** Do not touch personality, history, curated incident logs.
6. **Verify density.** Compliance falls as auto-loaded rules cross the model's attention threshold. Anthropic publishes ~200 lines as the marker for CLAUDE.md; treat that as a soft ceiling and use on-demand loading for everything else.
7. **Commit the diff with reasoning.** Future-you will want to know why a rule was deleted.

The whole pass should take an hour for a stack the size of the example, and pays back the time on the next several sessions.
