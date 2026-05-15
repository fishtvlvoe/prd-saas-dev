# Technology Selection Decision Tree

## Purpose

Provide a structured decision-making framework for technology selection. Every choice traces back to business requirements, not personal preference or hype.

## When to Use

- Choosing a framework, database, runtime, or architecture for a new project
- Evaluating whether to switch technologies mid-project
- Documenting technical decisions in design.md

---

## Core Method: Reverse Engineering from Requirements

### Wrong Approach (Forward, Technology-First)

```
I know technology X
    |
What can I build with X?
    |
Build an X application
```

### Correct Approach (Reverse, Requirement-First)

```
What does the market need? (What pain do users have?)
    |
What characteristics does the solution require?
(offline? speed? cross-platform? privacy?)
    |
What technologies can provide these characteristics?
    |
Among candidates, which best fits?
(not just capability -- also maintenance cost, learning curve, community)
```

---

## Decision Framework (5 Steps)

### Step 1: List Technical Constraints (From Business Requirements)

Constraints come from user needs and business model, not from technology preferences.

| Business Requirement | Technical Constraint |
|---------------------|---------------------|
| Works offline | Local data storage, no server dependency |
| Privacy-sensitive data | Data stays on device, no cloud |
| Cross-platform (Mac + Windows) | Cross-platform framework |
| One-person team maintenance | Low maintenance cost, manageable codebase |
| One-time purchase (not subscription) | Desktop app distribution, no recurring server cost |
| Time-sensitive workflow | Fast UI, minimal steps |

### Step 2: Identify Candidate Technologies (Usually 2-4)

For each constraint, list technologies that satisfy it. The intersection of all constraints narrows the candidates.

### Step 3: Compare Candidates Against Constraints

Build a comparison table:

| Dimension | Candidate A | Candidate B | Candidate C |
|-----------|-------------|-------------|-------------|
| Constraint 1 | Meets | Meets | Does not meet |
| Constraint 2 | Meets | Partially | Meets |
| Bundle size | 5 MB | 150 MB | N/A (single platform) |
| Maintenance cost | Low | Medium | Low but single platform |
| Community activity | Growing fast | Very mature | Mature |
| Learning curve | Moderate (Rust) | Low (JS only) | High (platform-specific) |

### Step 4: Select the Best Fit

Choose the candidate that satisfies all hard constraints and scores best on soft criteria (maintenance cost, community, learning curve).

### Step 5: Record Decision + Rejection Reasons

Write into design.md:

```markdown
## D[N] -- Decision Name

Choice: [selected technology]

Rationale:
- [Constraint 1]: [technology] satisfies this because...
- [Constraint 2]: [technology] satisfies this because...

Rejected [Alternative B]: [which constraint it fails, or why the trade-off is unacceptable]
Rejected [Alternative C]: [which constraint it fails, or why the trade-off is unacceptable]
```

---

## Example Decision Trees

### Desktop App vs Web App

```
Constraints:
  +-- Offline capability required --> leans desktop
  +-- Local file output (PDF to desktop) --> leans desktop
  +-- Local data storage (privacy) --> leans desktop
  +-- No multi-device sync needed --> desktop is sufficient
  +-- One-time purchase pricing --> desktop is more natural

Conclusion: Desktop App
```

### Native Framework Selection (Tauri vs Electron vs Native)

```
Known: Need desktop app, support macOS + Windows

Constraints:
  +-- One-person team --> codebase must be manageable
  +-- Familiar with React ecosystem --> excludes Qt / wxWidgets
  +-- Small distribution file (users download directly) --> small bundle preferred
  +-- Active community for long-term maintenance
  +-- Need system-level access (local DB, file I/O) --> needs native backend

Candidates:
  +-- Electron: bundle 80-150MB, Node.js backend, largest community, high resource usage
  +-- Tauri: bundle ~5MB, Rust backend, fast-growing community, low resource usage
  +-- Native (Swift/WPF): single-platform only --> excluded

Decision: Tauri
  - 5MB vs 150MB bundle = significant distribution and install experience difference
  - Backend logic is simple (CRUD + auth), Rust learning curve is acceptable
  - Lower long-term maintenance cost

Rejected Electron: High resource usage, oversized bundle
Rejected Native: Cannot cross-platform
```

