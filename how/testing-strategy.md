# 測試策略

這個檔案解決的問題：定義 AIRE 的三層測試架構，以及 mock backend 在測試中扮演的角色。

---

## 三層測試架構

```
Layer 1：單元測試（Unit）
  工具：Vitest（前端）+ cargo test（Rust 後端）
  速度：毫秒級
  覆蓋：單個函數、模組、元件

Layer 2：整合測試（Integration）
  工具：Vitest + mock backend
  速度：秒級
  覆蓋：多個模組的互動、IPC 通訊模擬

Layer 3：E2E 測試（End-to-End）
  工具：Tauri E2E（e2e_smoke.rs）
  速度：分鐘級（需要完整 App）
  覆蓋：完整使用者流程
```

---

## 各層測試的職責

### Layer 1：單元測試

**前端（Vitest）測試的重點**：
- `mock-backend.ts` 的每個 command handler
- `tauri-bridge.ts` 的三個分支（Tauri / dev mock / prod error）
- UI 元件的 props 和 state 邏輯
- 工具函數（日期格式化、條款搜尋等）

**Rust 後端（cargo test）測試的重點**：
- 每個 Tauri command 的業務邏輯
- SQLite 操作（CRUD）
- 授權驗證邏輯

**單元測試的邊界**：
- 不測試 UI 渲染（那是整合測試的範圍）
- 不測試 IPC 通訊（那是整合測試的範圍）
- 不依賴外部服務或檔案系統

---

### Layer 2：整合測試

**主要工具：mock backend + Vitest**

Mock backend 在整合測試中的角色：
- 讓測試在純 Node.js 環境跑（不需要 Tauri）
- 讓測試速度從「分鐘」降到「秒」
- 讓每個測試的初始狀態可以重設（`MockStore.reset()`）

**關鍵整合測試案例**：

```typescript
// tauri-bridge 三個分支
test('dev mock 環境 dispatch 到 mockInvoke', async () => {
  // 設定：NODE_ENV=development，非 Tauri 環境
  // 執行：safeInvoke('list_cases')
  // 驗證：mockInvoke 被呼叫，回傳 mock 資料
})

test('Tauri 環境走真實 invoke', async () => {
  // 設定：Tauri 環境（mock isTauriEnv = true）
  // 執行：safeInvoke('list_cases')
  // 驗證：tauri invoke 被呼叫
})

test('prod 非 Tauri 環境拋錯', async () => {
  // 設定：NODE_ENV=production，非 Tauri 環境
  // 執行：safeInvoke('list_cases')
  // 驗證：拋出 NotInTauriError
})

// MockStore 狀態管理
test('reset() 後狀態回到初始值', async () => {
  // 執行一些操作改變狀態
  MockStore.instance.reset()
  // 驗證所有欄位回到預設值
})

// 每個 command 的 happy path
test('list_cases 回傳預設 2 筆案件', async () => {
  const result = await mockInvoke('list_cases')
  expect(result).toHaveLength(2)
})
```

---

### Layer 3：E2E 測試

**工具：`src-tauri/tests/e2e_smoke.rs`**

E2E 測試的目的：
- 驗證完整的 Tauri App 能夠啟動
- 驗證關鍵使用者流程（授權啟動 → 進入 dashboard → 看到案件列表）
- 驗證 IPC 通訊在真實環境下正常運作

**E2E 測試的執行時機**：
- 不在每次 commit 都跑（太慢）
- 在 PR 合併前跑
- 在 release 前跑

---

## 瀏覽器測試 vs Tauri 測試的分工

| 測試類型 | 工具 | 環境 | 適合測試 |
|---------|------|------|---------|
| UI 行為測試 | Vitest + mock backend | 瀏覽器（localhost:3000）| 元件邏輯、表單驗證、狀態管理 |
| IPC mock 測試 | Vitest + mock backend | 純 Node.js | tauri-bridge 分支、MockStore 行為 |
| Rust 後端測試 | cargo test | Rust 環境 | command 業務邏輯、SQLite 操作 |
| E2E 測試 | e2e_smoke.rs | 完整 Tauri App | 完整使用者流程 |

---

## 測試執行指令

```bash
# 前端單元 + 整合測試（快，幾秒）
pnpm test

# 前端測試（watch 模式，開發時用）
pnpm test --watch

# Rust 後端測試
cd src-tauri && cargo test

# 所有測試（CI 用）
pnpm test && cd src-tauri && cargo test
```

---

## TDD 在這個架構中的位置

TDD 的失敗矩陣對應到 Layer 1 和 Layer 2：

```
失敗矩陣（設計階段）
    ↓
Layer 1 紅燈測試（單元測試，單個函數）
Layer 2 紅燈測試（整合測試，多模組互動）
    ↓
寫實作（讓紅燈變綠燈）
    ↓
Layer 3（E2E，驗收最終結果）
```

---

## 測試品質標準

**不看測試覆蓋率（原因見 why-tdd.md）**，看這些指標：

1. **失敗矩陣覆蓋**：每個失敗情境都有對應的測試嗎？
2. **獨立性**：每個測試能獨立跑嗎（不依賴其他測試的執行順序）？
3. **可讀性**：測試名稱能清楚說明它在測什麼嗎？
4. **速度**：Layer 1 + Layer 2 總共跑完在 30 秒以內嗎？
5. **穩定性**：測試不是 flaky 的（同樣的代碼，10 次跑出同樣結果）嗎？
