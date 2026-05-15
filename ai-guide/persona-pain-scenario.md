# Persona-Pain-Scenario Framework

## Purpose

Provide a structured requirements analysis framework that moves from "feels like a good idea" to "confirmed this is worth building." Applicable to any product's target user analysis.

## When to Use

- Starting a new product or feature
- Validating whether a problem is real
- Defining product scope and boundaries
- Before any design or implementation work

---

## Framework Structure

```
Persona (who)
    |
Pain (what cost are they bearing)
    |
Scenario (when and where does the pain occur)
    |
Need (what outcome do they actually require)
    |
Solution (what to build -- and what NOT to build)
```

Every arrow must be explainable with "why." No skipping steps.

---

## Step 1: Persona (Specific People, Not Abstract Markets)

**Definition**: A concrete person, not an abstract market segment.

| Wrong | Right |
|-------|-------|
| "Real estate professionals" | "Independent real estate agents with 5-15 years experience and their own client base" |
| "Small business owners" | "Solo online sellers who take orders via social media groups, manually replying to messages daily" |
| "People who need AI tools" | (This is a technology, not a person -- reject this framing) |

**How to make it concrete**: Age, occupation, work style, technical familiarity, what they do every day.

### Checklist

- [ ] Can you name a real person who fits this persona?
- [ ] Have you described their daily work routine?
- [ ] Is the persona narrow enough to have specific, identifiable pain points?

---

## Step 2: Pain (Concrete Costs, Not Inconveniences)

**Definition**: The specific cost this person is currently bearing -- time, money, mental burden.

"Inconvenient" is not enough to make someone pay or change habits. **Cost** is.

| Not specific enough | Specific (with cost) |
|--------------------|----------------------|
| "Making reports is annoying" | "Each report takes 2-3 hours; sometimes clients are urgent and I work overtime" |
| "Worried about compliance" | "Regulations change yearly; last time I filled in the wrong clause, the client complained, and I spent half a day redoing it" |
| "Inconsistent formatting" | "Company has a standard format but employees each do their own thing; clients receive inconsistent documents, reducing professional image" |

### How to Validate Pain

1. **Ask directly** -- but ask "when was the last time this happened?" not "do you have this problem?"
2. **Observe** -- shadow the user for half a day
3. **Find existing solutions** -- if they already use some tool to solve it, the pain is real

### Checklist

- [ ] Can you quantify the cost (hours, dollars, incidents per month)?
- [ ] Have you validated the pain with actual users (not just your assumption)?
- [ ] Is the pain recurring, not a one-time event?

---

## Step 3: Scenario (When and Where Pain Occurs)

**Definition**: The specific context in which the pain happens -- time, place, surrounding conditions.

Scenarios tell you what conditions the solution must operate under.

### Scenario Template

```
Scenario: [name]
Time: [when does this happen]
Location: [where -- office, mobile, field]
Context:
  - [What data/resources do they have at hand]
  - [What task are they trying to complete]
  - [What constraints exist]
Pain trigger: [the exact moment the pain occurs]
```

### What Scenarios Reveal About Solution Design

| Scenario detail | Design implication |
|----------------|-------------------|
| User is at a computer | Desktop app is appropriate; mobile version not needed |
| Needs PDF output | PDF rendering is a core feature |
| Reference data is known/static | Can be built-in; user does not need to look it up |
| User is in the field with spotty internet | Offline capability required |
| Task is time-sensitive | Speed and minimal steps are critical |

### Checklist

- [ ] Have you described the scenario with enough detail to derive design constraints?
- [ ] Does the scenario include the exact pain trigger moment?
- [ ] Can you trace each future design decision back to a scenario detail?

---

## Step 4: Need (Outcomes, Not Features)

**Definition**: What the user actually needs to achieve, extracted from the scenario. This is NOT a feature list.

Need and Feature are different things:

