# BuyGo+1 部署 / 執行期教訓彙整

WordPress 外掛部署到 InstaWP + GitHub Actions auto-release 鏈、Eloquent / wp-cli / opcache 等執行期細節。2026-05-15 修「商品分配統計三頁面對不起來」時踩到的坑，集中記錄。

---

## BUYGO-001：dependent PR 鏈，被依賴的 PR merge + delete-branch 會讓子 PR 永久 closed

**問題**：開了 PR #12 base 在 `fix/allocation-stats-stale-cache`（PR #11 的 head）。PR #11 合進 main 時用 `--delete-branch`，原 base 分支被刪 → PR #12 mergeStateStatus 變 DIRTY → 過幾秒自動 CLOSED。

**根因**：GitHub GraphQL 規定 `Cannot change the base branch of a closed pull request`，已 close 的 PR 也不能 reopen。

**解法**：開新 PR 替代（PR #13），把原 base 設成 main 並 rebase 處理 squash merge 帶來的「skipped previously applied commit」。

**教訓**：
- dependent PR chain merge 時，**先把子 PR 的 base 改成 main**（`gh pr edit <num> --base main`）再 merge 父 PR；或父 PR merge 用 `--no-delete-branch` 留住分支等子 PR 也合完
- 一旦 base 被刪 + PR closed，就是新開一個 PR，不要花時間試 reopen

---

## BUYGO-002：GitHub Actions GITHUB_TOKEN 推的 tag 不會觸發其他 workflow

**問題**：auto-release.yml 跑成功推了 v1.7.8 / v1.7.9 / v1.7.10 tag，但 release.yml（`on: push: tags: ['v*']`）完全沒被觸發，GitHub Releases 一直只到 v1.7.7。

**根因**：GitHub 防無限迴圈規定 — 用 `GITHUB_TOKEN` 推的 ref（含 tag）**不會**觸發其他 workflow。

**解法**：
1. 本機 `git push origin --delete v1.7.X` 刪遠端 tag
2. `git tag -d v1.7.X` 刪本機 tag
3. `git tag v1.7.X <commit-sha> -m "Release X"`
4. `git push origin v1.7.X`（用 PAT 推）→ release.yml 觸發

**教訓**：
- 任何「auto-tag → release pipeline」鏈，必須驗證實際有 release artifact 產出，不能信 auto-tag workflow 成功就放心
- 長期修：auto-release.yml 改用 PAT secret 推 tag（或用 `peter-evans/create-pull-request` 之類）
- 第二保險：release.yml 加 `workflow_dispatch:` 觸發器讓我能手動跑

---

## BUYGO-003：`wp eval --skip-plugins` 讓自家 plugin class 找不到

**問題**：用 wp-cli 跑 service 代碼驗證部署：
```bash
wp --skip-themes --skip-plugins eval '$calc = new \BuyGoPlus\Services\ProductStatsCalculator(...)'
```
PHP Fatal：`Class "BuyGoPlus\Services\ProductStatsCalculator" not found`。

**根因**：`--skip-plugins` 字面意思是不載入任何 plugin（包括自家），autoload 沒跑 → namespace 對不到。

**解法**：只加 `--skip-themes`（避主題輸出 HTML 干擾 stdout），不加 `--skip-plugins`：
```bash
wp --skip-themes eval '...'
```

**教訓**：要跑自家 plugin 的 service 程式時，記得 plugin 不能 skip。如果有第三方 plugin 噪音很大，用 `--skip-plugins=other-plugin1,other-plugin2` 白名單排除而非全 skip。

---

## BUYGO-004：`wp plugin install --force` 不會清 PHP opcache，部署後仍跑舊代碼

**問題**：升級 plugin 到 v1.7.9 後 wp-cli 跑 service 回傳正確值，但瀏覽器打開 buyers 頁仍顯示舊資料（0 件已分配）。

**根因**：`wp plugin install --force` 只覆蓋檔案，OS / OPcache 仍把舊版 PHP class 留在記憶體。FPM workers 處理 HTTP 請求時用記憶體中的舊版 → 服務舊邏輯。

**解法**：部署完強制清三層快取：
```bash
wp --skip-themes eval 'if(function_exists("opcache_reset"))opcache_reset();'
wp --skip-themes cache flush   # WP object cache
# 必要時 systemctl reload php-fpm 完全 reload workers
```

**教訓**：
- BuyGo+1 部署 SOP 加入「升級後必跑 opcache_reset」一步
- wp-cli 在 CLI 進程跑 → 不命中 FPM opcache（所以 wp-cli 看到新版、瀏覽器看到舊版）。永遠用瀏覽器或 curl 線上 endpoint 驗收，不要只信 wp-cli

---

## BUYGO-005：Eloquent Collection `->toArray()` 後 callback 仍用物件存取會 silent null

**問題**：v1.7.9 剛部署，service 層 wp eval 跑 `calculateAllocatedPerParentOrder` 拿到正確結果，但 service wrapper `getProductBuyers` 整個回傳 0 件。PHP error log 抓到：
```
Warning: Attempt to read property "order" on array in class-product-buyer-query-service.php on line 44
```

**根因**：
```php
$parentOrderIds = array_values(array_unique(array_filter(array_map(
    static fn($item) => $item->order ? (int) $item->order->id : null,
    $orderItems->toArray()   // ← 序列化成 array of arrays
))));
```
`$orderItems` 是 Eloquent Collection；`->toArray()` 把每個 model 變 array。callback 內 `$item->order` 用物件 syntax → PHP 8 raise warning + 回 null → 全部 filter 掉 → 空陣列 → allocatedMap 空 → 每筆 allocated=0。

PHP 7 這會 silent succeed（回 null），PHP 8 升 warning 但功能仍壞，PHP 9 會 fatal。

**解法**：用 Collection idiom，保持 model 物件存取：
```php
$parentOrderIds = $orderItems->pluck('order.id')
    ->filter()
    ->map(static fn($id) => (int) $id)
    ->unique()->values()->toArray();
```

**教訓**：
- Eloquent Collection 操作鏈不要中途切換到 PHP 原生 array — 要嘛全 collection（pluck/map/filter），要嘛先 `->toArray()` 再用 array_* + 陣列存取（`$item['order']['id']`）
- 單元測試直接 mock collection 物件，沒打到「真實 toArray」路徑 → 漏抓。整合測試或線上實機驗收才是把關
- PHP 8 起，**任何 array vs object 存取錯誤都會 raise warning 但繼續執行**，造成 silent failure；error_log 一定要看
