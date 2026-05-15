# AIRE 踩坑紀錄

這個檔案解決的問題：記錄 SaaS / 桌面 App 開發過程中踩過的坑，讓下次不用再踩第二次。

只收錄和 SaaS / 桌面 App 開發相關的教訓（WordPress 相關的坑在別的地方）。

---

## 桌面 App 開發

### AIRE-001：Tauri App 開發效率瓶頸

**問題**：每次改 UI 都要 `pnpm tauri build`（3 分鐘），每天浪費 1-2 小時在等待。

**根因**：`pnpm dev`（瀏覽器模式）所有頁面顯示 fallback UI，完全無法測試功能，開發者被迫用打包模式。

**解法**：實作 mock backend（`src/lib/mock-backend.ts`），讓 `pnpm dev` 能跑完整流程。

**關鍵設計**：
- `safeInvoke` 加 dev mock 分支：`!isTauriEnv() && dev → mockInvoke`
- MockStore 是 in-memory 單例，`reset()` 方法供測試用
- 預設狀態：2 筆案件、5 筆日誌、3 條法條、公司名「測試不動產」

**教訓**：桌面 App 開發一定要有 mock backend，否則 UI 開發效率比 web 開發慢 10 倍。

---

### AIRE-002：icon 不能只改圖片檔

**問題**：把 `src-tauri/icons/` 目錄下的 png 直接替換，App 的 icon 沒有更新。

**根因**：Tauri 需要多個尺寸的 icon 檔（不同平台、不同解析度），手動替換可能遺漏某些尺寸。

**解法**：
```bash
# 準備一張 1024x1024 的 png，放到 src/app-icon.png
pnpm tauri icon src/app-icon.png
# Tauri 自動生成所有需要的尺寸到 src-tauri/icons/
```

**教訓**：所有 icon 生成必須用 `pnpm tauri icon`，不要手動複製 png 檔。

---

### AIRE-003：electron-builder 多 arch dmg 同名覆蓋（類比 Tauri）

**問題**（Electron，但 Tauri 多 arch 打包有相同風險）：同時 build arm64 + x64 時，如果 artifact 名稱沒有包含 `${arch}`，後 build 的會覆蓋先 build 的。

**症狀**：GitHub Releases 看起來成功，但只有一個架構的 dmg，另一個被靜默覆蓋。

**解法**（Tauri `tauri.config.json`）：
```json
{
  "bundle": {
    "macOS": {
      "artifactName": "${name}-${version}-${arch}.${ext}"
    }
  }
}
```

**教訓**：多架構打包時，artifact 名稱必須包含架構標識，避免靜默覆蓋。

---

## Next.js / 前端

### AIRE-004：Next.js 手機測試 CSS 被跨域保護擋

**問題**：手機連本機 dev server（區域網路 IP），CSS/JS 被跨來源保護擋住，React hydration 失敗，畫面卡在 loading skeleton。

**症狀**：API 正常回應，HTML 有內容，但 skeleton 永遠不消失。

**解法**（`next.config.ts`）：
```typescript
const nextConfig = {
  experimental: {
    allowedDevOrigins: ['192.168.x.x'],  // 替換為你的 LAN IP
  },
}
```
重啟 dev server 後正常。

**教訓**：Next.js 16+ 手機實機測試必須加 `allowedDevOrigins`。

---

### AIRE-005：中文路徑讓 Vite / Next.js dev server 卡死

**問題**：專案放在含中文的路徑（如 `2-顧問/anismile/`），Next.js/Vite dev server 啟動後白畫面，`<div id="root">` 永遠空，無 console error。

**根因**：Vite 輸出含中文字元的 `@fs` URL，fetch 超時。symlink 無效（Node 會 resolve 到真實路徑）。

**解法**：把整個專案實體搬到 ASCII 路徑。不是建 symlink，是 `mv`。

```bash
mv /Users/fishtv/Development/2-顧問/anismile /Users/fishtv/Development/clients/anismile
```

**教訓**：任何用 Vite / Next.js / Webpack dev server 的專案，路徑必須全 ASCII。新專案不放在含中文字元的目錄。

---

### AIRE-006：localhost 換專案要先清 Service Worker

**問題**：同一個 localhost port 之前跑過別的專案，Service Worker 還在，新專案白屏但 curl 正常。

**症狀**：curl 抓 HTML 是對的，瀏覽器看是空白（或舊專案的 UI）。

**診斷**：Service Worker 攔截了請求，回傳快取的舊頁面。

