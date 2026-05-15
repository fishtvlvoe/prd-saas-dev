# Design Principles

## Purpose

Provide a constraint checklist for design decisions. Every design choice should be traceable to one of these principles. If a decision cannot be traced back, it needs re-examination.

## When to Use

- Making any feature, UX, or architecture decision
- Evaluating whether to add or remove a feature
- Reviewing design proposals
- Resolving design disagreements

---

## Foundation

**The tool adapts to the person, not the person to the tool.**

Most tools are designed as: "The tool provides features, the person learns to use them."

The correct approach is reversed: **"First understand what the person actually does, then decide what the tool looks like."**

This is not marketing copy. It is the litmus test for every design decision. If a feature makes the user feel "I am accommodating this tool," the feature is wrong.

---

## Four Core Principles

### 1. Entropy Reduction

**Physics**: The second law of thermodynamics -- entropy (disorder) in a closed system only increases, never decreases spontaneously.

**Applied to product development**: Without active management, features only accumulate, complexity only grows, and systems never simplify themselves.

**Entropy increase is the natural state. Entropy reduction is an active choice.**

#### Application Areas

| Area | Entropy Reduction Practice |
|------|---------------------------|
| Feature design | Only build what solves a validated core pain point |
| Interface design | Eliminate every unnecessary operation step |
| Code design | Do not add dependencies unless unavoidable |
| Development workflow | Eliminate unnecessary waiting time (e.g., mock backends) |

#### The Three Entropy Reduction Questions

Before any "should we add this?" decision:

1. **What is its responsibility?** Cannot explain in one sentence -> do not add.
2. **What happens if we do not add it?** No immediate impact -> do not add.
3. **Who maintains it after it is added?** No clear maintenance owner -> do not add.

All three answerable -> then consider adding.

#### Feature Addition Flow

```
Should we add this feature?
    |
What pain point does it solve?
    |
Is this pain point validated (with real users)?
    |
What is the maintenance cost?
    |
Maintenance cost < pain point value --> add
Maintenance cost >= pain point value --> do not add
```

#### Minimum Change Principle

- Bug fixes: only change what makes the bug disappear
- Do not "while I'm at it" refactor nearby code (unless it directly causes the bug)
- Do not "might as well" add "possibly needed later" features
- Smaller change scope = easier rollback = lower risk

#### Dependency Management

Before adding any package, ask:
1. Can we implement this in under 50 lines ourselves? (If yes, write it yourself)
2. What is the package's maintenance status? (No updates in 3 years = do not add)
3. What is the bundle size impact?

---

### 2. Inside-Out (Why Before How)

**Design order**: First ask why it exists, then design how it works.

**Wrong order (Outside-In)**: See what competitors built -> copy features -> rationalize after

**Right order (Inside-Out)**: Confirm the core pain point -> define "what solving looks like" -> design features

**Reasoning chain**: Why (purpose) -> What (corresponding feature) -> How (specific design)

**Reverse must also work**: How (why designed this way) -> What (which requirement) -> Why (back to user pain point)

If the reverse chain breaks, the design has a problem.

---

### 3. Externalized Meta-Framework

The user must always know three things while using the tool:

1. **Where am I now** (state is visible)
2. **What is the next step** (action path is clear)
3. **What if something goes wrong** (errors are guidance, not judgment)

When these three are achieved, the user has a sense of control.

---

### 4. Failure as Logging

Failure is not an identity verdict. It is a debug record.

**Two layers of application**:

| Layer | Practice |
|-------|----------|
| Product design | When the user makes a mistake, the UI says "try this instead," not "you are wrong" |
| Development workflow | When you hit a pitfall, record it in lessons, learn, do not repeat -- failure is learning material, not shame |

---

## Serotonin Design vs Dopamine Traps

| Aspect | Serotonin Design (recommended) | Dopamine Design (common in industry) |
|--------|-------------------------------|--------------------------------------|
| Goal | User finishes and leaves calmly | User keeps coming back |
| Means | Sense of control, feeling understood, clear next steps | Notifications, progress bars, infinite levels |
| User feeling | "The tool did its job" | "I need this tool" |
| Long-term effect | User trusts the tool, recommends it | User depends on the tool, but also resents it |
| Example | Document is done, session ends, no "come back" hooks | -- |

**Core of serotonin design**: The tool completes the task. The user continues with their life.

---

## Entropy Reduction vs Entropy Increase in Business

| Entropy Increase Thinking | Entropy Reduction Thinking |
|--------------------------|---------------------------|
| Add features to make the product stronger | Only add features that are truly needed |
| Users asked for it, so add it | First ask what the user's actual need is |
| More complex code = smarter | Simpler code = harder to break |
| Build first, figure it out later | Define the correct standard first, then build |
| More dependencies = more powerful | More dependencies = higher maintenance cost |
| Documentation comes after | Documentation is part of design |

---

## Bidirectional Verification Tool

After every design decision, run this two-way check:

**Forward**:
```
User pain point --> which feature --> specific design --> which principle it embodies
```

**Reverse**:
```
Design decision --> which principle --> which need it serves --> is the need real or assumed
```

Reverse chain breaks = design has a problem, or the principle needs updating.

---

## 12-Engine Quick Index

| # | Engine | Judgment Question |
|---|--------|-------------------|
| 1 | Entropy Reduction | Is the system cleaner after than before? Does every addition have a clear responsibility? |
| 2 | Underlying Migration | Strip the labels, keep the skeleton -- can it transfer to another domain? |
| 3 | Inside-Out | Did we ask why it exists before learning how to use it? |
| 4 | Externalized Meta-Framework | Is state visible? Is the next step clear? Are errors guidance, not verdicts? |
| 5 | Serotonin Design | Does it let users finish and leave calmly? No addiction hooks? |
| 6 | Failure = Log | Is failure a debug record, not an identity sentence? |
| 7 | Three-Question Test | Simpler for users? Easier for engineers to maintain? Better for future extension? |
| 8 | Opinionated Design | When there is a best answer, does the system decide without asking the user? |
| 9 | Separation of Concerns | Does the system manage its own chaos without overstepping into user decisions? |
| 10 | Discrete Code, Continuous Validation | Does deterministic logic produce a "this understands me" feeling? |
| 11 | Four-Layer Architecture | Built from the principle layer up, not guessing down from the experience layer? |
| 12 | Gray-Scale Thinking | Marks applicability boundaries instead of binary right/wrong judgments? |

---

## Design Decision Checklist

Before finalizing any design decision:

- [ ] Can this decision be traced back to a validated user pain point?
- [ ] Does the reverse reasoning chain hold (design -> principle -> need -> pain)?
- [ ] Have rejected alternatives been documented with reasons?
- [ ] Does this pass the three entropy reduction questions?
- [ ] Does the user maintain a sense of control (knows where they are, what is next, what to do on error)?
