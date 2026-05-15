# Development Pitfalls and Solutions

## Purpose

Document common SaaS and desktop application development pitfalls with root causes and solutions. These are generalized from real incidents -- each entry was encountered and resolved in production.

## When to Use

- Before starting a task in a related area (check for known pitfalls)
- When debugging an issue that matches a symptom described here
- During code review to verify known failure modes are addressed

---

## Desktop Application Development

### PIT-001: Native App Build Wait Time Kills Development Velocity

**Symptom**: Every UI change requires a full native build (2-3 minutes). 30 iterations/day = 90 minutes waiting.

**Root Cause**: Browser mode (`pnpm dev`) shows a fallback "requires desktop app" message because all IPC calls fail without the native backend. Developers are forced into native build mode for any testing.

**Solution**: Implement a mock backend that intercepts IPC calls in browser mode.

**Key design**:
- Single entry point (`safeInvoke`) with environment branching
- `!isNativeEnv() && NODE_ENV === "development"` -> mock branch
- MockStore is an in-memory singleton with `reset()` for tests
- Dynamic import ensures production builds exclude mock code entirely

**Lesson**: Any desktop app with a web frontend must have a mock backend. Without it, UI development velocity is 10x slower than web development.

---

### PIT-002: Icon Replacement Does Not Take Effect

**Symptom**: Replaced icon files in the icons directory manually. The application still shows the old icon.

**Root Cause**: Desktop frameworks (Tauri, Electron) require icons in multiple sizes for different platforms and display contexts. Manual replacement misses some sizes.

**Solution**: Always use the framework's icon generation command from a single 1024x1024 source image.

```bash
# Example for Tauri
pnpm tauri icon src/app-icon.png
# Generates all required sizes automatically
```

**Lesson**: Never manually copy icon files. Always regenerate from a single source.

---

### PIT-003: Multi-Architecture Build Artifact Name Collision

**Symptom**: Building for both arm64 and x64 appears to succeed, but only one architecture's file exists in the release. The other was silently overwritten.

**Root Cause**: Artifact name template does not include `${arch}`. Both architectures produce the same filename; the second build overwrites the first.

**Solution**: Include architecture identifier in artifact names.

```json
{
  "artifactName": "${name}-${version}-${arch}.${ext}"
}
```

**Lesson**: Multi-architecture builds must include architecture in artifact names. Check release assets after building to verify both exist.

---

## Frontend / Next.js Development

### PIT-004: Mobile Testing Blocked by Cross-Origin Protection

**Symptom**: Testing from a mobile device on LAN (e.g., `http://192.168.x.x:3000`). CSS/JS are blocked by cross-origin protection. React hydration fails. Page shows loading skeleton forever.

**Diagnosis clues**: API calls return normally. HTML has content. But the skeleton never disappears.

**Solution** (Next.js 16+):

```typescript
const nextConfig = {
  experimental: {
    allowedDevOrigins: ['192.168.x.x'],  // your LAN IP
  },
}
```

Restart the dev server after changing.

**Lesson**: Next.js 16+ requires explicit `allowedDevOrigins` for mobile/LAN testing.

---

### PIT-005: Non-ASCII Characters in Project Path Cause Dev Server to Hang

**Symptom**: Dev server starts, but the page is blank. Root div is empty. No console errors. `@fs` URLs in network tab contain URL-encoded non-ASCII characters that time out.

**Root Cause**: Vite / Next.js / Webpack dev servers output `@fs` URLs containing the real file path. Non-ASCII characters in the path get URL-encoded, causing fetch timeouts.

**Symlinks do not work**: Node.js resolves symlinks to the real path. The dev server still gets the non-ASCII path.

**Solution**: Move the project to a path with only ASCII characters.

```bash
mv /path/with/non-ascii/project /path/with/ascii/project
```

**Lesson**: Any project using Vite, Next.js, or Webpack dev server must be in an ASCII-only path.

---

### PIT-006: Service Worker from Previous Project Causes Blank Page

