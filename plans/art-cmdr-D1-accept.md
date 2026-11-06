# art-cmdr D1 驗收計畫

- 規格：`art-cmdr/DESIGN.md`；成果位置由主 session 給（分支 `art-cmdr-v1` 的 commit）
- 驗收 agent 只讀本計畫、DESIGN 與成果；不讀實作報告（`docs/build-report.md`）或實作過程
- 每項寫 PASS/FAIL 與證據（指令與輸出、截圖路徑、檔案:行），結果寫到 `plans/art-cmdr-D1-accept-result.md`
- FAIL 不修，回報主 session
- 標「手機」的項目要擁有者用 Android 手機配合：驗收 agent 準備好題目後，請擁有者操作，並以擁有者在題目上的回答與手機截圖作為證據

## A. 自動測試

| # | 檢查 | 方法 | PASS 條件 |
|---|---|---|---|
| A1 | server 與 cmdr 單元測試 | `python -m unittest discover -s art-cmdr/tests -v` | 0 failure、0 error；測試數 ≥ 實作計畫列出的檔案皆有 |
| A2 | 網頁單元測試 | `cd art-cmdr/web && npm ci && npm test` | 全綠，含 explore 上限、acceptance 全部表態、sprite 幀計算、音效 A/B 位置同步 |
| A3 | 網頁建置 | `npm run build` | 0 error；`web/dist` 含 manifest 與 service worker |
| A4 | 核心端到端 | `python art-cmdr/tests/e2e_core.py` | exit 0 |
| A5 | Playwright 手機視口端到端 | 實作提供的指令 | 412×915 視口下 explore 選 2 張 + 意見送出，`cmdr wait` 回 picks 與 notes |

## B. 合約與規則（驗收 agent 自己用 cmdr／curl 操作，不只看測試）

| # | 檢查 | PASS 條件 |
|---|---|---|
| B1 | 骨架驗證 | 把一個壞模板（plan 在 qa 前）放進 `types/` 後 server 拒絕啟動並說明原因；移除後正常啟動 |
| B2 | gate 強制 | `art` 類型 item 未核准 qa 就 `cmdr stage ... plan running` → 409 且訊息含要送的指令；explore 未核准就推到 plan → 409 |
| B3 | wait 結束碼 | 分別造出 0（決定）、2（追問）、3（`--max 5s` 逾時）、4（task cancel）、1（server 停止）並記錄實際 exit |
| B4 | explore 驗證 | 卡片少於 6 或多於 8 → 400；picks 少於 min 或多於 max → 400；合法回答後 wait JSON 含 `picks`、`notes` |
| B5 | acceptance 驗證 | 未全部表態送出 → 400；退回項無意見 → 400；有一項退回時 `choice == reject`；全過時 `approve` |
| B6 | deliver 前置 | 驗收題未全過時送 deliver gate → 409；全過後可送 |
| B7 | recall／watch／chat | agent 未在等時擁有者追問 → `cmdr watch` 回 0 且 JSON 含 recall；網頁對話留言 → watch 回 chats；`cmdr say` 後狀態 taken |
| B8 | 路徑白名單 | `cmdr media add` 或 `/api/file` 讀 `..` 跳出、另一磁碟、`~/.art-cmdr/token`、`.git/config` → 拒絕；`file_roots` 內檔案 → 成功 |
| B9 | token | 無 token、錯 token 的 API 請求 → 401；cookie 帶 `HttpOnly`、`Secure`、`SameSite=Strict` |
| B10 | 只聽本機 | `netstat -ano` 顯示 8765 只綁 127.0.0.1 |
| B11 | deliver review 與 push | 暫存 repo：審查後移動 head → `cmdr push` 拒絕；遠端多出 commit → 拒絕；正常情況推送的是審查時的 SHA，未用 `--force` |

## C. 媒體