**解法（瀏覽器 console）**：
```javascript
(async () => {
  const r = await navigator.serviceWorker.getRegistrations()
  for (const x of r) await x.unregister()
  const k = await caches.keys()
  for (const c of k) await caches.delete(c)
})()
```
或 DevTools → Application → Storage → Clear site data。

**教訓**：同一個 port 換不同專案的 dev server，先清 Service Worker + caches。

---

## UI 驗證

### AIRE-007：UI 任務不能只看原始碼，必須截圖驗證

**問題**：Agent 完成 UI 任務，grep 確認 HTML 結構存在，但實際渲染錯誤（tab 切換邏輯錯誤、CSS 覆蓋問題）。

**教訓**：任何 UI 任務 commit 前，必須用 playwright / Chrome MCP 截圖驗證每個畫面狀態。

**正確流程**：
```
Agent 回報完成
    ↓
用截圖工具截圖所有畫面 / 狀態
    ↓
主對話確認截圖渲染正確
    ↓
有錯 → 退回 Agent 重做
確認正確 → commit
```

禁止：「Agent 回報完成 + grep 字串存在」就直接 commit。

---

## SDD 工作流

### AIRE-008：SDD 是 Popper 證偽主義，不是蓋房子

**問題**：把 SDD 當蓋房子理解（設計圖確定了就不變），遇到需求變更不知道怎麼辦，或強行按舊 spec 做。

**正確理解**：SDD 是實驗物理學。

- Spec = 假說（可被推翻）
- 紅燈測試 = 可驗偽的實驗
- 綠燈 = 假說暫時成立
- 需求變更 = 新證據推翻了舊假說 → ingest 更新 spec

**動工順序**：
```
propose（提出假說）
    ↓
逆推失敗點（哪裡可能炸）
    ↓
寫紅燈測試（把失敗點具體化）
    ↓
紅燈成立（確認測試會失敗）
    ↓
寫實作（讓紅燈變綠燈）
    ↓
如果假說需要修正 → ingest → 回到 propose
```

禁止：「先 apply 看實物再回頭修圖」（這是瀑布式反模式）。

---

### AIRE-009：Spectra analyze 必須到 0 findings 才能繼續

**問題**：analyze 有 Suggestion 等級的 findings，誤以為可以跳過，繼續 apply 後發現文件不一致，要回頭修。

**規則**：所有等級（Critical / Warning / Suggestion）都要修完，0 findings 才能繼續。

**教訓**：Suggestion 雖然不是錯誤，但它標記了未來可能造成誤解的地方，在 propose 階段修便宜，在 apply 後修昂貴。

---

## 外部代理（Copilot CLI / Kimi CLI）

### AIRE-010：Copilot CLI 在 --yolo 模式會刪 SDD 檔案

**問題**：Copilot `--yolo` 模式下，看到工作區有 untracked 的 openspec/ 變更，自作主張跑 `git clean -fd` 把它們刪掉。

**防護 SOP**（派工前強制執行）：
```bash
git add openspec/ .claude/ && git commit -m "wip: pre-dispatch checkpoint"
# 然後才派 Copilot
copilot --yolo --add-dir src/ --model gpt-5.2 -p @prompt.txt
# 派工後
git diff --stat  # 確認沒有意外改動
```

**關鍵點**：
- `--add-dir src/` 限制 Copilot 只能主動編輯 src/，但不限制 bash 指令
- `git clean -fd` 是 bash 指令，不受 `--add-dir` 限制
- 唯一防護：派工前把重要檔案 commit

### AIRE-011：Kimi CLI -w 只指程式碼目錄

**問題**：Kimi CLI `-w` 設了專案根目錄，Kimi 修改了 openspec/ 或 .claude/ 的檔案。

**正確用法**：
```bash
kimi --print -w src/ -p "重構 X 模組... 只修改 src/ 下的檔案，禁止動 openspec/、.claude/、docs/"
```

**關鍵點**：`-w` 只指程式碼目錄（`src/`），不指專案根目錄。加明文禁令在 prompt 結尾。

---

## iOS / 行動裝置

### AIRE-012：iOS Safari 在 http:// 環境 navigator.share 和 download 不正常

**問題**：在本機 `http://` 測試，iOS Safari 的 `navigator.share()` 靜默失敗，`<a download>` 不觸發儲存彈窗。

**根因**：iOS Safari 要求 HTTPS 才允許這些 API。

**教訓**：iOS 實機測試這些現象不是 bug，部署到 Vercel（HTTPS）後自動修復。本機測試看到問題先確認是否和 HTTPS 相關。
