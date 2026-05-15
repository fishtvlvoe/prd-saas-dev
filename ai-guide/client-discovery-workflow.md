# Client Discovery Workflow

## Purpose

Define the human-AI collaboration protocol for evaluating a new prospect: who finds the problem, who maps the assets, who decides direction. Prevents the AI from generating both problems and solutions (which leads to self-reinforcing confirmation bias).

## When to Use

- A new client lead comes in
- The user shares a Perplexity / market research link
- The user says "help me look at this client"
- Before any `/spectra-propose` for a client engagement

---

## Roles

| Actor | Responsibility | Why |
|-------|----------------|-----|
| User (Fish) | Problem discovery via external sources (Perplexity, interviews) | External citations prevent self-justification |
| AI agent | Asset inventory and mapping | AI knows the internal codebase best |
| User (Fish) | Final go / no-go decision | Business judgment requires human owner |

---

## Three-Stage Pipeline

```
Stage 1: User runs external research (Perplexity / Gemini / interviews)
   |
   v
Stage 2: AI maps pain points to owned assets, produces options
   |
   v
Stage 3: User decides direction (A / B / C / modify / drop)
   |
   v
If go: trigger /spectra-propose
```

---

## Stage 2: AI Execution Steps

When the user provides Perplexity output or research summary, execute in order:

### Step 1: Confirm input quality

- [ ] Is there a named client?
- [ ] Are the pain points concrete (not abstract market needs)?
- [ ] Do pain points pass the "repetitive, time-consuming, hard-to-measure" filter?

If any check fails: ask the user to refine research before proceeding.

### Step 2: Inventory owned assets

Do not rely on memory. Re-scan every time:

```bash
ls products/
ls 8-外掛/
ls skills/
```

For each asset, record:
- Original problem it solved
- Whether it can be retooled for the client's pain (low / mid / high modification)

### Step 3: Build pain-to-asset mapping table

| Client Pain | Owned Asset | Modification Effort |
|-------------|-------------|---------------------|
| A           | X           | low / mid / high    |

Apply the methodology in `client-opportunity-mapping.md`.

### Step 4: Generate three-tier proposal

| Tier | Model | Client Commitment | Our Risk |
|------|-------|-------------------|----------|
| A | SaaS rental of existing tools | lowest | lowest |
| B | Co-development with client as case study | mid | mid |
| C | Full integration build + monthly | highest | highest (but most replicable) |

### Step 5: Recommend MVP

Selection criteria (both must hold):
1. Decision-maker (parent / boss / paying party) can feel the value within 7 days of launch
2. Operator (teacher / staff) has zero learning curve

If no candidate meets both: redo asset mapping or recommend dropping the client.

### Step 6: Output decision package to user

Format:
```
1. Mapping table
2. Three tier options (A/B/C) with scope, timeline, pricing basis
3. MVP recommendation with rationale
4. Anti-pattern flags (when NOT to take this client)
```

---

## Signal-to-Action Table

| User signal | AI action |
|-------------|-----------|
| Drops a Perplexity link or research dump | Enter Stage 2 |
| "Help me look at this client" without research | Suggest running Stage 1 first |
| "This proposal is wrong" | Return to mapping table, do not argue |
| "Go with A" | Trigger `/spectra-propose` |

---

## Anti-Patterns

- Do NOT generate pain points yourself (no external validation)
- Do NOT skip asset inventory and jump to "let's build X"
- Do NOT propose only one option (always three tiers)
- Do NOT recommend a new project when an existing asset covers the pain (violates entropy reduction)

---

## Cross-References

- Stage 2 detailed methodology: `client-opportunity-mapping.md`
- Pain validation criteria: `persona-pain-scenario.md`
- Post-decision workflow: `sdd-workflow.md`
