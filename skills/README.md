# Skills 規劃

這個目錄預留給未來可以轉化為 Claude Code skills 的工作流程。

---

## 目前規劃的 Skills

### tauri-app-init

**觸發條件**：開新 Tauri + Next.js 專案

**預期功能**：
- 建立標準目錄結構
- 複製 `tauri-bridge.ts` 模板
- 複製 `mock-backend.ts` 框架（清空 commands，保留架構）
- 建立基本的測試結構
- 初始化 `openspec/` 目錄（Spectra）

**素材來源**：`what/templates/tauri-bridge-template.md`

---

### mock-backend-setup

**觸發條件**：現有 Tauri 專案需要加 mock backend

**預期功能**：
- 分析現有的 IPC commands（讀 `src-tauri/src/commands/`）
- 自動生成對應的 mock handlers
- 修改 `tauri-bridge.ts` 加入 dev mock 分支
- 生成對應的測試檔案

**素材來源**：`what/templates/tauri-bridge-template.md`、`what/architecture/tauri-nextjs-stack.md`

---

### desktop-release

**觸發條件**：準備 Tauri App 的新版本 release

**預期功能**：
- 讀取 `what/checklists/release-checklist.md` 逐項確認
- 自動更新三個地方的版本號
- 跑測試
- 提醒需要手動做的步驟（icon 確認、授權測試）

**素材來源**：`what/checklists/release-checklist.md`

---

## 如何貢獻 Skill

1. 在這個目錄建立 `skill-name/` 子目錄
2. 在子目錄建立 `SKILL.md`（skill 的觸發條件、功能描述、使用方式）
3. 建立 `knowledge/` 子目錄，放相關的知識文件
4. 更新這個 README

Skill 的格式參考：`~/.claude/skills/dp/SKILL.md`
