# Client Opportunity Mapping

## Purpose

Determine in under 10 minutes whether owned assets can solve a prospect's pain points, and at what collaboration depth. Operates as Stage 2 of `client-discovery-workflow.md`.

## When to Use

- After receiving external research (Perplexity output, interview notes) about a prospect
- Before drafting any proposal or `/spectra-propose`
- When deciding whether to take or drop a client engagement

---

## Core Heuristic

Do not start from "what does the client need." Start from "what we already have × where it intersects with their pain." No intersection → drop the client. Do not force-fit by building new things.

---

## Four-Dimension Pre-Check (Run Every Time, Skipping = Pivot Hell)

Distilled from a real case with 4 wrong-direction pivots (see `pitfalls.md` — Client Discovery 4 Pivots). Before proposing any MVP, pass these four checks:

### Dimension 1: Who pays?

The payer is the real customer. The user is often not the payer.

| Context | Apparent customer | Real customer (payer) |
|---------|-------------------|-----------------------|
| Cram school | Children | Parents |
| B2B SaaS | End users | Procurement / boss |
| Pet products | Pet | Owner |
| Children's toys | Kid | Parents / grandparents |

Check: Does the solution solve the payer's pain? Solving the user's need but the payer feels nothing → no purchase.

### Dimension 2: Pleasure point vs Pain point

| Type | When used | When not used | Example |
|------|-----------|---------------|---------|
| Pleasure point | Happy, will share | Nothing happens | Result video, cute animation |
| Pain point | Anxiety relief, can sleep | Persistent anxiety → finds competitor | AI emotional companion, instant relief |

Rule: MVP main dish must be a pain killer. Pleasure points can be side dishes (amplify spread) but never the core sell.

Anti-pattern: Selling "child speaking English video" as main MVP. That's a pleasure point — parents feel good momentarily but the core anxiety about "my child's English" is untouched.

### Dimension 3: Friction check

Customers pay to outsource hassle. A solution that adds hassle back = inverted logic.

| Friction level | Example | Will buy? |
|---------------|---------|-----------|
| Zero | Passive LINE push | ✅ |
| Low | Voluntary click when curious | ✅ |
| Mid | 10 min weekly cooperation | ⚠️ Depends on value |
| High | Daily task to perform | ❌ Cancellation |

Anti-pattern: "Every-night parent-child English task" — parents send kids to cram school precisely to outsource this. Throwing it back at them = no buy + complaints.

Check: If I were the payer, does this add to my burden?

### Dimension 4: External research mandatory

Run external research (Perplexity, client interviews, public community discussions) before guessing pain points.

Anti-pattern: Assuming "teaching quality is the cram school's pain" based on stereotypes. A franchise cram school often has teaching covered by HQ — the real pain is enrollment/marketing.

SOP: Stage 1 of `client-discovery-workflow.md` is mandatory. Skipping = the start of consecutive pivots.

---

## Five-Step Procedure

### Step 1: Filter pain points

Keep only pains that satisfy all three:
- Repetitive (occurs daily or weekly)
- Time-consuming (over 30 minutes per occurrence)
- Hard to measure (the paying party cannot see progress)

Reject pains that fail any filter. AI tools are not the right intervention.

### Step 2: Inventory assets (no memory shortcuts)

Re-scan each time:

```bash
ls products/
ls 8-外掛/
ls skills/
```

For each asset, document:
- Original problem solved
- Retoolability for the current pain (low / mid / high)

### Step 3: Build the mapping table

| Pain | Owned Asset | Modification Effort |
|------|-------------|---------------------|
| A    | X           | low/mid/high        |

Modification effort drives pricing tier:
- Low effort → SaaS rental
- Mid effort → co-development
- High effort → full project

### Step 4: Choose collaboration depth

| Tier | Mode | Client Cost | Our Risk |
|------|------|-------------|----------|
| A | Monthly SaaS rental of existing tools | lowest | lowest |
| B | Co-development with client as case study (revenue share or discount) | mid | mid |
| C | One-time integration build + monthly subscription | highest | highest, but template for future replication |

A is for testing the waters. C is the starting point for building a replicable template.

### Step 5: Select MVP

Selection rules (both must hold):
1. The decision-maker (paying party) can feel value within 7 days of launch
2. The operator (end user of the tool) has zero learning curve

If both rules cannot be satisfied: redo Step 3 or recommend dropping.

Do NOT select MVP based on technical novelty.

---

## Worked Example: English Cram School (2026-05)

### Step 1: Filtered Pains

1. Slow speaking pronunciation feedback (teacher cannot do 1-on-1 corrections at scale)
2. Wide student skill gap (same classroom, different levels)
3. Teacher lesson-prep overhead (multiple difficulty versions)
4. Parents cannot see progress
5. No after-class practice partner

### Step 2: Owned Assets

- Voicebox (voice processing)
- MOLTOS (learning dashboard)
- AIRE (document / question bank generation)
- LineHub (LINE bot + notifications)
- BuyGo+1 (payment / enrollment)
- Webinar-Go (live streaming + recording)
- Paygo (subscription billing)

### Step 3: Mapping Table

| Pain | Asset | Modification |
|------|-------|--------------|
| Slow speaking feedback | Voicebox retooled for English shadowing + scoring | mid |
| Parents cannot see progress | MOLTOS dashboard | low |
| Teacher prep overhead | AIRE customized question banks | mid |
| No after-class practice | LineHub 24h practice bot | mid |
| Parent notifications | LineHub + BuyGo+1 | low |
| Class recording | Webinar-Go | low |
| Tuition collection | Paygo | low |

### Step 4: Three Tiers

- A (rental): LineHub notifications + MOLTOS reports, monthly fee
- B (co-development): School as first case study, revenue share
- C (full): Enrollment → class → practice → reporting → billing, one-time build + monthly

### Step 5: MVP Selection

LINE bot speaking practice + weekly parent report.

Rationale:
- Built on existing LineHub + Voicebox, deliverable in 2-3 weeks (controlled modification effort)
- Strongest signal to paying party (weekly progress report direct to parents)
- Zero teacher learning curve (students use it via LINE, backend runs automatically)

---

## Drop Criteria

Decline the client if any holds:
- No intersection between filtered pains and owned assets (new build cost too high)
- Client wants AI to "replace the expert," not assist (violates tool-serves-human principle in `design-principles.md`)
- MVP requires significant workflow change from the operator (adoption friction will dominate)

---

## Cross-References

- Upstream workflow: `client-discovery-workflow.md`
- Pain extraction: `persona-pain-scenario.md`
- Pain validation: `decision-tree.md`
- Post-decision execution: `sdd-workflow.md`
- Why reuse beats new-build: `design-principles.md` (entropy reduction section)
