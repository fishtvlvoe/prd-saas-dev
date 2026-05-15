# 開發環境三模式

這個檔案解決的問題：解釋 AIRE 的三種開發模式各自的用途、啟動方式、以及什麼時候用哪個。

---

## 三模式總覽

| 模式 | 指令 | 等待時間 | 適合做什麼 |
|------|------|---------|----------|
| **瀏覽器模式** | `pnpm dev` | < 5 秒 | UI 開發、邏輯測試、每日主力模式 |
| **Tauri 開發模式** | `pnpm tauri dev` | 2-3 分鐘（首次）| 測試 IPC 通訊、Tauri 特有功能 |
| **完整打包** | `pnpm tauri build` | ~3 分鐘 | 驗收最終版本、release 前確認 |

---

## 模式 1：瀏覽器模式（主力開發用）

**啟動**：
```bash
cd /Users/fishtv/Development/products/AIRE
pnpm dev
# → localhost:3000
```

**能做什麼**：
- 開發 UI 元件
- 測試表單邏輯
- 測試頁面流程（有 mock backend 後）
- 測試狀態管理
- 跑 Vitest 測試

**不能做什麼**：
- 測試真實的 SQLite 資料持久化（mock 是 in-memory）
- 測試真實的 PDF 生成（需要 Tauri 環境）
- 測試授權序號的真實驗證

**Mock Backend 架構**：

```
瀏覽器呼叫 safeInvoke('list_cases')
    ↓
safeInvoke 判斷環境
    → 非 Tauri + NODE_ENV=development
    ↓
dynamic import mockInvoke
    ↓
mockInvoke 的 switch-case 找到 'list_cases'
    ↓
MockStore.instance.cases 的 Map 轉 Array 回傳
    ↓
頁面元件收到資料，渲染 UI
```

**MockStore 預設狀態**：
```typescript
license:      { status: "none", serialKey: null }
cases:        Map（2 筆預設案件）
drafts:       Map（空）
brandSettings: { companyName: "測試不動產", ... }
logs:         LogEntry[]（5 筆預設記錄）
logo:         null
clauses:      Map（3 條預設法條）
```

重新整理頁面 → MockStore 狀態重設（回到上面的預設值）。

---

## 模式 2：Tauri 開發模式（整合測試用）

**啟動**：
```bash
pnpm tauri dev
```

**第一次啟動**：需要編譯 Rust 後端，約 2-3 分鐘。之後的改動會熱重載（前端改動幾秒，Rust 改動重新編譯）。

**能做什麼**：
- 測試真實的 IPC 通訊
- 測試 SQLite 資料操作
- 測試真實的授權流程
- 測試 PDF 生成
- 確認 Tauri 特有的 API 正常運作

**使用時機**：
- 實作了新的 IPC command 之後，要驗證 Rust 後端正確
- 測試需要檔案系統操作的功能
- 在 PR 合併前的最終驗證

---

## 模式 3：完整打包（Release 前用）

**啟動**：
```bash
pnpm tauri build
```

**等待時間**：約 3 分鐘（完整 Rust 編譯 + 前端 build + 打包）

**輸出**：
- macOS：`src-tauri/target/release/bundle/macos/AIRE.app` 和 `.dmg`
- Windows：`src-tauri/target/release/bundle/msi/*.msi`

**使用時機**：
- Release 前的最終驗收
- 給測試者 / 使用者的版本

**打包前必做**（見 `what/checklists/release-checklist.md`）：
- 版本號已更新
- icon 已用 `pnpm tauri icon` 重新生成
- 所有測試通過
- 授權邏輯在 prod 環境下正常

---

## 環境變數

```bash
# .env.development（開發用）
NEXT_PUBLIC_APP_ENV=development

# .env.production（打包用）
NEXT_PUBLIC_APP_ENV=production
```

Mock backend 的啟用條件：
```typescript
!isTauriEnv() && process.env.NODE_ENV === "development"
```

Production build 完全不包含 mock 代碼（tree-shaking 會移除）。

---

## 常見問題排查

**問題：pnpm dev 後頁面出現「需在 AIRE 桌面 App 中使用」**

原因：mock backend 沒有啟用，或 `TauriRequired` 元件沒有更新。

排查步驟：
1. 確認 `process.env.NODE_ENV === "development"`（應該是）
2. 確認 `isTauriEnv()` 回傳 false（瀏覽器環境應該是）
3. 確認 `src/lib/mock-backend.ts` 存在
4. 確認 `src/lib/tauri-bridge.ts` 有加 dev mock 分支

---

**問題：pnpm tauri dev 跑不起來**

常見原因：
- Rust 工具鏈沒安裝：`rustup update`
- macOS 需要 Xcode Command Line Tools：`xcode-select --install`
- 依賴沒更新：`cargo update`

---

**問題：打包後 App 無法開啟**

常見原因：
- macOS Gatekeeper 阻擋未簽名 App：右鍵 → 打開 → 允許
- 授權邏輯在 prod 環境有問題：檢查 serial key 驗證流程
- SQLite 初始化失敗：查 App 的 log 目錄

---

## 開發效率建議

**日常開發的工作流**：
1. `pnpm dev` 開著，邊改邊看（熱重載）
2. UI 改動在瀏覽器確認
3. 新的 IPC 邏輯：先在 mock backend 實作，瀏覽器測試 OK 後
4. 切換到 `pnpm tauri dev` 確認 Rust 後端整合正確
5. 只在 release 前用 `pnpm tauri build`

不要在 `pnpm tauri dev` 或 `pnpm tauri build` 模式下做大量 UI 迭代——每次等待 2-3 分鐘是不必要的代價。
