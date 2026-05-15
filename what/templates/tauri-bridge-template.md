# tauri-bridge.ts 通用模板

這個檔案解決的問題：提供一個可以直接複製到新 Tauri + Next.js 專案的 `tauri-bridge.ts` 模板，包含 dev mock 分支機制。

---

## 使用前提

1. 專案是 Tauri + Next.js（或 Tauri + Vite + React）
2. 所有 IPC 呼叫都要統一通過 `safeInvoke`（不直接呼叫 `invoke`）
3. 有 `src/lib/mock-backend.ts`（見下方）

---

## 主要檔案：src/lib/tauri-bridge.ts

```typescript
/**
 * Tauri IPC 統一入口
 *
 * 環境判斷：
 * - Tauri 環境 → 走真實 invoke
 * - dev + 非 Tauri → 走 mockInvoke（mock-backend.ts）
 * - prod + 非 Tauri → 拋 NotInTauriError
 */

import { invoke } from "@tauri-apps/api/core"

// 判斷是否在 Tauri 環境
function isTauriEnv(): boolean {
  return typeof window !== "undefined" && "__TAURI_INTERNALS__" in window
}

// 自定義錯誤類型
export class NotInTauriError extends Error {
  constructor() {
    super("此功能需在桌面 App 中使用")
    this.name = "NotInTauriError"
  }
}

/**
 * 統一 IPC 入口
 * 所有頁面元件和 hooks 都通過這個函數呼叫 Tauri commands
 */
export async function safeInvoke<T>(
  cmd: string,
  args?: Record<string, unknown>
): Promise<T> {
  if (isTauriEnv()) {
    // Tauri 環境：走真實 invoke
    return invoke<T>(cmd, args)
  }

  if (process.env.NODE_ENV === "development") {
    // dev 環境 + 非 Tauri：走 mock backend
    // dynamic import 確保 production build 不包含 mock 代碼
    const { mockInvoke } = await import("./mock-backend")
    return mockInvoke<T>(cmd, args)
  }

  // production 環境 + 非 Tauri：拋錯
  throw new NotInTauriError()
}
```

---

## Mock Backend 框架：src/lib/mock-backend.ts

