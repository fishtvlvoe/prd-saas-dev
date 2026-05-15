# Native Bridge Template (IPC with Dev Mock Support)

## Purpose

Provide a copy-paste-ready IPC bridge template for desktop applications with web frontends. Includes the dev mock branch mechanism for browser-mode development.

## When to Use

- Starting a new desktop application project (Tauri, Electron, or similar)
- Adding mock backend support to an existing desktop project
- Reference when implementing the three-branch environment routing pattern

---

## Prerequisites

1. Project uses a native backend (Tauri/Rust, Electron/Node.js, etc.) with a web frontend
2. All IPC calls will go through `safeInvoke` (never call native invoke directly)
3. A `mock-backend.ts` file exists (template below)

---

## Main File: src/lib/native-bridge.ts

```typescript
/**
 * Native IPC unified entry point
 *
 * Environment routing:
 * - Native environment -> real invoke (native backend)
 * - dev + not native -> mockInvoke (mock-backend.ts)
 * - prod + not native -> throw NotInNativeEnvError
 */

import { invoke } from "@tauri-apps/api/core"
// For Electron: import { ipcRenderer } from 'electron'

// Detect native environment
function isNativeEnv(): boolean {
  // Tauri detection
  return typeof window !== "undefined" && "__TAURI_INTERNALS__" in window
  // Electron detection alternative:
  // return typeof window !== "undefined" && "electronAPI" in window
}

// Custom error type
export class NotInNativeEnvError extends Error {
  constructor() {
    super("This feature requires the desktop application")
    this.name = "NotInNativeEnvError"
  }
}

/**
 * Unified IPC entry point
 * All page components and hooks call native commands through this function
 */
export async function safeInvoke<T>(
  cmd: string,
  args?: Record<string, unknown>
): Promise<T> {
  if (isNativeEnv()) {
    // Native environment: use real invoke
    return invoke<T>(cmd, args)
  }

  if (process.env.NODE_ENV === "development") {
    // Dev environment + not native: use mock backend
    // Dynamic import ensures production build excludes mock code
    const { mockInvoke } = await import("./mock-backend")
    return mockInvoke<T>(cmd, args)
  }

  // Production environment + not native: throw error
  throw new NotInNativeEnvError()
}
```

---

## Mock Backend Framework: src/lib/mock-backend.ts

