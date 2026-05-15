# Development Environment: Three Modes

## Purpose

Define three development modes for desktop/hybrid applications (web frontend + native backend), their purposes, and when to use each. This eliminates the efficiency trap of rebuilding the entire application for every UI change.

## When to Use

- Setting up development workflow for any desktop application with a web frontend
- Deciding which mode to use for a specific task
- Onboarding new developers

---

## Three Modes Overview

| Mode | Command (example) | Wait Time | Best For |
|------|-------------------|-----------|----------|
| **Browser Mode** | `pnpm dev` | < 5 seconds | UI development, logic testing, daily primary mode |
| **Native Dev Mode** | `pnpm tauri dev` / `electron .` | 2-3 min (first time) | Testing IPC, native features, integration |
| **Full Build** | `pnpm tauri build` / `electron-builder` | ~3 minutes | Final acceptance, release |

---

## Mode 1: Browser Mode (Primary Development)

**What it does**: Runs the web frontend in a standard browser with a mock backend replacing native IPC calls.

**What you can do**:
- Develop UI components with hot reload (< 1 second feedback)
- Test form logic
- Test page flows and navigation
- Test state management
- Run unit and integration tests

**What you cannot do**:
- Test real database persistence (mock is in-memory)
- Test real file system operations (PDF export, file I/O)
- Test native API integrations (system tray, notifications, etc.)
- Test license/activation flows against real backend

**Architecture**:

```
Browser calls safeInvoke('list_items')
    |
safeInvoke checks environment
    --> not native + NODE_ENV=development
    |
dynamic import mockInvoke
    |
mockInvoke's switch-case finds 'list_items'
    |
MockStore returns in-memory data
    |
Component receives data, renders UI
```

**Key design**: All IPC calls go through a single entry point (`safeInvoke`). The mock branch is added in this one function. Business code (pages, components, hooks) does not change at all.

**Mock activation condition**:
```typescript
!isNativeEnv() && process.env.NODE_ENV === "development"
```

Production builds do not include mock code (tree-shaking removes it).

---

## Mode 2: Native Dev Mode (Integration Testing)

**What it does**: Runs the full application with both the web frontend and native backend, with hot reload for frontend changes.

**First launch**: Requires compiling the native backend (2-3 minutes). Subsequent frontend changes hot-reload in seconds. Backend changes trigger recompilation.

**What you can do**:
- Test real IPC communication
- Test database operations
- Test real file system operations
- Test native-only APIs
- Final verification before merging PRs

**When to use**:
- After implementing a new IPC command, verify the native backend works correctly
- Testing features that require file system or hardware access
- Pre-merge final verification

---

## Mode 3: Full Build (Release Only)

**What it does**: Produces the distributable application package (.app, .dmg, .msi, .exe).

**When to use**:
- Final acceptance before release
- Producing builds for testers or end users

**Do not** use this mode for iterative development. Every change requires a full rebuild (~3 minutes).

---

## The Efficiency Trap (Why Mock Backend Matters)

Without a mock backend, the development loop for desktop apps looks like this:

```
Change UI code
    |
Rebuild native app (~3 minutes)
    |
Open app, test
    |
Find bug, change code
    |
Rebuild (~3 minutes again)
    |
...
```

10 iterations = 30 minutes waiting. 30 iterations per day = 90 minutes (1.5 hours watching a progress bar).

Beyond time: waiting breaks flow state. Each wait forces context-switching, and rebuilding mental context after each wait costs additional minutes.

**With mock backend**:

```
Change UI code
    |
Browser hot-reload (< 1 second)
    |
Test immediately
    |
Find bug, change code
    |
Hot-reload again (< 1 second)
```

---

## Setting Up Mock Backend (Generalized)

### The Pattern

All native IPC calls go through a single entry point. The entry point has three branches:

```typescript
async function safeInvoke<T>(cmd: string, args?: Record<string, unknown>): Promise<T> {
  if (isNativeEnv()) {
    // Native environment: use real backend
    return nativeInvoke<T>(cmd, args)
  }

  if (process.env.NODE_ENV === "development") {
    // Dev + not native: use mock backend (dynamic import)
    const { mockInvoke } = await import("./mock-backend")
    return mockInvoke<T>(cmd, args)
  }

  // Production + not native: error
  throw new NotInNativeEnvError()
}
```

### MockStore Design

- **In-memory singleton**: Data persists during the session, resets on page reload
- **reset() method**: For test use -- restore all state to defaults between test cases
- **Sensible defaults**: Pre-populated with 2-3 sample records so the UI has something to render

### Why This Pattern

| Alternative | Why Rejected |
|-------------|-------------|
| MSW (Service Worker Mock) | IPC is not HTTP; cannot intercept native calls |
| Separate mock provider context | Invasive; requires modifying every component tree |
| Environment variable toggle | Condition is already sufficient (dev + not native); extra toggle adds complexity |

---

## Applying to Different Platforms

| Platform | Native Call Mechanism | Mock Strategy |
|----------|----------------------|---------------|
| Tauri (Rust + Web) | `invoke()` via IPC | Mock IPC handler |
| Electron (Node.js + Web) | `ipcRenderer.invoke()` | Mock IPC handler |
| React Native | Native modules | Mock native module responses |
| Apps requiring hardware | Hardware SDK calls | Mock SDK responses |
| Apps requiring cloud services | API calls | MSW or local mock server |
| Apps requiring databases | DB queries | In-memory database (SQLite `:memory:`) |

**General principle**: Reduce test environment complexity to the minimum. Let developers verify changes quickly without the full environment.

---

## When NOT to Use Mock

Mock is a tool, not a goal. Do not rely on mock for:

- Testing native backend logic (use real backend tests: `cargo test`, etc.)
- Testing rendered output quality (e.g., actual PDF rendering)
- Final acceptance of a release (use the full build)
- Data migration testing (must use real database)

**Mock boundary**: Mock is suitable for testing UI behavior. It is not suitable for testing system integration.

---

## Environment Variables

```bash
# .env.development
NEXT_PUBLIC_APP_ENV=development

# .env.production
NEXT_PUBLIC_APP_ENV=production
```

---

## Recommended Daily Workflow

1. Start `pnpm dev`, keep it running (hot reload)
2. Make UI changes, verify in browser
3. For new IPC logic: implement in mock backend first, test in browser
4. Switch to native dev mode to confirm backend integration
5. Only use full build for release

**Do not** iterate on UI in native dev or full build mode. The wait time is unnecessary cost.

---

## Common Issues

**Browser mode shows "requires desktop app"**

The mock backend is not activating. Check:
1. `process.env.NODE_ENV` is `"development"`
2. `isNativeEnv()` returns `false` in browser
3. Mock backend file exists
4. The bridge file has the dev mock branch

**Native dev mode fails to start**

Common causes:
- Native toolchain not installed (e.g., `rustup update`, `xcode-select --install`)
- Dependencies outdated (`cargo update` / `npm install`)

**Built app will not open (macOS)**

- Gatekeeper blocking unsigned app: right-click -> Open -> Allow
- Backend initialization failed: check app log directory

**CSS/JS blocked when testing from mobile device on LAN**

Next.js 16+ cross-origin protection blocks resources from LAN IPs. Add to config:
```typescript
const nextConfig = {
  experimental: {
    allowedDevOrigins: ['192.168.x.x'],  // replace with your LAN IP
  },
}
```
Restart dev server after changing.

**Non-ASCII characters in project path cause dev server to hang**

Vite / Next.js / Webpack dev servers cannot handle non-ASCII characters in file paths. The `@fs` URLs will contain encoded characters that time out. Solution: move the project to a path with only ASCII characters. Symlinks do not work (Node resolves to real path).