```typescript
/**
 * Dev 環境 mock backend
 *
 * 只在 development 環境 + 非 Tauri 時使用
 * In-memory 存儲，頁面重新整理後重設
 */

// ===== 型別定義（根據你的 IPC 命令調整）=====

interface LicenseStatus {
  status: "none" | "valid" | "expired"
  serialKey: string | null
}

interface CaseRow {
  id: string
  title: string
  address: string
  status: "draft" | "completed"
  createdAt: string
  updatedAt: string
}

interface LogEntry {
  id: string
  action: string
  timestamp: string
}

interface BrandingData {
  companyName: string
  logoPath: string | null
  theme: string
}

interface ClauseData {
  key: string
  title: string
  content: string
}

// ===== MockStore =====

class MockStore {
  private static _instance: MockStore | null = null

  // 授權狀態
  license: LicenseStatus = { status: "none", serialKey: null }

  // 案件（預設 2 筆）
  cases: Map<string, CaseRow> = new Map([
    [
      "case-001",
      {
        id: "case-001",
        title: "信義區大樓 - 王先生",
        address: "台北市信義區信義路五段",
        status: "draft",
        createdAt: "2024-01-15T10:00:00Z",
        updatedAt: "2024-01-15T10:00:00Z",
      },
    ],
    [
      "case-002",
      {
        id: "case-002",
        title: "大安區公寓 - 陳小姐",
        address: "台北市大安區敦化南路一段",
        status: "completed",
        createdAt: "2024-01-10T09:00:00Z",
        updatedAt: "2024-01-12T14:30:00Z",
      },
    ],
  ])

  // 草稿
  drafts: Map<string, unknown> = new Map()

  // 品牌設定
  brandSettings: BrandingData = {
    companyName: "測試不動產",
    logoPath: null,
    theme: "professional",
  }

  // 操作日誌（預設 5 筆）
  logs: LogEntry[] = [
    { id: "log-001", action: "建立案件：信義區大樓", timestamp: "2024-01-15T10:00:00Z" },
    { id: "log-002", action: "儲存草稿：信義區大樓", timestamp: "2024-01-15T10:05:00Z" },
    { id: "log-003", action: "建立案件：大安區公寓", timestamp: "2024-01-10T09:00:00Z" },
    { id: "log-004", action: "完成案件：大安區公寓", timestamp: "2024-01-12T14:30:00Z" },
    { id: "log-005", action: "匯出 PDF：大安區公寓", timestamp: "2024-01-12T14:35:00Z" },
  ]

  // Logo
  logo: Uint8Array | null = null

  // 法條（預設 3 條）
  clauses: Map<string, ClauseData> = new Map([
    ["clause-001", { key: "clause-001", title: "第一條", content: "（示範法條內容）" }],
    ["clause-002", { key: "clause-002", title: "第二條", content: "（示範法條內容）" }],
    ["clause-003", { key: "clause-003", title: "第三條", content: "（示範法條內容）" }],
  ])

  static get instance(): MockStore {
    if (!MockStore._instance) {
      MockStore._instance = new MockStore()
    }
    return MockStore._instance
  }

  // 重設狀態（供測試使用）
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

    // Cases
    case "list_cases":
      return Array.from(store.cases.values()) as unknown as T

    case "get_case": {
      const id = args?.id as string
      const c = store.cases.get(id)
      if (!c) throw new Error(`Case not found: ${id}`)
      return c as unknown as T
    }

    case "create_case": {
      const id = `case-${Date.now()}`
      const newCase: CaseRow = {
        id,
        ...(args?.data as Omit<CaseRow, "id" | "createdAt" | "updatedAt">),
        status: "draft",
        createdAt: new Date().toISOString(),
        updatedAt: new Date().toISOString(),
      }
      store.cases.set(id, newCase)
      return newCase as unknown as T
    }

    case "update_case": {
      const id = args?.id as string
      const existing = store.cases.get(id)
      if (!existing) throw new Error(`Case not found: ${id}`)
      const updated = {
        ...existing,
        ...(args?.data as Partial<CaseRow>),
        updatedAt: new Date().toISOString(),
      }
      store.cases.set(id, updated)
      return updated as unknown as T
    }

    case "delete_case": {
      store.cases.delete(args?.id as string)
      return { success: true } as unknown as T
    }

    case "mark_completed": {
      const id = args?.id as string
      const c = store.cases.get(id)
      if (c) {
        c.status = "completed"
        c.updatedAt = new Date().toISOString()
      }
      return { success: true } as unknown as T
    }

    // PDF
    case "export_pdf":
      return { success: true, path: args?.output_path } as unknown as T

    // Drafts
    case "save_draft":
      store.drafts.set(args?.case_id as string, args?.content)
      return { success: true } as unknown as T

    case "load_draft":
      return (store.drafts.get(args?.case_id as string) ?? null) as unknown as T

    // Log
    case "list_recent_logs": {
      const limit = (args?.limit as number) ?? 10
      return store.logs.slice(0, limit) as unknown as T
    }

    // Branding
    case "get_brand_settings":
      return store.brandSettings as unknown as T

    case "save_brand_settings":
      store.brandSettings = { ...store.brandSettings, ...(args?.data as Partial<BrandingData>) }
      return { success: true } as unknown as T

    case "upload_logo":
      store.logo = args?.file as Uint8Array
      return { success: true } as unknown as T

    case "get_logo":
      return store.logo as unknown as T

    case "list_themes":
      return ["professional", "minimal"] as unknown as T

    // Legal
    case "get_clause": {
      const key = args?.key as string
      const clause = store.clauses.get(key)
      if (!clause) throw new Error(`Clause not found: ${key}`)
      return clause as unknown as T
    }

    case "list_clauses":
      return Array.from(store.clauses.values()) as unknown as T

    case "sync_clauses":
      return { success: true, synced: store.clauses.size } as unknown as T

    default:
      throw new Error(`Mock not implemented: ${cmd}`)
  }
}

// 匯出 MockStore 供測試使用
export { MockStore }
```

---

## 使用說明

### 在頁面元件中使用

```typescript
import { safeInvoke } from "@/lib/tauri-bridge"

// 直接呼叫，不需要判斷環境
const cases = await safeInvoke<CaseRow[]>('list_cases')
```

### 在測試中重設狀態

```typescript
import { MockStore } from "@/lib/mock-backend"

beforeEach(() => {
  MockStore.instance.reset()
})
```

### 新增 mock command

在 `mockInvoke` 的 `switch` 加一個 `case`：

```typescript
case "my_new_command": {
  // 處理邏輯
  return result as unknown as T
}
```

記得同時在 `design.md` 更新 Command 清單，並新增對應的測試。

---

## 調整說明

這個模板以 AIRE 的命令為例。移植到新專案時，需要調整：

1. **型別定義**：根據你的 IPC 命令定義對應的型別
2. **MockStore 預設狀態**：根據你的業務邏輯設定有意義的預設值
3. **switch-case**：根據你的命令清單新增 case
4. **測試**：根據失敗矩陣寫對應的單元測試
