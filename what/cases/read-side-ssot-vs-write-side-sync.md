# 讀取端 SSOT vs 寫入端 Sync — BuyGo+1 已分配統計案例

## 情境

BuyGo+1 把同一份語意「已分配」拆寫到三處儲存：

1. **child orders**：`fct_orders` 中 `parent_id IS NOT NULL AND type='split'` 的 row，每筆代表一次分配
2. **parent line_meta 快照**：`fct_order_items.line_meta._allocated_qty`（JSON 欄）
3. **product post_meta 快照**：`wp_postmeta._buygo_allocated`

寫入端有 4-5 個 service（AllocationWriteService、ShipmentService、AllocationBatchService 等）會 mutate child orders，但歷史路徑漏 sync 後兩個快照。結果三層發散：

- product 1055：`_buygo_allocated=''`、parent line_meta 加總=0、但 2 個 shipped child orders qty 加總=2
- product 1258：三層都是 52（一致）

讀取端三條獨立路徑各挑一處讀：
- allocation 詳情頁 → 從 child orders 算（看到 2 ✓）
- buyers 頁 → 加總 parent line_meta（看到 0 ✗）
- products 列表頁 → 讀 post_meta（看到 0 ✗）

業主反映「三個畫面數字對不起來」。

## 選擇

**讀取端 SSOT**：所有讀取點改成從 child orders 即時計算（單一真相來源），post_meta 與 parent line_meta 維持「寫入但不讀」的狀態，加 `@deprecated` 註記未來移除。

## 理由

1. **單一函式收斂風險**：把所有「讀已分配」收到 `ProductStatsCalculator::calculateAllocatedToChildOrders` + `calculateAllocatedPerParentOrder`。未來進新寫入路徑也不會破讀取。
2. **不需 backfill**：讀取改完，存量資料的快照漂移自動「看起來修好了」，不需碰 production DB。
3. **動作面小**：3 個讀取點改 vs 4-5 個寫入 service + backfill migration + cache busting。
4. **流量低**：BuyGo 後台同時在線 < 10 人，list query 多一個 GROUP BY join 仍 ms 級。

## 放棄的選項

**A：寫入端 Sync + Backfill Migration**

放棄理由：
- 寫入路徑分散在 4-5 個 service，全部 audit 加 sync 是工程帳
- backfill migration 對 production 風險高（SQL 寫錯可能改壞良性資料）
- 即使這次修好，未來進新寫入路徑仍可能漏 sync — 治標不治本

**B：保留 meta 讀取 + 加快取失效機制**

放棄理由：
- WordPress transient 無原生 wildcard 失效，要嘛改 `wp_cache_*` group + flush，要嘛自建版本號 key
- 寫入端仍要每處加 invalidation hook，分散風險
- BuyGo+1 流量低，省的 query 不抵「快取失效漏寫」的信任成本

**C：保留三層、改 read 邏輯加 max(child, line_meta, post_meta) 取最大**

放棄理由：
- 治標：取最大可能掩蓋資料漂移錯誤（例如 child 真實=0、meta 漂移=5，取 5 是錯的）
- 後續開發者看不出哪個是真相，認知負擔疊加

## 結果

- PR #11（拔 transient cache + 統一 reserved 公式）+ PR #13（讀取端 SSOT）+ PR #14（hotfix collection toArray bug）合進 main
- v1.7.10 部署到 buygo.me，product 1055 三頁面數字一致顯示 2，product 1258 迴歸 52
- 寫入端漏 sync 的歷史 bug **沒修也不會再影響顯示**（讀取不再依賴那兩個快照）
- 後續可獨立開 change 把寫入端 deprecated 欄位徹底刪掉，但不急

## 適用通則

**遇到「資料分散儲存、寫入路徑多、部分歷史漂移」型 bug 時：**

| 條件 | 偏好 |
|------|------|
| 流量低、讀取點 ≤ 5 | 讀取端 SSOT |
| 流量高、讀取每次重算成本 > 100ms | 寫入端 Sync + 強化 invalidation |
| 真相來源不存在（純快照堆疊） | 必須先建立真相來源（新 table 或計算邏輯） |
| 歷史資料量大 + 跨多 production | backfill 風險評估 → 偏 SSOT |

關鍵問題：**「真相在哪裡？」**先找出唯一可信來源，再決定讀寫策略。如果真相就是「事件記錄」（child orders / event log），讀取端從真相算永遠是最安全的解。