| # | 檢查 | PASS 條件 |
|---|---|---|
| C1 | 衍生檔 | 對 DESIGN §8 每種輸入各上傳一個：PNG 有 `thumb.webp`（長邊 480）；GIF 有 `preview.mp4`（H.264）與 poster；1080p MP4 有 720p `preview.mp4`；WAV 有 `audio.opus`；GLB 與 OGG 原檔可取；以 ffprobe 輸出為證 |
| C2 | 去重 | 同一檔上傳兩次，回同一 `M<id>`，媒體庫只有一份 |
| C3 | Range | `curl -r 0-99` 取 `preview.mp4` → 206、100 bytes、`Content-Range` 正確 |
| C4 | 快取標頭 | `/media/*` 有 `ETag`、`Cache-Control` 含 `immutable`、`X-Content-Type-Options: nosniff` |
| C5 | SVG 不 inline | 上傳 SVG 後以 `attachment` 或 sandbox 方式提供 |
| C6 | snowbell 實檔 | 用 snowbell-town 的 `public/sprites/gold/gold.png`＋`gold.json`＋`gold-anims.json` 建 sprite media，網頁可選 `gold_walk_down` 播放；用一張 `public/assets/props/*.png` 開尺度預覽，比例尺為 76px |

## D. 手機實機（擁有者配合，Android Chrome，經 Tailscale）

| # | 檢查 | PASS 條件 |
|---|---|---|
| D1 | 連線 | 手機開 `https://<筆電>.<tailnet>.ts.net/`，輸入 token 後進收件匣；行動網路（非家中 Wi-Fi）下也可開 |
| D2 | 安裝 PWA | 加到主畫面，以 standalone 開啟 |
| D3 | 推播 | 開啟通知後，驗收 agent 送一題 → 手機 30 秒內收到通知，點擊直接開到該題 |
| D4 | choice 題 + 媒體 | 選項各附一張圖，擁有者在手機上看圖、選擇並附一張相簿照片與一段錄音回答；`cmdr wait` JSON 含 media，`cmdr fetch` 取回的檔案可開 |
| D5 | explore | 7 張候選卡（含圖、GIF、影片、音效各至少一張），擁有者選 2 張並寫意見；全螢幕可左右滑動切換 |
| D6 | acceptance | 4 項驗收（含 1920×1080 錄影與截圖），擁有者逐項表態（至少 1 項退回並附意見）；wait JSON 的 items 與手機上一致 |
| D7 | 媒體檢視器 | 擁有者確認：圖片透明處棋盤格、雙指放大；影片可拖曳進度（Range）；音效 A/B 切換保留位置；sprite 播放與調速；3D 模型可旋轉縮放；尺度預覽顯示比例尺 |
| D8 | 載入表現 | 含 6 張圖 + 2 段影片的頁面：首次開啟只下載縮圖（以 server log 或 DevTools 為證）；第二次開啟媒體由 service worker 快取提供 |
| D9 | 離線提示 | 筆電關閉 server 後，手機顯示「筆電離線」而非空白或錯誤頁 |
| D10 | deliver 審查 | 手機上看 diff、留一則逐行意見、按退回；再送一次按 Push，agent 用 `cmdr push` 推送成功 |
| D11 | 版面 | 擁有者確認四個主要頁面（收件匣、探索、驗收、決策）在手機上可順暢操作，沒有橫向捲動（diff 區除外） |

## E. 部署與文件

| # | 檢查 | PASS 條件 |
|---|---|---|
| E1 | 安裝腳本 | 在乾淨的使用者目錄（暫時改 `USERPROFILE` 或新帳號）跑 `ops/install.ps1` 成功；重跑不壞（idempotent） |
| E2 | 自動啟動 | 登出再登入後 server 自動起來（`cmdr show` 成功） |
| E3 | 文件 | `README.md` 照著做可以完成 D1–D3；`skill/art-cmdr/SKILL.md` 涵蓋 explore、accept-report、media、帳本寫入規則 |
| E4 | 不改擁有者設定 | 擁有者的 `~/.claude/CLAUDE.md` 未被修改；`ops/claude-md-snippet.md` 存在 |
| E5 | 範圍 | 未改 snowbell-town repo；未使用 `tailscale funnel`；repo 內無複製自其他 repo 的原始碼（擁有者抽查即可） |
