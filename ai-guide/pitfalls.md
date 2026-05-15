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

---

## CI / Build Pipeline

### PIT-013: Electron + Next.js CI Breaks Native Modules

**Symptom**: `npm ci --ignore-scripts` in CI pipeline. Build fails with "Could not locate the bindings file" for better-sqlite3 (or any native module using prebuild/node-gyp).

**Root Cause**: `--ignore-scripts` blocks ALL postinstall scripts, including native module compilation (better-sqlite3 prebuild) and chromium download (puppeteer). You wanted to skip only chromium, but killed everything.

**Solution**: Remove `--ignore-scripts`. Use targeted environment variables instead:

```bash
PUPPETEER_SKIP_DOWNLOAD=true PUPPETEER_SKIP_CHROMIUM_DOWNLOAD=true npm ci
```

This skips only the chromium download while allowing native modules to compile normally.

**Lesson**: Be surgical with CI optimizations. Blanket `--ignore-scripts` breaks native dependencies. Use per-package skip flags instead.

---

## React / Frontend Patterns

### PIT-014: React Hook Callback Closure Trap

**Symptom**: Function called inside a hook's callback (`onConnect`, `onOpen`, `onMessage`) silently fails. No error in console. No crash. The function simply does nothing.

**Root Cause**: The callback captures a stale reference to the hook's return value. At the time the callback is created, the hook hasn't finished initializing. The closure holds the initial (empty/noop) version of the method.

```typescript
// BAD: sendMessage captured before hook initializes
const { sendMessage } = useWebSocket({
  onOpen: () => {
    sendMessage('hello')  // silently fails — stale closure
  }
})
```

**Solution**: Callback should only update state. Use `useEffect` to react to that state change:

```typescript
const [isConnected, setIsConnected] = useState(false)
const { sendMessage } = useWebSocket({
  onOpen: () => setIsConnected(true)  // only update state
})

useEffect(() => {
  if (isConnected) {
    sendMessage('hello')  // now sendMessage is current
  }
}, [isConnected, sendMessage])
```

**Lesson**: Never call hook-returned methods inside hook callbacks. The closure always captures the wrong value. State + useEffect is the safe pattern.

---

## Environment / Deployment

### PIT-015: vercel env add with echo Injects Newline

**Symptom**: API calls fail with mysterious 4xx errors. Environment variable looks correct in dashboard. But the actual stored value has a trailing `\n`.

**Root Cause**: `echo "value"` always appends a newline character. When piped into `vercel env add`, Vercel stores the newline as part of the value.

```bash
# BAD: stores "fish@example.com\n"
echo "fish@example.com" | vercel env add FROM_EMAIL production
```

**Solution**: Use `printf` (no trailing newline) and always verify:

```bash
# GOOD: stores "fish@example.com" exactly
printf "fish@example.com" > /tmp/v && vercel env add FROM_EMAIL production < /tmp/v

# VERIFY: pull and check
vercel env pull /tmp/.env.prod --environment=production
grep "^FROM_EMAIL=" /tmp/.env.prod
```

**Lesson**: `echo` always adds a newline. Never pipe `echo` into environment management tools. Use `printf` and verify with `env pull`.

---

## Database

### PIT-016: Postgres CREATE TABLE IF NOT EXISTS Does Not Upgrade Existing Tables

**Symptom**: Migration runs successfully (no errors). But new columns do not appear in the production table.

**Root Cause**: `CREATE TABLE IF NOT EXISTS` is a **no-op** when the table already exists. It does not add, remove, or modify columns. The migration file looks correct but does nothing.

**Solution**: Use explicit ALTER statements for schema evolution:

```sql
-- Adding columns
ALTER TABLE processed_emails ADD COLUMN IF NOT EXISTS category TEXT;

-- Modifying constraints
DO $$ BEGIN
  ALTER TABLE orders DROP CONSTRAINT IF EXISTS orders_status_check;
  ALTER TABLE orders ADD CONSTRAINT orders_status_check
    CHECK (status IN ('pending', 'processing', 'completed', 'cancelled'));
END $$;
```

**Verification**: Always check production schema directly before assuming migrations worked:

```bash
psql $DATABASE_URL -c "\d processed_emails"
```

**Lesson**: Migration files are not the source of truth for production schema. `CREATE TABLE IF NOT EXISTS` on an existing table is a no-op. Always verify with `\d <table>`.

---

## Desktop Application Security

### PIT-017: OS Keychain for Desktop App API Key Storage

**Context**: Desktop apps (Tauri, Electron) should never store API keys in `.env` files, `localStorage`, or plain config files. Unlike web apps, desktop app files are directly accessible on the user's filesystem.

**Pattern**: Use the operating system's secure credential storage:

| Platform | Backend | Rust Crate | Node.js Package |
|----------|---------|------------|-----------------|
| macOS | Keychain | `keyring` | `keytar` |
| Windows | Credential Manager | `keyring` | `keytar` |
| Linux | Secret Service (GNOME Keyring) | `keyring` | `keytar` |

**Implementation**:

```rust
// Rust (Tauri)
use keyring::Entry;

let entry = Entry::new("my-app", "api-credentials")?;
entry.set_password(&serde_json::to_string(&credentials)?)?;

// Retrieve
let stored = entry.get_password()?;
let credentials: ApiCredentials = serde_json::from_str(&stored)?;
```

```typescript
// Node.js (Electron)
import keytar from 'keytar'

await keytar.setPassword('my-app', 'api-credentials', JSON.stringify(credentials))

const stored = await keytar.getPassword('my-app', 'api-credentials')
const credentials = JSON.parse(stored)
```

**Lesson**: Browser security model does not apply to desktop apps. Files on disk are readable by any process. Use OS-level secure storage for secrets.
