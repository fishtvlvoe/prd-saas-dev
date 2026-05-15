# AIRE 技術架構

這個檔案解決的問題：提供 AIRE 完整的技術架構參考，讓任何接手的開發者（或 AI 協作者）能快速理解系統的各個層次。

---

## 技術棧總覽

| 層級 | 技術 | 版本 / 說明 |
|------|------|------------|
| 桌面框架 | Tauri 2.x | Rust 後端 + Web 前端，跨平台（macOS + Windows）|
| 前端框架 | Next.js 16 + React 19 | App Router，TypeScript |
| 樣式 | Tailwind CSS + shadcn/ui | 16+ 原子元件，統一設計系統 |
| 圖示 | Lucide React | 統一圖示風格（禁止 Emoji）|
| PDF 渲染 | @react-pdf/renderer | 聲明式 React API，支援中文字型 |
| 後端語言 | Rust | Tauri 後端，IPC command 實作 |
| 資料庫 | SQLite（本地）| 透過 rusqlite，嵌入式，離線可用 |
| 前後端通訊 | Tauri Commands（IPC）| `invoke(cmd, args)` |
| 測試（前端）| Vitest | 單元 + 整合測試 |
| 測試（後端）| cargo test | Rust 單元 + E2E |

---

## 系統架構圖

```
使用者（不動產經紀人）
    ↓
[Tauri App Shell]
    ↓ Webview
[Next.js 前端（React 19）]
    ├── pages/activation          — 授權啟動流程
    ├── pages/(dashboard)/cases   — 案件管理
    ├── pages/(dashboard)/settings — 品牌設定、日誌
    └── components/pdf-blocks     — PDF 區塊元件
    ↓ safeInvoke（tauri-bridge.ts）
[IPC 層（Tauri Commands）]
    ├── dev mock 環境 → mockInvoke（mock-backend.ts）
    └── Tauri 環境 → invoke（Rust backend）
    ↓
[Rust 後端（src-tauri/）]
    ├── commands/license.rs  — 授權管理
    ├── commands/cases.rs    — 案件 CRUD
    ├── commands/pdf.rs      — PDF 輸出
    ├── commands/drafts.rs   — 草稿儲存
    ├── commands/log.rs      — 操作日誌
    ├── commands/branding.rs — 品牌設定
    └── legal_clauses/       — 法條快取
    ↓
[SQLite（本地檔案）]
```

---

## IPC 命令清冊（22 個命令，7 類）

### license（4 個）— 授權管理

| 命令 | 輸入 | 輸出 | 說明 |
|------|------|------|------|
| `get_license_status` | — | `LicenseStatus` | 查詢目前授權狀態 |
| `activate_license` | `serial_key: String` | `Result` | 輸入序號啟動 |
| `deactivate_license` | — | `Result` | 停用授權 |
| `check_license` | — | `Result` | 驗證授權有效性 |

### cases（6 個）— 案件管理

| 命令 | 輸入 | 輸出 | 說明 |
|------|------|------|------|
| `list_cases` | — | `Case[]` | 列出所有案件 |
| `get_case` | `id: String` | `Case` | 取得單一案件 |
| `create_case` | `data: CaseData` | `Case` | 建立新案件 |
| `update_case` | `id: String, data: CaseData` | `Case` | 更新案件 |
| `delete_case` | `id: String` | `Result` | 刪除案件 |
| `mark_completed` | `id: String` | `Result` | 標記案件完成 |

### pdf（1 個）— PDF 輸出

| 命令 | 輸入 | 輸出 | 說明 |
|------|------|------|------|
| `export_pdf` | `pdfBytes: Bytes, outputPath: String` | `Result` | 輸出 PDF 到桌面 |

### drafts（2 個）— 草稿管理

| 命令 | 輸入 | 輸出 | 說明 |
|------|------|------|------|
| `save_draft` | `case_id: String, content: JSON` | `Result` | 儲存草稿 |
| `load_draft` | `case_id: String` | `JSON` | 載入草稿 |

### log（1 個）— 操作日誌

