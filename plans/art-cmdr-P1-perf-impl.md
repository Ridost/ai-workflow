# art-cmdr P1 手機載入速度 實作計畫

- 起因：擁有者在手機上開 art cmdr 跟 agent 做 QA 討論與驗收，每次開都要載入很久（2026-11-12）。
- 規格正本：`art-cmdr/DESIGN.md` §9（本次已改：API 讀取走 stale-while-revalidate、速度目標、首屏 JS 預算）。衝突時以 DESIGN 為準並回報。
- 執行環境：擁有者的 Windows 筆電，Claude Code（DESIGN D8）。在 `art-cmdr-v1` 上開分支 `art-cmdr-perf`；不 push，交付走 deliver review。
- 驗收：另一個 agent 依 `plans/art-cmdr-P1-perf-accept.md` 進行。
- 原則：**先量再改**。下面「可能原因」是推測（寫計畫時看不到實作程式碼），每一項都要先用 §0 的數據確認是不是瓶頸；不是瓶頸的就記錄「已確認不是問題」並跳過，不要為了改而改。

## 速度目標（DESIGN §9）

| 情境 | 目標 |
|---|---|
| 熱開啟：PWA 已安裝、service worker 已啟用，從主畫面點開 | 1 秒內畫出收件匣（可為快取資料）；2 秒內換成最新資料 |
| 冷開啟：清除網站資料後第一次開，4G | 3 秒內可操作 |
| 開一題（決策／探索／驗收頁），題目已在快取 | 0.5 秒內畫出文字與縮圖佔位 |
| 首屏 JS（gzip 後） | ≤ 150 KB |

## 0. 量測（先做、先回報，不改程式）

把結果寫到 `art-cmdr/docs/perf-baseline.md`，每項附原始輸出。

1. **Tailscale 路徑**：筆電上 `tailscale status` 與 `tailscale ping <手機的 tailscale 名稱>`（手機分別在家中 Wi-Fi、行動網路各一次）。記錄是 `direct` 還是 `via DERP(<地區>)` 以及延遲。走 DERP 時每個往返可能 100–300 ms 以上，這會決定「少幾次往返」有多重要。
2. **server 端耗時**：暫時在 `api.py` 每個請求記 `method path status bytes ms` 到 `server.log`（這個記錄保留，見 §1-1）。在手機熱開啟、冷開啟各 3 次，列出每次開啟發出的請求清單（數量、順序、各自耗時、大小）。
3. **前端**：手機 USB 接筆電，Chrome `chrome://inspect` 遠端偵錯，對熱開啟與冷開啟各錄一次 Performance + Network，記 FCP、「收件匣有內容」的時間點、JS/CSS 總量（傳輸量與解壓後）、service worker 是否真的回應了 shell（Network 的 Size 欄顯示 `(ServiceWorker)`）。
4. **bundle 組成**：`npm run build` 後列出 `web/dist/assets` 每個檔案大小（原始與 gzip）；用 `rollup-plugin-visualizer`（只在量測時加，或用 `vite build --sourcemap` + `source-map-explorer`）找出最大的幾個依賴。
5. **server 併發**：確認 server 是 `HTTPServer` 還是 `ThreadingHTTPServer`。有 agent 正在 `cmdr wait`（輪詢）或手機正在串流影片時，同時開收件匣，`GET /api/tasks` 是否被卡住。
6. **資料量**：`~/.art-cmdr/tasks/` 檔案數與總大小、最大的 task JSON 大小、`GET /api/tasks` 回應大小與耗時。

量完先停，用 `cmdr log` 回報摘要（前三大耗時來源 + 對應 §1 哪幾項），主 session 確認後再動手。

## 1. 可能原因與改法（依 §0 數據挑選，依影響排序）

### 1-1 server 一次只能處理一個請求

- 症狀：§0-5 中收件匣請求要等其他請求（影片串流、`cmdr wait` 輪詢、ffmpeg 轉檔觸發的請求）結束才回。
- 改法：改用 `http.server.ThreadingHTTPServer`（仍是標準庫，符合 D2），`daemon_threads = True`；確認 `store.py` 的單一 lock 涵蓋所有讀寫，讀取不在 lock 內做慢的事（ffprobe、git、檔案掃描）。
- `protocol_version = "HTTP/1.1"` 並對每個回應正確設 `Content-Length`（keep-alive）；做不到的回應明確 `Connection: close`。
- 請求耗時記錄保留在 `server.log`（一行一請求），之後排查用。

### 1-2 每次開啟都要等網路才畫得出畫面

- 症狀：§0-3 中熱開啟時 `index.html`、JS 不是從 service worker 來，或 shell 雖從快取來但畫面一直是 loading 直到 `/api/tasks` 回來。
- 改法（DESIGN §9 已改）：
  - service worker precache `index.html` 與帶 hash 的 assets；導航請求回快取的 `index.html`（app shell），背景更新。新版 service worker 啟用時不強制 reload 正在輸入的頁面；有新版時在設定頁或頁首顯示「有新版，重新整理」。
  - `GET /api/tasks`、`GET /api/tasks/<t>` 走 stale-while-revalidate：快取（Cache Storage 或 IndexedDB）裡有就立刻畫，頁首顯示「更新中」，新資料到了再替換；草稿與輸入中狀態不能被替換打斷（沿用 DESIGN §9 輪詢規則）。
  - 快取資料只供顯示：網路回不來時是「筆電離線」狀態，送出按鈕停用。
  - token 登出時清掉 API 快取。

