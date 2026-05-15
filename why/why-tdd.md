# 為什麼先寫測試再寫實作（TDD）

這個檔案解決的問題：解釋 TDD 不是「聰明的開發者才需要的工作量」，而是讓每次實作都有明確完成標準的根本方法。

---

## TDD 的核心邏輯

TDD = 先證明「這會壞」，再去修。

```
Phase 1：定義「什麼是對的」（Spec）
    ↓
Phase 2：寫測試，證明「現在是錯的」（紅燈）
    ↓
Phase 3：寫實作，讓測試通過（綠燈）
    ↓
Phase 4：清理（重構，不改行為，只改結構）
```

關鍵：Phase 2 和 Phase 3 不能顛倒。

---

## 為什麼不能跳過 Phase 2

直接進 Phase 3（先寫實作，再寫測試）的問題：

**問題 1：測試會在不知不覺中配合實作**

先有實作，再寫測試的人，會傾向於寫「能通過這個實作」的測試，而不是「能驗證需求是否滿足」的測試。

測試配合實作 → 測試失去意義（它只驗證了「代碼做了它做的事」，不是「代碼做了它應該做的事」）。

**問題 2：完成的定義不清楚**

沒有紅燈，不知道什麼叫「做完了」。開發者可以一直加功能，一直「優化」，永遠不確定是否完成。

有紅燈的情況：測試一個個從紅變綠，完成的定義是所有紅燈都變成綠燈。清楚、客觀、不爭議。

**問題 3：實作完之後很難寫測試**

代碼已經存在了，要測試它，測試代碼必須了解實作細節，導致測試和實作強耦合。

強耦合的測試 = 改實作就要改測試 = 測試的維護成本極高 = 測試最終被刪掉。

---

## 失敗矩陣：TDD 的規劃工具

在開始寫測試之前，先做失敗矩陣分析：

**格式**：

| 失敗點 | 紅燈測試名稱 | 預期錯誤訊息 |
|-------|-----------|------------|
| 未知的 IPC 命令 | `mock-unknown-cmd-throws` | `Error: Mock not implemented: unknown_cmd` |
| reset() 後狀態回初始值 | `mock-store-reset-restores-defaults` | — |
| prod 環境不啟用 mock | `tauri-bridge-prod-throws-not-in-tauri` | `Error: NotInTauriError` |

失敗矩陣的用途：
1. 讓每個失敗情境有明確的測試名稱和預期輸出
2. 讓 Fish 確認「這些失敗情境是完整的」再開始實作
3. 讓派工的 Agent 知道「要寫哪些測試」而不是「任意寫幾個」

---

## AIRE 的失敗矩陣範例（mock backend）

從 browser-dev-mock 的 spec 整理：

| 失敗點 | 紅燈測試 | 預期結果 |
|-------|---------|---------|
| dev 環境非 Tauri，命令正常 | `invoke-dev-mock-dispatches` | mockInvoke 被呼叫，回傳 mock 資料 |
| Tauri 環境，命令走真實 invoke | `invoke-tauri-uses-real-invoke` | tauri invoke 被呼叫 |
| prod 環境非 Tauri，命令拋錯 | `invoke-prod-non-tauri-throws` | `NotInTauriError` |
| reset() 後 license 恢復 none | `store-reset-license-to-none` | `{ status: "none", serialKey: null }` |
| reset() 後 cases 恢復 2 筆預設 | `store-reset-cases-to-defaults` | `cases.size === 2` |
| 未知命令拋明確錯誤 | `mock-unknown-cmd-throws` | `Error: Mock not implemented: <cmd>` |
| license activate 後狀態改變 | `mock-license-activate-updates-status` | `status === "valid"` |

---

## TDD 和 SDD 的關係

```
SDD（Spec）定義「什麼是對的」
    ↓
TDD Phase 2（紅燈）把「對的」轉成可執行的驗證
    ↓
TDD Phase 3（綠燈）讓實作滿足驗證
    ↓
Spectra analyze 確認 spec 和實作沒有背離
```

三個工具服務同一個目標：**讓「完成」有客觀的定義。**

沒有 spec，不知道做什麼。
沒有測試，不知道是否做對。
沒有 analyze，不知道做的和說的是否一致。

---

## 在 SDD 工作流中的位置

```
spectra propose（建立 spec + design + tasks）
    ↓
spectra analyze（0 findings 才繼續）
    ↓
[apply 開始前] 產出失敗矩陣 + 紅燈測試清單 → Fish 確認
    ↓
派 Sonnet / Copilot 只寫紅燈測試（不寫實作）
    ↓
跑測試，確認全部紅燈
    ↓
[Fish 確認] 紅燈清單是否完整
    ↓
派 Copilot / Codex 寫實作讓紅燈變綠燈
```

Phase 2（只寫測試不寫實作）和 Phase 3（寫實作讓測試通過）必須分開執行。
混在一起 = 測試失去獨立性。

---

## TDD 的常見誤解

**誤解 1：TDD 讓開發變慢**

表面上多了寫測試的時間。實際上：
- 減少了 debug 的時間（bug 被測試攔截，不是上線後才發現）
- 減少了回歸的時間（改了 A 不會不知道壞了 B）
- 減少了「不敢改」的恐懼（有測試當安全網，重構才有底氣）

**誤解 2：只有邏輯複雜的代碼才需要測試**

簡單的代碼也可能在重構時被破壞。

測試的存在不是因為「這段代碼很難」，而是因為「我想確認這段代碼以後還會繼續正確」。

**誤解 3：測試覆蓋率越高越好**

100% 覆蓋率的測試可能都是配合實作寫的（先有實作再寫測試的副作用）。

重要的不是覆蓋率，是「測試有沒有驗證需求」。

失敗矩陣比覆蓋率更有意義：每個重要的失敗情境都有對應的測試。
