# AI Implementation Guide

This directory contains structured, actionable guides for AI agents (Claude, Codex, GPT, etc.) to follow when building SaaS or desktop applications using Spec-Driven Development.

All content is derived from battle-tested practices -- not theory.

---

## File Index

| File | Purpose | When to Use |
|------|---------|-------------|
| [sdd-workflow.md](./sdd-workflow.md) | Step-by-step SDD (Spec-Driven Development) execution flow | Starting any feature, bug fix, or refactor |
| [tdd-pattern.md](./tdd-pattern.md) | Red-Green-Refactor TDD pattern with failure matrix | Before writing any implementation code |
| [persona-pain-scenario.md](./persona-pain-scenario.md) | Requirements analysis framework: Persona, Pain, Scenario, Need, Solution | Analyzing target users for any product |
| [design-principles.md](./design-principles.md) | Design constraint checklist for decision-making | Making any design or feature decision |
| [dev-environment.md](./dev-environment.md) | Three development modes for desktop/hybrid applications | Setting up dev workflow for desktop apps |
| [decision-tree.md](./decision-tree.md) | Technology selection decision tree | Choosing frameworks, databases, or architecture |
| [pitfalls.md](./pitfalls.md) | Common SaaS/desktop development pitfalls and solutions | Avoiding known failure modes |
| [templates/native-bridge-template.md](./templates/native-bridge-template.md) | IPC bridge template with dev mock support | Building desktop apps with web frontends |
| [templates/release-checklist.md](./templates/release-checklist.md) | Pre-release verification checklist | Before any version release |

---

## How to Use This Guide

1. **Starting a new feature**: Read `sdd-workflow.md` first, then `tdd-pattern.md`
2. **Starting a new product**: Read `persona-pain-scenario.md`, then `design-principles.md`, then `decision-tree.md`
3. **Setting up development**: Read `dev-environment.md` and `templates/`
4. **Debugging or stuck**: Check `pitfalls.md` for known failure patterns

---

## Conventions

- All files use structured format: **Purpose**, **When to Use**, **Steps/Checklist**
- Product-specific references have been replaced with generic terms
- Code blocks are preserved from original sources with English comments
