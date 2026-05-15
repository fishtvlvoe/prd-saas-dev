# 為什麼要消除打包等待（Mock Backend 的動機）

這個檔案解決的問題：解釋 mock backend 這個看起來像「額外工程」的東西，實際上是直接解決開發效率瓶頸的熵減實踐。

---

## 問題：Tauri App 的開發效率陷阱

AIRE 是 Tauri 桌面 App，前端是 Next.js，後端是 Rust。

開發迭代的傳統流程：

```
改 UI 代碼
    ↓
pnpm tauri build（Rust 編譯 + 打包，約 3 分鐘）
    ↓
開啟 .app 測試
    ↓
發現 bug → 改代碼
    ↓
pnpm tauri build（再等 3 分鐘）
    ↓
...
```

每次改一個按鈕的顏色，要等 3 分鐘。

這個等待時間在開發初期看起來不嚴重，但實際上：
- 10 次迭代 = 30 分鐘等待
- 每天 30 次迭代 = 90 分鐘等待（1.5 小時，在等 Rust 編譯）
- 一個月 = 45 小時等待

而且不只是時間的問題：等待會打斷思維流，每次從「我剛才改了什麼」的狀態進入「等等，剛才想到什麼」，都要重新建立上下文。

---

## 現象：pnpm dev 的完全失能

更嚴重的問題：Tauri App 的前端有 `pnpm dev`（純瀏覽器模式，port 3000），但在沒有 mock backend 的情況下，所有頁面都顯示：

> 「此功能需在 AIRE 桌面 App 中使用」

意思是：`pnpm dev` 模式完全無法測試任何功能。

開發者的唯一選擇就是 `pnpm tauri dev`（熱重載，但要等 Rust 編譯初始化，第一次開啟要 2-3 分鐘）或 `pnpm tauri build`（完整打包，每次 3 分鐘）。

這是一個典型的熵增結果：沒有人刻意設計這個瓶頸，但因為沒有主動解決，它就存在了。

---

## 解法：Mock Backend 讓 pnpm dev 能跑完整流程

Mock backend 的設計目標：讓 `pnpm dev`（瀏覽器模式）能夠完整測試 UI 功能，不需要 Tauri 環境。

```
開發者改 UI 代碼
    ↓
瀏覽器熱重載（< 1 秒）
    ↓
直接在 localhost:3000 測試完整流程
    ↓
只需要驗收最終結果時才跑 pnpm tauri dev
```

迭代速度從「等 3 分鐘」變成「< 1 秒」。

---

## 架構設計（熵減原則的體現）

關鍵設計決策：不侵入業務代碼。

所有 IPC 呼叫都通過 `safeInvoke` 這一個入口點。在 `safeInvoke` 加入一行條件判斷：

```
環境 = Tauri     → 走真實 invoke（Rust backend）
環境 = dev + 非 Tauri → 走 mockInvoke（in-memory MockStore）
環境 = prod + 非 Tauri → 拋 NotInTauriError（使用者看到錯誤提示）
```

這個設計的好處：
- 所有業務代碼（頁面元件、hooks）不需要改
- mock 只在 development 環境存在，production build 完全不包含 mock 代碼
- MockStore 是 in-memory 的，頁面重新整理就重設，不污染真實資料

被否決的方案：
- MSW（Service Worker Mock）：IPC 不是 HTTP，無法攔截
- 獨立 mock provider context：需要侵入每個頁面的組件樹
- 環境變數開關 `NEXT_PUBLIC_USE_MOCK`：條件已經足夠（dev + 非 Tauri），不需要多一個開關

---

## 熵減的體現

mock backend 解決的不只是「等待時間」，而是整個開發循環的複雜度：

| 之前 | 之後 |
|------|------|
| 改 UI → 打包 → 開 App → 測試 | 改 UI → 瀏覽器熱重載 → 測試 |
| 每次迭代 3 分鐘 | 每次迭代 < 1 秒 |
| 測試 UI 需要 Rust 環境 | 測試 UI 只需要 Node.js |
| CI 測試需要打包完整 App | CI 可以只跑前端測試 |
| 新加入的開發者需要設定 Rust 才能測試 | 新開發者 npm install → pnpm dev 就能看到完整 UI |

每個「之後」都是熵減：移除了不必要的複雜度，讓關鍵工作（改 UI）更快完成。

---

## 這個原則的通用性

這不只是 AIRE 的問題。任何有「測試要先準備環境」瓶頸的專案，都可以問同樣的問題：

**「能不能讓開發者不準備完整環境，也能測試核心流程？」**

方案類型：
- Tauri App → mock IPC calls
- 需要硬體的 App → mock 硬體 SDK
- 需要雲端服務的 App → mock API responses（用 MSW 或 local server）
- 需要資料庫的 App → in-memory database（SQLite :memory:）

原則：**把測試環境的複雜度降到最低，讓開發者能快速驗證改動。**

---

## 什麼時候不用 mock

Mock 是工具，不是目標。以下情況不應該依賴 mock：

- 測試 Rust 後端的邏輯（要用真實的 Cargo test）
- 測試 PDF 渲染的輸出（要看真實的 PDF）
- 驗收最終版本（要用完整的 Tauri App 跑一遍）
- 資料遷移（要在真實資料庫上測試）

Mock 的邊界：**mock 適合測試 UI 行為，不適合測試系統整合。**
