# CLAUDE.md Rule Design — Principles for AI Agent Behavior Contracts

## Purpose

This document explains how to design rule files (CLAUDE.md, AGENTS.md, system prompts) that actually change an AI coding agent's behavior — versus rule files that look impressive but get ignored.

Generalized from 6+ weeks of iterating a personal CLAUDE.md across 5+ projects (WordPress plugins, Next.js SaaS, Electron desktop apps, Spectra SDD workflow), and a critical review of viral CLAUDE.md templates circulating in early-2026.

## When to Use

- Before adding a "best practice" rule you copied from a blog post
- When your CLAUDE.md crosses ~200 lines and rules start getting ignored
- After being burned by the same agent mistake three times — you need a rule, not a wish
- Before writing rules for a new project — pick from a real incident log, not from generic principles

---

## The "Karpathy 12 Rules" Episode (Case Study)

In January 2026, Andrej Karpathy publicly complained about three Claude coding pitfalls: silent wrong assumptions, over-complication, touching unrelated code. Forrest Chang turned the complaints into a 4-rule CLAUDE.md template that hit 120,000 GitHub stars. In May 2026, @mnilax published an extended 12-rule version, claiming:

- No CLAUDE.md → 41% mistake rate
- 4 rules → 11% mistake rate, 78% compliance
- 12 rules → 3% mistake rate, 76% compliance

The 12 rules are good *general principles* (think first, simple first, surgical changes, goal-driven; plus: no deterministic work to AI, token budget, style conflict resolution, read neighbors first, behavior-not-pass-rate tests, checkpoints, respect existing conventions, fail loudly).

**But the 3% number cannot be trusted.** The study:

- Has one author, no third-party replication
- Doesn't publish the 30 codebases, 50 tasks, or what counts as a "mistake"
- Has no significance test, no blinding, no inter-rater reliability
- Shows compliance *dropping* from 78% → 76% when going from 4 → 12 rules (i.e. more rules = lower follow-through, the opposite of what you'd want)

The 12-rule template is marketing for a rules-as-a-service blog. Take the principles, ignore the metrics.

---

## Why "More Rules" Usually Backfires

Anthropic's published guidance: CLAUDE.md compliance starts dropping noticeably past ~200 lines. The mechanism is straightforward:

- Rules compete for the model's attention budget in the system prompt
- Generic "should" rules get crowded out by specific "must" rules (and vice versa, depending on positioning)
- Vague rules ("write clean code") don't fire on any specific action — they decorate the file without changing behavior
- Each new rule dilutes priority of every existing rule

A 6-rule file targeting your three real recurring mistakes will beat a 12-rule file that includes those three plus nine generic principles.

---

## The Principle: Rules Earn Their Place

A rule belongs in CLAUDE.md if and only if:

1. **It traces to a specific incident.** "On YYYY-MM-DD, the agent did X, which cost Y." If you can't fill in the blanks, the rule is aspirational, not behavioral.
2. **It is operationally checkable.** A rule a reviewer can grade pass/fail. "Write clean code" — ungradable. "Run `npm test` and paste the result before claiming done" — gradable.
3. **It survives the negation test.** Read the rule, then read the version where it's deleted. If the file still describes the desired agent behavior without it, delete it.
4. **It is not subsumed by another rule.** If rule B implies rule A, keep B and delete A.
5. **It is not redundant with the model's defaults.** Modern coding models already try to follow conventions, write tests, avoid speculative features. Restating defaults wastes attention. Reserve rules for places where the default fails *for you specifically*.

---

## The Anti-Pattern: Wish-List CLAUDE.md

A wish-list file is one where every rule sounds reasonable but no rule was earned through a specific failure. Symptoms:

- Rules are phrased as virtues ("be thoughtful", "be careful")
- Multiple rules say the same thing in different words
- The author can't recall the last time they enforced any specific rule
- The file grows monotonically — nothing is ever deleted

The opposite — an **incident-driven CLAUDE.md** — looks like a logbook. Each rule has a tag, a brief incident description, and a remediation instruction. Rules get rewritten or deleted when a sharper version replaces them.

---

## A Workable Structure

For files that need to scale past ~30 rules without losing compliance, split by *load context*, not by topic alphabetization:

```
~/.claude/CLAUDE.md             # Entry point, ~100 lines
  → @soul.md                    # Identity + behavioral non-negotiables
  → @lessons.md                 # Cross-project incident-driven rules (~25)
  → @rules/routing.md           # Tool dispatch matrix

~/.claude/lessons-spectra.md    # Loaded on-demand when SDD workflow triggers
~/.claude/lessons-web-deploy.md # Loaded on-demand for CSS/Vercel/Postgres
~/.claude/lessons-frontend.md   # Loaded on-demand for React/Next/iOS
~/.claude/lessons-products.md   # Loaded on-demand per product
```

The auto-loaded surface stays under the compliance-drop threshold. Domain rules ride in only when the conversation enters that domain. The model's attention budget stays focused on rules relevant to the current task.

---

## Rule Drafting Template

Every rule entry follows the same shape:

```
| L0XX | <rule statement starting with a verb or "MUST"> | <trigger context> |
```

The trigger column matters. Without it, the rule has no `if`; the model has no way to decide when to apply it. With a sharp trigger ("before any rsync deployment", "when CSS doesn't fix after two attempts"), the model fires the rule reliably.

---

## When to Retire a Rule

A rule has expired and should be deleted (not archived in a separate file) when any of the following are true:

- The tool it constrains is no longer in use (e.g., a rule about Cursor when Cursor has been deprecated from the workflow)
- It has been folded into a sharper rule that fires more reliably
- The incident pattern hasn't recurred in 90+ days and the rule is now part of the agent's default behavior
- The rule applies to a single project that has its own CLAUDE.md — move it there

Retirement is healthier than accumulation. A rule file that never shrinks is dying.

---

## What Karpathy's 12 Rules Got Right

Ignoring the marketing, four of the 12 rules are durable and worth absorbing into any agent rule file:

1. **Surgical changes only.** Rule out "while I'm here, I'll clean up..." behavior. Every diff line traces to the explicit request.
2. **Read neighbors before adding.** A model that respects local convention writes code that survives review.
3. **Style conflicts get resolved explicitly.** Don't average two styles into a hybrid that matches neither. Pick one and state which.
4. **Fail loudly, not silently.** "Task completed" with quietly-skipped steps is worse than admitting "I couldn't do step 3."

The other eight are either redundant with model defaults (think first, simple first) or domain-dependent (token budget — useful for some workflows, irrelevant for others).

---

## Closing Heuristic

Before adding a new rule, ask: **"What recent incident does this prevent?"**

If the honest answer is "none, but it sounds smart" — don't add it. Your CLAUDE.md is a behavior contract with finite attention budget. Spend it on rules you've earned through real pain.