| 命令 | 輸入 | 輸出 | 說明 |
|------|------|------|------|
| `list_recent_logs` | `limit: Int` | `LogEntry[]` | 取得最近的操作紀錄 |

### branding（5 個）— 品牌設定

| 命令 | 輸入 | 輸出 | 說明 |
|------|------|------|------|
| `get_brand_settings` | — | `BrandingData` | 取得品牌設定 |
| `save_brand_settings` | `data: BrandingData` | `Result` | 儲存品牌設定 |
| `upload_logo` | `file: Bytes` | `Result` | 上傳公司 logo |
| `get_logo` | — | `Bytes` | 取得 logo 圖片 |
| `list_themes` | — | `String[]` | 列出可用主題 |

### legal（3 個）— 法條管理

| 命令 | 輸入 | 輸出 | 說明 |
|------|------|------|------|
| `get_clause` | `key: String` | `ClauseData` | 取得單一法條 |
| `list_clauses` | — | `ClauseData[]` | 列出所有法條 |
| `sync_clauses` | — | `Result` | 同步最新法條 |

---

## 前後端通訊模式

**統一入口點**：`src/lib/tauri-bridge.ts` 的 `safeInvoke`

```typescript
// 所有 IPC 呼叫都走這個函數，不直接呼叫 invoke
async function safeInvoke<T>(cmd: string, args?: Record<string, unknown>): Promise<T>
```

**三個分支**：

```
isTauriEnv()
    true  → tauri.invoke(cmd, args)
    false → process.env.NODE_ENV === "development"
                true  → mockInvoke(cmd, args)
                false → throw new NotInTauriError()
```

---

## 專案目錄結構

```
AIRE/
├── src/                         — Next.js 前端
│   ├── app/
│   │   ├── (dashboard)/
│   │   │   ├── cases/           — 案件管理頁面
│   │   │   └── settings/        — 品牌設定、日誌頁面
│   │   └── activation/          — 授權啟動流程
│   ├── lib/
│   │   ├── tauri-bridge.ts      — IPC 統一入口（safeInvoke）
│   │   ├── mock-backend.ts      — Dev 環境 mock handler + MockStore
│   │   ├── pdf-blocks/          — PDF 區塊元件庫
│   │   ├── pdf-engine/          — PDF 渲染引擎核心
│   │   └── pdf-themes/          — 主題定義（淡雅 / 專業）
│   ├── components/ui/           — shadcn/ui 原子元件（16+）
│   └── hooks/
│       └── useLicenseStatus.ts  — 授權狀態管理 hook
├── src-tauri/                   — Rust 後端
│   ├── src/commands/            — IPC command 實作
│   │   ├── cases.rs
│   │   ├── license.rs
│   │   ├── pdf.rs
│   │   ├── drafts.rs
│   │   ├── log.rs
│   │   └── branding.rs
│   ├── src/db/                  — SQLite 資料層
│   ├── src/legal_clauses/       — 法條快取與同步
│   └── tests/e2e_smoke.rs       — E2E 整合測試
└── openspec/                    — SDD 規格文件
    ├── specs/                   — 功能規格
    └── changes/                 — 變更提案
```

---

## 關鍵設計決策（摘要）

| 決策 | 選擇 | 關鍵理由 |
|------|------|---------|
| 桌面 App 框架 | Tauri 2.x | Bundle 5MB vs Electron 150MB；Rust 安全性 |
| 前端框架 | Next.js 16 | 完整框架，未來 web 版可複用 |
| 資料庫 | SQLite | 嵌入式，零設定，離線可用 |
| PDF 渲染 | @react-pdf | React 聲明式 API，中文字型支援 |
| IPC 入口 | safeInvoke 統一 | 單一入口方便 mock / 測試 / 錯誤處理 |
| Mock backend | in-memory MockStore | dev 環境不依賴 Rust，迭代速度快 |

完整決策理由：`how/decision-tree.md`

---

## 系統需求

| 項目 | 規格 |
|------|------|
| 作業系統 | macOS 12.0+（Monterey）或 Windows 10+ |
| 記憶體 | 4GB RAM 以上 |
| 磁碟空間 | 500MB 以上 |
| 網路 | 法條同步時需要；其他功能離線可用 |
