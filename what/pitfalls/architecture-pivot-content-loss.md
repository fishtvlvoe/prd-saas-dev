# 架構轉移時內容層消失（Content Layer Lost on Arch Pivot）

**首次記錄**：2026-05-15 | AIRE PDF 不動產說明書事件

---

## 發生了什麼

AIRE 不動產說明書的 PDF，Fish 已經花大量時間在 SDD 裡定義了 7 個章節（封面、法規告知、產權調查表建物標示、土地標示、他項權利、稅費、成交行情）的完整欄位、資料來源、AI 要產什麼、版面規格。

這些規格存在：
- `docs/dossier-chapter-structure.md`
- `docs/dossier-implementation-spec.md`

但實際產出的 PDF 只有：
```
案件編號：AIRE-2026-002
型：土地
地號：磁二小段 88-1
地址：新北市磁文化路一段 188 號
屋主：林大仁
```

完全沒有章節，沒有結構，就是純文字。

---

## 根因：架構轉移時只搬基礎設施，沒搬內容

**舊架構（Next.js SaaS）**：

有完整的 PDF 產生器：
- `src/lib/pdf-generator/dossier.ts` — 7 章節產生邏輯
- `src/lib/pdf-generator/templates/dossier.html` — Handlebars 模板，完整版面
- `src/lib/document-generator/build-input.ts` — 資料組裝
- `src/app/api/documents/export-pdf/route.ts` — 匯出 API

**新架構（Tauri 桌面 App）**：

PDF 產生器被重寫為 `@react-pdf/renderer`，但只搬了基礎設施：
- `src/lib/pdf-engine/engine.ts` — render pipeline（有）
- `src/lib/pdf-engine/document.tsx` — 文件模板（stub，空殼）

`document.tsx` 的實際內容：
```tsx
<Page>封面頁（Page 1）</Page>
<Page>基本資訊（Page 2）</Page>
<Page>位置圖（Page 3）</Page>
```

加了一個 comment：`// ...其他欄位（Stage 3+ 再補齊）`

**這個 comment 就是坑**：沒有任何 Spectra task 被建立，沒有任何追蹤。

---

## 為什麼這個坑這麼深

### 1. Spectra spec 成了孤兒

Spectra specs（pdf-live-preview、fillable-pdf-output、template-rendering）的 Purpose 全部是：
```
TBD - created by archiving change 'xxx'. Update Purpose after archive.
```

`@trace code:` 指向的檔案（如 `src/lib/pdf-generator/dossier.ts`）在新架構裡根本不存在。Spec 存在，code 已刪，沒人知道。

### 2. Stub + "Stage N+ 再補齊" = 靜默技術債

```tsx
// ...其他欄位（Stage 3+ 再補齊）
```

這個 comment 沒有：
- 對應的 Spectra task
- 對應的 GitHub issue
- 任何可追蹤的東西

它就消失在 codebase 裡，直到有人問「為什麼 PDF 只有五行文字？」

### 3. 架構轉移沒有「要重做什麼」清單

從 Next.js SaaS → Tauri 桌面 App 這個轉移，有把：
- 路由系統搬過去
- 資料庫換掉（Postgres → SQLite）
- API routes 換成 Tauri commands
- PDF 引擎換成 @react-pdf/renderer

但沒有明確問：**「舊架構裡哪些業務邏輯在新架構裡還沒有對應實作？」**

---

## 損失

Fish 花在 SDD 定義 7 章節的時間沒有轉換成實作，要從頭再跑一次 propose → apply。

---

## 預防規則

### 規則 1：架構轉移必須有「業務邏輯移植清單」

每次做大架構轉移（換框架、換平台、換語言）前，先列出：

| 舊架構實作 | 業務邏輯 | 新架構狀態 |
|-----------|---------|-----------|
| `pdf-generator/dossier.ts` | PDF 7章節產生 | ❌ 未移植 |
| `template-engine.ts` | Handlebars 合板 | ❌ 未移植 |
| `document-generator/build-input.ts` | 資料組裝 | ❌ 未移植 |

沒有這張表 = 轉移視為未完成。

### 規則 2：Stub + comment 必須同時建立 Spectra task

寫了任何 `TODO`、`// 待補齊`、`Stage N+ 再補齊`：
- **必須** 同一個 commit 裡建立對應的 Spectra task（或 GitHub issue）
- Stub 裡放 task ID：`// TODO: see change/dossier-chapter-impl`
- 禁止無對應追蹤的 stub comment

### 規則 3：Spec archive 後必須 Purpose 填完

Spectra `archive` 之後，spec 的 Purpose 如果還是 `TBD`：
- 下一個 session 開始前先補 Purpose
- `@trace code:` 指向不存在的檔案 → 更新為新架構的對應檔案路徑
- 做不到 → 明確標記這個 spec 是「歷史參考，新架構未實作」

### 規則 4：Fish 花時間定義的東西不能消失

SDD 規格（欄位清單、資料來源、版面設計）= Fish 的時間 = 資產。

架構轉移時，如果某個 spec 對應的 code 被刪掉，必須：
1. 在新的 Spectra change 裡明確說「這是移植自舊架構的規格」
2. 從舊 spec 裡把 Fish 定義的欄位和邏輯帶過來
3. 不能讓規格消失在 `docs/` 裡沒人用

---

## 診斷指令（發現是否踩坑）

```bash
# 找所有 stub comment（有 TODO/待補/再補齊 但沒有 spectra task ID）
grep -r "Stage.*再補齊\|TODO\|待補" src/ --include="*.ts" --include="*.tsx" | grep -v "spectra\|change/"

# 找所有 Purpose TBD 的 spec
grep -r "TBD" openspec/specs/*/spec.md

# 找 @trace code 指向不存在檔案的 spec
# （手動核對 @trace 的路徑是否在 src/ 裡存在）
```