| Scenario | Need | Feature |
|----------|------|---------|
| Finding the right legal clause is slow | Quickly select the correct clause | Clause database + search |
| Document formatting is hard to control | Output must have consistent formatting | Template engine + PDF rendering |
| Need to include company branding | Output includes company brand | Brand settings (logo + company name) |
| Client is waiting urgently | Complete the entire process quickly | Workflow optimization (fewer steps) |

**From need to feature is where "design" happens.** Do not skip needs and jump straight to features.

### Checklist

- [ ] Is each need described as an outcome, not a solution?
- [ ] Can you trace each need back to a specific scenario?
- [ ] Have you resisted the urge to define features at this stage?

---

## Step 5: Solution (What to Build AND What NOT to Build)

**Definition**: The feature set designed to address the needs -- with explicit boundaries on what is excluded.

### Solution Table Template

| Need | Solution | Will NOT Do |
|------|----------|-------------|
| Quickly select correct clause | Built-in clause database, searchable, one-click insert | Custom clause authoring (regulations are fixed) |
| Consistent formatting | Standardized PDF template, no user format editing | Free-form format editor (freedom = inconsistency) |
| Company branding | Brand settings page (set once, auto-applied) | Per-document brand settings |
| Fast workflow | Minimum fields principle (only ask for required info) | Full property database / CRM direction |

**Key judgment**: Solution boundaries come from "what NOT to do." Without clear boundaries, features expand indefinitely.

### Checklist

- [ ] Does each solution trace back to a validated need?
- [ ] Is the "Will NOT Do" column filled for every solution?
- [ ] Have you resisted scope creep from "nice to have" features?

---

## Using This Framework

Before starting any new feature, walk through these five steps:

1. Who is this for (a specific person)?
2. What cost are they bearing right now?
3. In what scenario does this cost occur?
4. What is their actual need (not what they say they want)?
5. Where is the solution boundary (what will you NOT build)?

Complete this analysis, then begin design.

---

## Problem Validation: Falsifiable Thinking

Every assumption must be falsifiable (can be proven wrong).

### Falsifiable vs Non-Falsifiable

| Type | Example |
|------|---------|
| Falsifiable | "Users spend an average of 2-3 hours producing each report" -- disproven if interviews show average under 30 minutes |
| Non-falsifiable | "Users need better tools" -- nothing can disprove this, so it is meaningless |

### Validation Process

```
What is the problem?
    |
Write it as a falsifiable proposition (specific, measurable, with disproval conditions)
    |
What evidence could disprove it?
    |
Go look for disproving evidence (not supporting evidence)
    |
No disproving evidence found?
--> Tentatively accept the problem as real
    |
Write the solution as a falsifiable proposition
    |
Test it (red tests)
```

### Why Seek Disproving Evidence

Humans have confirmation bias: we tend to find evidence supporting our ideas and ignore contradicting evidence.

**Wrong question**: "Do you find making reports annoying?" (Users will say yes -- but annoyed does not equal willing to pay.)

**Right approach**: Find conditions that would invalidate the assumption.
- "If their company already has a system doing this for them" -> assumption invalid
- "If they only need to do this once per month, time cost is negligible" -> assumption invalid
- "If they are already fast with their current tool" -> assumption invalid

### Quick Validation Checklist

- [ ] Is the assumption specific (has numbers, has names)?
- [ ] What is the disproval condition (if X happens, it is wrong)?
- [ ] Have you actively looked for disproving evidence?
- [ ] Could the supporting evidence you found be confirmation bias?

---

## Common Error Patterns

**Error 1: Treating "possible" as "certain"**
"Users might need this feature" -> No disproval condition -> Meaningless.

**Error 2: Leading questions**
"Do you think automating reports would be helpful?" -> Users always say yes -> Pain not validated.

**Error 3: Sample bias**
Only asking "people already looking for a solution" -> Of course they have the pain -> Does not prove the market is large enough.

**Error 4: Confusing feature requests with problems**
"Users want feature X" is not the same as "X solves a real problem." First ask: what problem does X solve?