**Symptom**: Same localhost port previously ran a different project. New project shows a blank page (or the old project's UI). But `curl localhost:3000` returns the correct HTML.

**Diagnosis**: `curl` returns correct content but the browser shows wrong content = Service Worker is intercepting and returning cached responses.

**Solution** (browser console):

```javascript
(async () => {
  const r = await navigator.serviceWorker.getRegistrations()
  for (const x of r) await x.unregister()
  const k = await caches.keys()
  for (const c of k) await caches.delete(c)
})()
```

Or: DevTools -> Application -> Storage -> Clear site data.

**Lesson**: When switching projects on the same localhost port, always clear Service Workers and caches first.

---

## UI Verification

### PIT-007: UI Task Verified by Code Inspection Passes, But Rendering is Wrong

**Symptom**: Agent reports UI task complete. `grep` confirms HTML structure exists. But actual browser rendering has errors (tab switching logic wrong, CSS override issues, content in wrong section).

**Root Cause**: Code structure being correct does not guarantee correct rendering. CSS interactions, JS toggle logic, and component state can all produce wrong visual output from correct-looking code.

**Solution**: Every UI task must be visually verified before commit.

```
Agent reports completion
    |
Take screenshots of all pages/states (Playwright / browser automation)
    |
Review screenshots to confirm correct rendering
    |
Wrong --> return to Agent for fixes
Correct --> commit
```

**Lesson**: "Agent reports complete + grep finds the string" is never sufficient for UI tasks. Screenshots are the only objective evidence.

---

## SDD Workflow

### PIT-008: Treating SDD as Waterfall

**Symptom**: Spec is written, then treated as immutable. When requirements change, the team either forces implementation to match the outdated spec, or diverges from the spec without updating it.

**Root Cause**: Misunderstanding SDD as "blueprint then build" (construction metaphor) instead of "hypothesis then test" (scientific method).

**Correct understanding**:
- Spec = hypothesis (can be disproven)
- Red tests = falsification experiments
- Green tests = hypothesis tentatively holds
- Requirement change = new evidence -> update spec (ingest), re-analyze, continue

**Lesson**: Never "build first, then fix the spec." Update the spec first, always.

---

### PIT-009: Skipping Analysis Findings

**Symptom**: Analysis reports Suggestion-level findings. Developer assumes they can be skipped. After implementation, inconsistencies between spec and code surface, requiring rework.

**Root Cause**: Treating Suggestions as optional. They mark potential future misunderstandings that are cheap to fix in the propose phase and expensive to fix in the apply phase.

**Solution**: Fix all findings -- Critical, Warning, and Suggestion -- to reach 0 findings before proceeding.

**Lesson**: In the propose phase, every finding is cheap to fix. In the apply phase, the same finding costs 10x more.

---

## External Agent Tooling

### PIT-010: AI Agent in Unrestricted Mode Deletes SDD Files

**Symptom**: Dispatched an AI coding agent in unrestricted mode. Agent saw untracked files in the working directory (SDD artifacts, config files) and ran `git clean -fd` or `git restore` to "clean up" before starting its task. SDD files permanently deleted.

**Root Cause**: Unrestricted agents have full bash access. Directory restriction flags (like `--add-dir src/`) only limit which files the agent actively edits, not which bash commands it runs.

**Solution** (mandatory before dispatching any agent):

```bash
# 1. Commit all important files first
git add openspec/ .claude/ docs/ && git commit -m "wip: pre-dispatch checkpoint"

# 2. Restrict working directory to code-only
agent --add-dir src/ -p @prompt.txt

# 3. After agent completes, verify no unexpected changes
git diff --stat
```

**Additionally**: Include explicit prohibitions in the agent prompt: "Only run `git diff`, `git status`. Do not run `git clean`, `git restore`, `git checkout`, `git reset`."

**Lesson**: Never dispatch an agent with important uncommitted files in the working tree. Commit first, dispatch second.

---

### PIT-011: Agent Working Directory Targets Wrong Git Repository

**Symptom**: Agent runs implementation tasks, makes commits, but commits are in a submodule or fork repository that you do not have push access to.

**Root Cause**: SDD artifacts were created in a directory that is a git submodule pointing to an upstream repository (not your fork).

**Solution**: Before dispatching any agent:

1. `cd` to the target directory
2. `git remote -v` -- confirm origin is your repository
3. `git rev-parse --show-superproject-working-tree` -- confirm not inside a submodule (should return empty)

Any check fails = stop, correct the path or re-propose the SDD change.

**Lesson**: SDD should be created inside a git repository that you can push to. Verify before proposing.

---

## iOS / Mobile

### PIT-012: iOS Safari Features Silently Fail Over HTTP

**Symptom**: On localhost (HTTP), `navigator.share()` throws `NotAllowedError` silently. `<a download>` does not trigger the save dialog.

**Root Cause**: iOS Safari requires HTTPS for these APIs. Over HTTP, they silently fail or are blocked.

**Solution**: These are not bugs. Deploying to a platform with HTTPS (e.g., Vercel) resolves both issues automatically.

**Lesson**: When testing on iOS, check whether the behavior requires HTTPS before debugging. If testing locally over HTTP, these failures are expected.
