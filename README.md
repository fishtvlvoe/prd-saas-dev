# PRD SaaS 開發知識庫

這個知識庫解決一個問題：把「Fish 為什麼這樣做事」變成可傳遞的結構，讓 AI 協作者、未來的自己、以及任何接手這個方向的人，都能快速對齊思維框架，不用重新問一遍「為什麼」。

知識庫以 AIRE（不動產說明書自動化桌面 App）作為主要實例，但所有觀念對其他 SaaS / 桌面 App 同樣適用。

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
├── why/                     設計哲學與動機
│   ├── design-philosophy.md     Fish 的設計哲學核心（工具觀）
│   ├── entropy-reduction.md     熵減 vs 熵增（為什麼「少」比「多」更難）
│   ├── persona-pain-scenario.md 人群→痛點→場景→需求框架
│   ├── problem-validation.md    可驗偽的思維流程（Popper 在商業的應用）
│   ├── why-sdd.md               為什麼先寫 spec 再寫 code
│   ├── why-tdd.md               為什麼先寫測試再寫實作
│   ├── why-local-dev.md         為什麼要消除打包等待（mock backend 的動機）
│   └── why-st-foundation.md     為什麼用 Supastarter 作為 SaaS 底層
├── how/                     工作流程與決策方法
│   ├── reverse-engineering.md   逆向拆解法（市場→技術選型）
│   ├── sdd-workflow.md          Spectra 完整工作流
│   ├── decision-tree.md         技術選型決策樹
│   ├── testing-strategy.md      三層測試策略
│   └── dev-environment.md       開發環境三模式
└── what/                    具體產出與參考
    ├── architecture/
    │   └── tauri-nextjs-stack.md    技術架構（含 IPC 命令清冊）
    ├── pitfalls/
    │   └── aire-lessons.md          踩坑紀錄（SaaS / 桌面 App 相關）
    ├── templates/
    │   └── tauri-bridge-template.md tauri-bridge.ts 通用模板
    └── checklists/
        └── release-checklist.md     打包前檢查清單
```

---

## 給誰用

- **AI 協作者（Claude/Copilot）**：每個 session 開始可以讀 `why/` 目錄建立哲學底層，再讀 `how/` 理解工作流，執行 `what/` 的具體工作。
- **未來的自己**：忘了「為什麼這樣決定」的時候，先查這裡，不要憑記憶猜。
- **接手專案的人**：從 `why/design-philosophy.md` 開始讀，順著邏輯鏈往下。

---

## 更新機制

踩到新坑 → 更新 `what/pitfalls/aire-lessons.md`。

技術選型有新決策 → 更新 `what/architecture/` 和 `how/decision-tree.md`。

工作流有調整 → 更新 `how/sdd-workflow.md`。

哲學有新體悟 → 更新 `why/design-philosophy.md`，並確認邏輯鏈（Why→How→What）仍然通得過。