### 1-3 首屏 JS 太大

- 症狀：§0-4 首屏 JS gzip 後 > 150 KB，或 §0-3 冷開啟的下載／解析時間明顯。
- 最可能的大頭：`@google/model-viewer`（含 three.js，數百 KB 以上）。改成只有在媒體檢視器打開 3D 模型時才 `import()`。
- 其他非首屏功能也拆成動態 import：diff 審查頁、錄音（MediaRecorder 相關）、sprite 播放器、markdown 渲染器若很大也一樣。
- Vite 輸出的 assets 帶 hash；server 對 `/assets/*` 回 `Cache-Control: public, max-age=31536000, immutable`，對 `index.html`、`sw.js`、`manifest.webmanifest` 回 `no-cache`。

### 1-4 沒有壓縮

- 症狀：§0-3 JS/CSS/JSON 的傳輸量 ≈ 解壓後大小（`http.server` 預設不壓縮，`tailscale serve` 也不會幫你壓）。
- 改法：build 時用 `vite-plugin-compression`（或建置後腳本）產生 `.br` 與 `.gz`；server 依 `Accept-Encoding` 回預壓縮檔並加 `Vary: Accept-Encoding`。`/api/*` JSON 回應大於 1 KB 時用 `gzip`（標準庫）即時壓縮。媒體（WebP、MP4、Opus）不壓。

### 1-5 收件匣要發很多請求或回應太大

- 症狀：§0-2 開收件匣時發出 N 個 `GET /api/tasks/<t>`（每個 task 一個），或 `GET /api/tasks` 回應很大／很慢。
- 改法：
  - `GET /api/tasks` 一次回收件匣需要的全部摘要（DESIGN §9 收件匣三區：需要你、進行中、agent 等待中）：每題只帶 `id, kind, title, status, item, at`，不帶 `context_md`、`thread`、`options` 全文；收件匣不需要再發其他 API 請求。
  - 預設不含 `done`／`cancelled` task（可用 `?all=1` 取得）。
  - server 端依檔案 mtime 快取每個 task 的摘要，不要每次請求都讀全部 task JSON 與 `.log.jsonl`。
  - 回應加 `ETag`；前景 5 秒輪詢帶 `If-None-Match`，沒變回 304。
- 開一題的頁面：題目內容已在 `GET /api/tasks/<t>` 時就先顯示；媒體只先載縮圖（DESIGN D5），原檔／影片在點開時才載。

### 1-6 其他首屏阻塞

- 外部字型、CDN 腳本、外部圖示：全部改成本機打包或系統字型（從 Tailscale 網址開時，外部資源多一次 DNS + TLS）。
- 開啟時 `GET /api/push/key`、`/api/projects` 等非首屏必要請求延後到首屏畫完之後。
- 登入檢查若會先打一次 API 再導向：改成直接帶 cookie 打 `/api/tasks`，401 才顯示登入。

### 1-7 Tailscale 連線（不是程式問題，但要回報）

- 若 §0-1 顯示行動網路下一直 `via DERP`：記錄在 `docs/perf-baseline.md`，並把可行做法列給擁有者決定（例如 Tailscale 文件中改善直連的設定），**不要自行修改筆電的網路、防火牆或 Tailscale 設定**。
- 手機上 Tailscale 若常被系統省電關掉導致每次要重連：同樣只記錄、回報。

## 2. 交付物

- 每項改動一個 commit，訊息寫是哪一項（如 `perf 1-1: ThreadingHTTPServer`）。
- 測試：
  - `tests/test_api.py` 加：一個長時間請求（模擬 `cmdr wait` 輪詢或 Range 串流）進行中時，`GET /api/tasks` 仍在 200 ms 內回；`/assets/*` 與 `index.html` 的 `Cache-Control`；`Accept-Encoding: br, gzip` 時回壓縮檔與 `Vary`；`If-None-Match` 回 304。
  - web 單元測試加：stale-while-revalidate 先回快取再更新、更新時草稿不被覆蓋、離線時送出按鈕停用。
  - 既有測試全部仍要綠。
- `art-cmdr/docs/perf-baseline.md`（改前）與 `art-cmdr/docs/perf-result.md`（改後，同樣方法重量一次，列對照表）。
- 若 bundle 預算或速度目標做不到：在 `perf-result.md` 寫清楚卡在哪、還差多少，停下回報，不要放寬目標。

## 3. 不做

- 不換技術棧（D2）、不加反向代理或 CDN、不用 `tailscale funnel`。
- 不改筆電的網路／防火牆／Tailscale 設定（只回報建議）。
- 不改 snowbell-town repo。
