# 打包前檢查清單

這個檔案解決的問題：確保每次 release 前都走完必要的驗證步驟，不因為遺漏導致版本出問題。

---

## 使用方式

打包前，逐一核對。所有項目 ✓ 才能執行 `pnpm tauri build`。

---

## 1. 版本號（必做，3 個地方）

- [ ] `src-tauri/tauri.conf.json`：`version` 欄位已更新
- [ ] `src-tauri/Cargo.toml`：`version` 欄位已更新
- [ ] `package.json`：`version` 欄位已更新

三個地方的版本號必須一致。

版本規則：
- Bug fix → patch（0.1.0 → 0.1.1）
- 新功能 → minor（0.1.0 → 0.2.0）
- Breaking change → major（0.1.0 → 1.0.0）

---

## 2. Icon（必做）

- [ ] `src/app-icon.png` 是最新的 logo（1024x1024 px）
- [ ] 跑過 `pnpm tauri icon src/app-icon.png` 重新生成所有尺寸
- [ ] 確認 `src-tauri/icons/` 目錄下的檔案時間戳是最新的

**禁止**：直接替換 `src-tauri/icons/` 下的個別 png 檔（會遺漏某些尺寸）。

---

## 3. 測試通過

- [ ] 前端單元測試全部通過：`pnpm test`
- [ ] Rust 後端測試通過：`cd src-tauri && cargo test`
- [ ] 沒有 TypeScript 編譯錯誤：`pnpm tsc --noEmit`

---

## 4. 授權邏輯（prod 環境）

- [ ] Mock backend 在 prod build 不啟用（確認 `NODE_ENV=production` 時不走 mockInvoke）
- [ ] 授權序號驗證在 Tauri 環境正常運作
- [ ] 啟動流程（輸入序號 → 驗證 → 進入 dashboard）可以完整走過

---

## 5. SDD 同步

- [ ] 所有 active spectra changes 都已 archive 或 park（沒有留著未完成的 change）
- [ ] `spectra list` 回傳空清單（或只有刻意 park 的 change）

---

## 6. Git 狀態

- [ ] `git status` 顯示 working tree 乾淨（沒有 uncommitted 的改動）
- [ ] 所有改動都已 commit 並 push 到正確的 branch
- [ ] PR 已合併到 main

---

## 7. 打包執行

```bash
# macOS
pnpm tauri build

# 輸出位置
# .app → src-tauri/target/release/bundle/macos/AIRE.app
# .dmg → src-tauri/target/release/bundle/macos/AIRE_x.y.z_aarch64.dmg（arm64）
#       → src-tauri/target/release/bundle/macos/AIRE_x.y.z_x64.dmg（x64）
```

---

## 8. 打包後驗收

- [ ] 開啟 .app / .dmg，確認版本號顯示正確
- [ ] 走一遍授權啟動流程（序號輸入 → 啟動 → 進入 dashboard）
- [ ] 確認案件列表、品牌設定、PDF 匯出等核心功能正常
- [ ] 確認 App icon 是正確的

---

## 9. Release

- [ ] 建立 GitHub Release，tag 格式：`v0.1.0`
- [ ] 上傳 .dmg（arm64 + x64）和 .msi
- [ ] Release notes 說明本版本的改動
- [ ] 確認 artifact 名稱包含架構標識（避免同名覆蓋）

Artifact 命名規則：
- `AIRE-0.1.0-aarch64.dmg`（macOS arm64）
- `AIRE-0.1.0-x86_64.dmg`（macOS x64）
- `AIRE-0.1.0-x86_64.msi`（Windows）

---

## 常見問題排查

**打包失敗：Rust 編譯錯誤**
```bash
cd src-tauri
cargo build --release
# 看詳細錯誤訊息
```

**打包失敗：前端 build 錯誤**
```bash
pnpm build
# 看 Next.js build 錯誤訊息
```

**App 開啟後白屏**
1. 確認 `NODE_ENV=production`（不是 development）
2. 確認 mock backend 沒有在 prod 啟用
3. 查 App console log（Tauri → 右鍵 → Inspect Element → Console）

**macOS Gatekeeper 阻擋**
- 首次開啟：右鍵 → 打開 → 允許
- 長期解法：代碼簽署（Apple Developer 帳號）
