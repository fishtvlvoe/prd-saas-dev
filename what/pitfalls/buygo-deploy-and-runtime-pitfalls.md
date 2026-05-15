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

---

## BUYGO-006：改 REST API 行為時只追 service caller 不夠，必須從前端 URL 往後 trace

**問題**：fix-shipment-details-expand-variations（v1.7.11）改了 `ShipmentService::get_shipment_items()` 加 LEFT JOIN `fct_product_variations`，PR merged + 部署、wp eval 跑 service 直接 OK、單元測試 351/351 全綠。但實際打開出貨明細 UI 仍不顯示子列。

**根因**：前端 `useShipmentDetails.js::loadShipment(id)` 打的是 `/shipments/{id}/detail` endpoint，不是 `/shipments/{id}`。前者走 `Shipments_API::get_shipment_detail()` 含**獨立 inline SQL**（不呼叫 service）；後者走 `get_shipment` 才呼叫 `ShipmentService::get_shipment_items()`。改了後者，前者沒動 → 前端拿不到 variation 欄位 → `mergeItemsByProduct` 算出 subItems 全 variation_id=null 被合併成 1 個 → length=1 → 子列不渲染。

**解法**：開 hotfix v1.7.12 (PR #16)，把 `get_shipment_detail()` 內的 inline SQL 也加 LEFT JOIN `fct_product_variations` 拉 variation 三欄位。

**教訓**：
- Cross-impact 預檢只 grep service method caller 不夠。**MUST 加一步：從前端打的 URL 往後 trace** — `grep "fetch.*shipments"` / `grep "/shipments/.*}/detail"` 找實際被呼叫的 REST endpoint URL → 對應 controller method → 改對 handler。
- WordPress REST API 經常有「list endpoint vs detail endpoint」兩條獨立 SQL（為了 detail 多帶 customer 資訊或其他 JOIN）。改 list 不會影響 detail。
- 單元測試用 mock $wpdb 直接打 service method，這條路 100% 過。但實際 production 用戶走的是 REST endpoint。**驗收 MUST 用真實 endpoint**（curl / 瀏覽器網頁 / Chrome MCP）不能只信單元測試。
- L079 已補強規則：「改 REST API 行為前 MUST 從前端往後 trace 確認改對 handler」。