```typescript
/**
 * Dev environment mock backend
 *
 * Only used in development + non-native environment
 * In-memory storage, resets on page reload
 */

// ===== Type definitions (adjust for your IPC commands) =====

interface LicenseStatus {
  status: "none" | "valid" | "expired"
  serialKey: string | null
}

interface ItemRow {
  id: string
  title: string
  description: string
  status: "draft" | "completed"
  createdAt: string
  updatedAt: string
}

interface LogEntry {
  id: string
  action: string
  timestamp: string
}

interface SettingsData {
  companyName: string
  logoPath: string | null
  theme: string
}

// ===== MockStore =====

class MockStore {
  private static _instance: MockStore | null = null

  // License state
  license: LicenseStatus = { status: "none", serialKey: null }

  // Items (default 2 records)
  items: Map<string, ItemRow> = new Map([
    [
      "item-001",
      {
        id: "item-001",
        title: "Sample Item A",
        description: "First sample record",
        status: "draft",
        createdAt: "2024-01-15T10:00:00Z",
        updatedAt: "2024-01-15T10:00:00Z",
      },
    ],
    [
      "item-002",
      {
        id: "item-002",
        title: "Sample Item B",
        description: "Second sample record",
        status: "completed",
        createdAt: "2024-01-10T09:00:00Z",
        updatedAt: "2024-01-12T14:30:00Z",
      },
    ],
  ])

  // Drafts
  drafts: Map<string, unknown> = new Map()

  // Settings
  settings: SettingsData = {
    companyName: "Test Company",
    logoPath: null,
    theme: "professional",
  }

  // Activity log (default 5 entries)
  logs: LogEntry[] = [
    { id: "log-001", action: "Created item: Sample Item A", timestamp: "2024-01-15T10:00:00Z" },
    { id: "log-002", action: "Saved draft: Sample Item A", timestamp: "2024-01-15T10:05:00Z" },
    { id: "log-003", action: "Created item: Sample Item B", timestamp: "2024-01-10T09:00:00Z" },
    { id: "log-004", action: "Completed item: Sample Item B", timestamp: "2024-01-12T14:30:00Z" },
    { id: "log-005", action: "Exported PDF: Sample Item B", timestamp: "2024-01-12T14:35:00Z" },
  ]

  // Logo binary
  logo: Uint8Array | null = null

  static get instance(): MockStore {
    if (!MockStore._instance) {
      MockStore._instance = new MockStore()
    }
    return MockStore._instance
  }

  // Reset state (for test use)
  reset(): void {
    MockStore._instance = new MockStore()
  }
}

// ===== mockInvoke =====

export async function mockInvoke<T>(
  cmd: string,
  args?: Record<string, unknown>
): Promise<T> {
  const store = MockStore.instance

  switch (cmd) {
    // License
    case "get_license_status":
      return store.license as unknown as T

    case "activate_license": {
      store.license = { status: "valid", serialKey: args?.serial_key as string }
      return { success: true } as unknown as T
    }

    case "deactivate_license": {
      store.license = { status: "none", serialKey: null }
      return { success: true } as unknown as T
    }

    case "check_license":
      return { valid: store.license.status === "valid" } as unknown as T

    // Items
    case "list_items":
      return Array.from(store.items.values()) as unknown as T

    case "get_item": {
      const id = args?.id as string
      const item = store.items.get(id)
      if (!item) throw new Error(`Item not found: ${id}`)
      return item as unknown as T
    }

    case "create_item": {
      const id = `item-${Date.now()}`
      const newItem: ItemRow = {
        id,
        ...(args?.data as Omit<ItemRow, "id" | "createdAt" | "updatedAt">),
        status: "draft",
        createdAt: new Date().toISOString(),
        updatedAt: new Date().toISOString(),
      }
      store.items.set(id, newItem)
      return newItem as unknown as T
    }

    case "update_item": {
      const id = args?.id as string
      const existing = store.items.get(id)
      if (!existing) throw new Error(`Item not found: ${id}`)
      const updated = {
        ...existing,
        ...(args?.data as Partial<ItemRow>),
        updatedAt: new Date().toISOString(),
      }
      store.items.set(id, updated)
      return updated as unknown as T
    }

    case "delete_item": {
      store.items.delete(args?.id as string)
      return { success: true } as unknown as T
    }

    case "mark_completed": {
      const id = args?.id as string
      const item = store.items.get(id)
      if (item) {
        item.status = "completed"
        item.updatedAt = new Date().toISOString()
      }
      return { success: true } as unknown as T
    }

    // Export
    case "export_pdf":
      return { success: true, path: args?.output_path } as unknown as T

    // Drafts
    case "save_draft":
      store.drafts.set(args?.item_id as string, args?.content)
      return { success: true } as unknown as T

    case "load_draft":
      return (store.drafts.get(args?.item_id as string) ?? null) as unknown as T

    // Log
    case "list_recent_logs": {
      const limit = (args?.limit as number) ?? 10
      return store.logs.slice(0, limit) as unknown as T
    }

    // Settings
    case "get_settings":
      return store.settings as unknown as T

    case "save_settings":
      store.settings = { ...store.settings, ...(args?.data as Partial<SettingsData>) }
      return { success: true } as unknown as T

    case "upload_logo":
      store.logo = args?.file as Uint8Array
      return { success: true } as unknown as T

    case "get_logo":
      return store.logo as unknown as T

    default:
      throw new Error(`Mock not implemented: ${cmd}`)
  }
}

// Export MockStore for test use
export { MockStore }
```

---

## Usage

### In Page Components

```typescript
import { safeInvoke } from "@/lib/native-bridge"

// Call directly -- no environment checks needed
const items = await safeInvoke<ItemRow[]>('list_items')
```

### In Tests (Reset State Between Tests)

```typescript
import { MockStore } from "@/lib/mock-backend"

beforeEach(() => {
  MockStore.instance.reset()
})
```

### Adding a New Mock Command

Add a new `case` to the `mockInvoke` switch statement:

```typescript
case "my_new_command": {
  // Handle logic
  return result as unknown as T
}
```

Also update: design.md command list, and add corresponding tests.

---

## Adaptation Guide

When porting to a new project:

1. **Type definitions**: Define types matching your IPC commands
2. **MockStore defaults**: Set meaningful default data for your domain
3. **switch-case**: Add cases for each of your commands
4. **Tests**: Write tests based on your failure matrix
5. **Environment detection**: Adjust `isNativeEnv()` for your platform (Tauri, Electron, etc.)
