# SDD 工作流（Spectra 完整流程）

這個檔案解決的問題：給 AI 協作者（或未來的自己）一份可以直接照著走的 SDD 執行手冊。

---

## 一句話總結

**先定義對的標準（spec），再去達到它（code），再驗證達到了（test），最後歸檔（archive）。**

---

## 完整工作流

```
discuss（可選）
    ↓
propose（必做）
    ↓
analyze（必做，0 findings 才繼續）
    ↓
[TDD] 失敗矩陣 + 紅燈測試（apply 前必做）
    ↓
apply（派工執行）
    ↕
ingest（需求變更時回來更新）
    ↓
archive（完成後歸檔）
```

---

## 各階段說明

### discuss（討論，可選）

**何時用**：需求不清楚、有多個方向需要討論、或有架構上的重大決策要做。

**產出**：共識（寫在 discuss 記錄中，或直接進入 propose 時體現）。

**跳過條件**：需求已經很清楚，不需要討論就能直接寫 spec。

---

### propose（提案，必做）

**產出三個文件**：

`spec.md` — 功能規格
```
# 功能名稱

## 問題陳述
這個 change 解決什麼問題？

## 目標
成功的定義是什麼？

## 功能範圍
做什麼（列出）

## 排除範圍（Non-Goals）
不做什麼（同樣重要）

## 接受標準（Acceptance Criteria）
可驗證的條件，每條都對應紅燈測試
```

`design.md` — 設計決策
```
# 設計決策

## D1 — 決策名稱
選擇：[方案名]
理由：[為什麼選這個]
否決：[方案 B]（理由：[為什麼不選]）
否決：[方案 C]（理由：[為什麼不選]）

## D2 — 下一個決策
...
```

`tasks.md` — 實作任務清單
```
## Wave 1（可並行）
- [ ] Task 1.1 [Tool: Copilot] — 描述
- [ ] Task 1.2 [Tool: Kimi] — 描述

## Wave 2（依賴 Wave 1）
- [ ] Task 2.1 [Tool: Sonnet] — 描述
```

**Propose 完成後的自我審查**（L026）：
- 每個 design.md 的決策都要在 tasks.md 中有對應的實作任務
- 每個 spec.md 的接受標準都要有對應的測試任務
- 否決方案要有明確理由

---

### analyze（一致性分析，必做）

**跑指令**：
```bash
spectra analyze
```

**四個維度**：
- Coverage：spec 的所有需求都有對應的 tasks 嗎？
- Consistency：design.md 的決策和 tasks.md 的實作一致嗎？
- Ambiguity：有沒有模糊不清的描述？
- Gaps：有沒有遺漏的邊界情況？

**標準**：0 findings 才能繼續（包含 Suggestion 等級，全部修完）。

如果 analyze 發現問題，回去修 spec / design / tasks，再跑一次。

---

### TDD 失敗矩陣（apply 前必做，強制）

在開始寫實作代碼之前，產出失敗矩陣：

**格式**：
```
| 失敗點 | 紅燈測試名稱 | 預期錯誤訊息 |
|-------|------------|------------|
| ...   | ...        | ...        |
```

**步驟**：
1. 讀 spec.md 的接受標準
2. 讀 design.md 的 Risks / Trade-offs
3. 列出每個可能失敗的情境
4. 對每個失敗情境，寫出對應的測試名稱和預期結果
5. 給 Fish 確認失敗矩陣是否完整
6. 派 Sonnet / Copilot 只寫紅燈測試（不寫實作）
7. 跑測試，確認全紅燈
8. 紅燈清單確認後，才進入寫實作

---

### apply（執行，派工）

**開始前**：
```bash
git status        # 確認工作區乾淨
git add openspec/ .claude/ && git commit -m "wip: pre-dispatch checkpoint"  # 保護 SDD 檔案
```

**Wave 執行**：
1. 讀 tasks.md，列出本 Wave 的任務和工具分配
2. 給 Fish 確認分配表
3. 並行派出（不同檔案的任務可並行）
4. 等所有任務完成
5. Code Review（Kimi CLI，diff > 10 行）
6. 跑測試：`npm run test` 或 `cargo test`
7. git commit（conventional commits）
8. 更新 tasks.md（標記 `[x]`）
9. 下一個 Wave

**Wave 完成驗收（缺一不過）**：
- build 0 錯誤
- Code Review 無 Critical
- git commit 存在
- tasks.md `[x]` 已更新

---

### ingest（需求變更時）

何時用：apply 過程中發現需求有誤、或 Fish 改變了方向。

```bash
spectra ingest    # 進入互動模式，更新 spec
spectra analyze   # 重新分析一致性
# 繼續 apply
```

不要在代碼裡直接偏離 spec，要先更新 spec，再繼續。

---

### archive（完成後歸檔）

何時用：所有 tasks 都標記 `[x]`，測試都通過，功能已合併到主分支。

```bash
spectra archive
```

archive 後，change 從 active 移入 archive，不再出現在 `spectra list`。

---

## 常用 Spectra 指令

```bash
spectra list              # 列出所有 active changes
spectra list --parked     # 列出暫存的 changes
spectra unpark <name>     # 恢復暫存的 change
spectra analyze           # 分析當前 change 的一致性
spectra validate          # 驗證（analyze 通過後才能跑）
spectra park <name>       # 暫存 change
```

---

## 工具分配速查

| 任務類型 | 用什麼工具 |
|---------|-----------|
| UI 元件 / API / 業務邏輯 | Copilot CLI（`copilot --yolo --model gpt-5.2`）|
| 機械式重構 / 批量改名 | Kimi CLI（`kimi --print -w src/ -p "..."`）|
| 複雜整合 / E2E 測試 | Sonnet 子代理 |
| Code Review（diff > 10 行）| Kimi CLI（`kimi -p --print -w <dir>`）|
| 外部研究 / 文件查詢 | Gemini CLI（`gemini -p "..."`）|
| 1-2 行 hotfix | 主對話直接做 |

完整分工表：`~/.claude/rules/routing.md`