### Frontend Framework Selection

```
Known: Using Tauri, need a React framework

Constraints:
  +-- Multi-page app --> needs routing
  +-- Possible future SaaS/web version --> SSR capability useful
  +-- PDF rendering uses React ecosystem
  +-- Long-term maintenance --> prefer mature solutions

Candidates:
  +-- Next.js: full framework, SSR/SSG/CSR, built-in routing, rich ecosystem
  +-- Vite + React Router: lightweight, but manual tool assembly
  +-- Remix: strong SSR, but desktop env does not need SSR

Decision: Next.js
  - Full-featured framework, future web version can reuse directly
  - No need to assemble routing and SSR manually

Rejected Vite + React Router: Assembly adds maintenance cost
Rejected Remix: Over-invested in SSR for a desktop environment
```

### Database Selection

```
Known: Desktop app, offline, local data

Constraints:
  +-- Offline --> local database, excludes cloud DB
  +-- Embedded (no server installation) --> SQLite
  +-- One-person maintenance --> avoid complex DB setup
  +-- Low data volume (hundreds of records per user)

Decision: SQLite (embedded)
  - Zero config, mature ecosystem support
  - Data volume fits perfectly

Rejected PostgreSQL: Requires server, unsuitable for desktop
Rejected IndexedDB: Web technology, not accessible from native backend
```

### PDF Rendering Selection

```
Known: Need high-quality PDF, precise formatting, React ecosystem

Constraints:
  +-- Precise layout control (legal document formatting)
  +-- React ecosystem compatibility
  +-- CJK font support
  +-- High customization (logo, themes, formatting)

Decision: @react-pdf/renderer
  - Declarative React API, consistent with frontend code style
  - Full CJK support, flexible customization

Rejected PDFKit: Low-level API, manual layout management, high development cost
Rejected Puppeteer (Print to PDF): Limited format control, heavy dependency in desktop env
```

---

## Logic Tree Template

For any technology selection:

```
[Business Goal / User Need]
  +-- [Constraint A] (technical boundary)
  |     +-- Candidate 1
  |     +-- Candidate 2
  |     +-- Candidate 3 --> selected, reason: ...
  +-- [Constraint B]
  |     +-- ...
  +-- [Constraint C]
        +-- ...
            |
        Intersection (satisfies all constraints)
            |
        Secondary comparison (cost, ecosystem, maintenance)
            |
        Final selection + decision rationale (write into design.md)
```

Every choice has a corresponding constraint (derived from business requirements), not "I am familiar with this" or "this technology is new and cool."

---

## Selection Pitfalls

**Pitfall 1: Technology FOMO (Fear of Missing Out)**

"This technology is new, everyone is talking about it, we should use it." New does not equal appropriate.

**Pitfall 2: Over-Engineering**

"We might need this capability in the future, so choose the option that supports it now." Build for current needs. Re-evaluate when you actually need it.

**Pitfall 3: Ignoring Maintenance Cost**

"This option is the most powerful." Powerful usually means higher learning curve and maintenance cost.

**Pitfall 4: Familiarity Replacing Fitness**

"I know X, so I will use X." Familiar does not equal appropriate for this problem.

### Correct Questions to Ask

- "Which specific constraint does this technology address?"
- "If we do not choose this, what is the biggest cost?"
- "Three years from now, who maintains this technology decision?"

---

## Decision Record Format (for design.md)

```markdown
## D[N] -- [Decision Name]

Choice: [selected approach]

Rationale:
- [Constraint 1]: satisfies because [reason]
- [Constraint 2]: satisfies because [reason]
- Secondary: [maintenance cost / community / learning curve advantage]

Rejected [Alternative B]: [reason -- which constraint fails or unacceptable trade-off]
Rejected [Alternative C]: [reason -- which constraint fails or unacceptable trade-off]
```

Six months later, anyone reading this can immediately understand why this choice was made and under what conditions it should be reconsidered.
