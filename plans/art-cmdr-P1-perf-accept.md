# art-cmdr P1 手機載入速度 驗收計畫

- 規格：`art-cmdr/DESIGN.md` §9（速度目標與快取策略）；實作計畫 `plans/art-cmdr-P1-perf-impl.md`；成果位置由主 session 給（分支 `art-cmdr-perf` 的 commit）
- 驗收 agent 只讀本計畫、DESIGN、實作計畫與成果；不讀實作過程
- 每項寫 PASS/FAIL 與證據，結果寫到 `plans/art-cmdr-P1-perf-accept-result.md`；FAIL 不修，回報主 session
- 標「手機」的項目由擁有者用 Android 手機配合，以手機截圖、`chrome://inspect` 錄製或擁有者在題目上的回答為證據
- 改前數據以 `art-cmdr/docs/perf-baseline.md` 為準

## A. 自動測試

| # | 檢查 | PASS 條件 |
|---|---|---|
| A1 | server／cmdr 單元測試 | `python -m unittest discover -s art-cmdr/tests -v` 0 failure、0 error，含實作計畫 §2 新增的併發、快取標頭、壓縮、304 測試 |
| A2 | 網頁單元測試 | `npm test` 全綠，含 stale-while-revalidate、草稿不被覆蓋、離線停用送出 |
| A3 | 既有端到端 | `python art-cmdr/tests/e2e_core.py` exit 0；P1 前的 Playwright 端到端仍通過 |

## B. server 行為（驗收 agent 用 curl 自己操作）

| # | 檢查 | PASS 條件 |
|---|---|---|
| B1 | 併發 | 一個 `curl` 以限速（`--limit-rate 50k`）下載 `preview.mp4`、另開一個 `cmdr wait` 進行中時，`GET /api/tasks` 回應時間 < 200 ms（`curl -w '%{time_total}'`，連 5 次） |
| B2 | 快取標頭 | `/assets/*.js` 帶 `immutable`；`index.html`、`sw.js` 為 `no-cache` |
| B3 | 壓縮 | `curl -H 'Accept-Encoding: br, gzip' -I` 取首屏 JS 回 `Content-Encoding: br`（或 gzip）與 `Vary: Accept-Encoding`；媒體不壓縮 |
| B4 | 304 | 帶上次的 `ETag` 再 `GET /api/tasks` 且資料未變 → 304 |
| B5 | 收件匣單一請求 | 收件匣首屏除 `GET /api/tasks`（與 shell／assets）外不發其他 `/api/*` 請求（看 `server.log`）；回應不含 `context_md`、`thread` 全文；預設不含 done／cancelled task |
| B6 | bundle 預算 | `npm run build` 後首屏 JS gzip 合計 ≤ 150 KB；`model-viewer` 在獨立 chunk，首屏不載入 |

## C. 手機實機（擁有者配合，Android Chrome PWA，經 Tailscale）

| # | 檢查 | PASS 條件 |
|---|---|---|
| C1 | 熱開啟 | 從主畫面開 PWA 共 5 次（家中 Wi-Fi 3 次、行動網路 2 次）：每次 1 秒內看到收件匣內容、2 秒內顯示最新資料（`chrome://inspect` 錄製為證，至少 1 次 Wi-Fi + 1 次行動網路） |
| C2 | 冷開啟 | 清除網站資料後於行動網路開啟，3 秒內可操作 |
| C3 | 開題目 | 從收件匣點開一題 choice、一題 explore、一題 acceptance：0.5 秒內出現文字與縮圖佔位；原檔／影片未點開前不下載（`server.log` 為證） |
| C4 | 快取不亂寫 | 決策頁輸入草稿時背景資料更新，草稿與游標不變；選項已被 agent 改變時有提示而不是悄悄換掉 |
| C5 | 離線 | 筆電停 server 後開 PWA：顯示「筆電離線」與上次資料，送出按鈕停用；server 起來後自動恢復 |
| C6 | 新版 | 部署新 build 後，舊頁面不被強制 reload；看得到「有新版」提示，按下後載入新版 |
| C7 | 擁有者主觀 | 擁有者確認：日常開 art cmdr 做 QA 與驗收「不再覺得要等很久」（gate 題回答） |

## D. 數據對照

| # | 檢查 | PASS 條件 |
|---|---|---|
| D1 | 前後對照 | `art-cmdr/docs/perf-result.md` 有與 `perf-baseline.md` 同方法的量測，列出熱開啟、冷開啟、請求數、首屏 JS 大小的前後數字；驗收 agent 抽 1 項自己重量，誤差 < 20% |
| D2 | 未做項目 | 實作計畫 §1 每一項都有結論：已改（commit）或已確認不是瓶頸（附數據） |
| D3 | 範圍 | 未改筆電網路／防火牆／Tailscale 設定；未用 `tailscale funnel`；未改 snowbell-town |
