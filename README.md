# PRD SaaS 開發知識庫

用哲學角度設計軟體開發 — 從「為什麼」出發，通用於任何 SaaS 產品或 AI Agent 系統。

這份知識庫有兩種閱讀方式：

### 給人看 — 視覺化 Landing Page

**[打開 Landing Page](https://fishtvlvoe.github.io/prd-saas-dev/site/index.html)** — 單頁網頁，所有專業術語都有白話翻譯 + 生活比喻 + 圖解（費曼學習法）。沒有工程背景也能讀懂。

### 給 AI 看 — 英文實作指南

**[打開 AI Guide](ai-guide/README.md)** — 10 份英文文件，結構化的 step-by-step 格式。AI agent（Claude / Codex / GPT）讀完可以直接照做，不需要人類翻譯。

---

## 黃金圈邏輯鏈

```
Why（為什麼這樣做）
  熵減哲學 ── 每個元素都要有職責，多餘的砍掉
  血清素設計 ── 工具讓人用完安心離開，不靠上癮留人
  工具配合人 ── 不是人配合工具
    ↓
How（怎麼做到）
  逆向拆解 ── 從市場需求反推技術選型
  SDD（規格驅動開發） ── 先定義「什麼是對的」再動手
  TDD（測試驅動開發） ── 先證明「這會壞」再去修
    ↓
What（具體做了什麼）
  Tauri + Next.js 架構 ── 桌面 App 的技術實作
  Mock Backend ── 消除開發迭代效率瓶頸
  踩坑紀錄 ── 這些坑不要再踩第二次
```

---

## 知識庫結構

```
prd-saas-dev/
├── site/                    給人看（視覺化網頁）
│   └── index.html               單頁 Landing Page，費曼學習法解釋所有術語
│
├── ai-guide/                給 AI 看（英文實作指南）
│   ├── README.md                索引與閱讀順序
│   ├── sdd-workflow.md          SDD 完整工作流（step-by-step）
│   ├── tdd-pattern.md           TDD 紅綠燈模式
│   ├── persona-pain-scenario.md 需求分析框架
│   ├── design-principles.md     設計原則清單
│   ├── dev-environment.md       三種開發模式設定指南
│   ├── decision-tree.md         技術選型決策樹
│   ├── pitfalls.md              通用 SaaS / 桌面應用踩坑清單
│   └── templates/               可複用的程式碼模板與檢查清單
│
├── why/                     原始素材：設計哲學與動機（中文）
│   ├── design-philosophy.md     工具觀核心
│   ├── entropy-reduction.md     熵減 vs 熵增
│   ├── persona-pain-scenario.md 人群→痛點→場景→需求框架
│   ├── problem-validation.md    可驗偽的思維流程（Popper）
│   ├── why-sdd.md               為什麼先寫 spec 再寫 code
│   ├── why-tdd.md               為什麼先寫測試再寫實作
│   ├── why-local-dev.md         為什麼要消除打包等待
│   └── why-st-foundation.md     為什麼用 Supastarter 作為 SaaS 底層
├── how/                     原始素材：工作流程與決策方法（中文）
│   ├── reverse-engineering.md   逆向拆解法
│   ├── sdd-workflow.md          Spectra 完整工作流
│   ├── decision-tree.md         技術選型決策樹
│   ├── testing-strategy.md      三層測試策略
│   └── dev-environment.md       開發環境三模式
└── what/                    原始素材：具體產出與參考（中文）
    ├── architecture/            技術架構
    ├── pitfalls/                踩坑紀錄
    ├── templates/               程式碼模板
    └── checklists/              檢查清單
```

---

## 給誰用

| 讀者 | 從哪裡開始 |
|------|-----------|
| **完全不懂程式的人** | 打開 `site/index.html`，從術語翻譯站開始 |
| **AI agent** | 讀 `ai-guide/README.md`，按索引順序執行 |
| **開發者（人類）** | 讀 `why/design-philosophy.md`，順著黃金圈邏輯鏈往下 |
| **未來的自己** | 忘了「為什麼這樣決定」→ 查 `why/`；忘了「怎麼做」→ 查 `how/` |

---

## 更新機制

踩到新坑 → 更新 `what/pitfalls/aire-lessons.md`。

技術選型有新決策 → 更新 `what/architecture/` 和 `how/decision-tree.md`。

工作流有調整 → 更新 `how/sdd-workflow.md`。

哲學有新體悟 → 更新 `why/design-philosophy.md`，並確認邏輯鏈（Why→How→What）仍然通得過。
