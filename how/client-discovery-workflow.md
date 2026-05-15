# 客戶發掘工作流（Client Discovery Workflow）

這個檔案解決的問題：定義 Fish 跟 Claude 之間的協作分工 — 誰負責「找問題」、誰負責「配資產」、誰負責「給方案」，避免角色重疊或漏掉步驟。

---

## 核心分工

```
Fish ─── Perplexity ─── 找問題
   ↓（把 Perplexity 結果丟給 Claude）
Claude ─── 自有資產盤點 ─── 配工具
   ↓（產出對應表 + 合作方案）
Fish ─── 決策 ─── 選方向 / 修方向 / 放掉
```

**不是**：Claude 同時做問題分析 + 解決方案 → 容易自圓其說，看不出問題真假。
**是**：問題從外部來源（Perplexity / 客戶訪談）取得，Claude 只負責「拿我們有的東西去配」。

---

## 完整流程（兩階段）

### 階段 1：問題探索（Fish 主導）

工具：Perplexity / Gemini / 客戶訪談

產出：
- 客戶身分（產業、規模、決策者）
- 痛點清單（重複、耗時、難量化的工作）
- 市場現況（其他人怎麼解、解得好不好）

格式：把 Perplexity 整段搜尋結果或摘要丟給 Claude，附帶客戶名稱。

### 階段 2：資產配對與方案產出（Claude 主導）

收到 Perplexity 結果後，Claude 執行：

1. **盤點自有資產**（每次重新盤，不憑記憶）
   - products/、8-外掛/、skills/、技術元件

2. **執行客戶機會對應法**（見 `how/client-opportunity-mapping.md`）
   - 痛點 ↔ 資產對應表
   - 三層合作深度（租賃 / 共同開發 / 全包）
   - 選 MVP（家長/老闆感受最強 + 操作者零學習成本）

3. **產出給 Fish 的決策包**
   - 對應表
   - 三方案各自的範圍、時程、報價基準
   - MVP 建議 + 理由
   - 反例（什麼情況不要接）

### 階段 3：Fish 決策

Fish 收到方案後決定：
- 走哪一個方向（A / B / C 或修改）
- 進 Spectra 提案（/spectra-propose）
- 還是放掉這個客戶

---

## 為什麼這樣分工

**外部來源做問題分析的好處**：
- Perplexity 引用真實網路資料（補習班、行業報導、競品文章），不會自圓其說
- Fish 能看到「市場上其他人怎麼描述這個痛點」，校驗自己的假設
- Claude 不適合做「市場調研」（Claude 知識有截止日 + 不會主動上網找對的關鍵字）

**Claude 做資產配對的好處**：
- Claude 對自有專案（products/、8-外掛/）的脈絡最熟
- 配對邏輯需要記住熵減、工具配合人等內部哲學（見 `why/design-philosophy.md`）
- 同一個痛點可以被多個資產解，需要算改造幅度跟複用率，這是工程判斷

**Fish 做最終決策的好處**：
- 商業判斷（定價、合作對象選擇、戰略方向）只有 Fish 能做
- 避免 Claude 自圓其說地把所有客戶都判定成「可以接」

---

## 訊號 → 動作對照

| Fish 給的訊號 | Claude 該做的事 |
|--------------|---------------|
| 丟一段 Perplexity 連結或結果 | 進入階段 2，跑客戶機會對應法 |
| 「幫我看一下這個客戶」 | 先問是否有 Perplexity 分析，沒有則建議 Fish 先做階段 1 |
| 「這個方案不對」 | 不要硬辯，回到對應表重新挑 MVP 或重盤資產 |
| 「就 A 方案」 | 直接走 `/spectra-propose` 開新 change |

---

## 反模式（不要這樣做）

- ❌ Claude 自己生痛點 + 自己給方案（缺外部驗證，容易過度樂觀）
- ❌ Fish 丟 Perplexity 後 Claude 直接跳到「來實作」，跳過資產對應表
- ❌ 沒盤點自有資產就推薦「開新專案」（違反熵減 — 能複用就不新建）
- ❌ 方案只給一個（應該給三層 A/B/C 讓 Fish 選合作深度）

---

## 跟其他文件的關係

- 階段 2 詳細步驟 → `how/client-opportunity-mapping.md`
- 痛點要怎麼挖 → `why/persona-pain-scenario.md`
- 痛點真假驗證 → `why/problem-validation.md`
- Fish 選完方向後動工 → `how/sdd-workflow.md`
- 為什麼能複用就不新建 → `why/entropy-reduction.md`
