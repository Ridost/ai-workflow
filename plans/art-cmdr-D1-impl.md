# art-cmdr D1 實作計畫

- 規格正本：`art-cmdr/DESIGN.md`（本計畫只排順序、交付物與完成條件；細節不重抄，衝突時以 DESIGN 為準並回報）
- 執行環境：擁有者的 Windows 筆電，Claude Code（DESIGN D8）。repo：`Ridost/ai-workflow`，目錄 `art-cmdr/`
- 分支：`art-cmdr-v1`，每個 milestone 一個或多個 commit；不 push，交付時由主 session 走 deliver review
- 驗收：另一個 agent 依 `plans/art-cmdr-D1-accept.md` 進行，本計畫的實作 agent 不做驗收

## 0. 開工前（實作 agent 先做、先回報）

1. 讀 `art-cmdr/DESIGN.md` 全文。
2. 確認筆電環境，記到 `art-cmdr/docs/env-check.md`：`python --version`（≥3.11）、`node --version`（22.x）、`ffmpeg -version`、`ffprobe -version`、`tailscale version`、`tailscale status`、`git --version`。缺什麼就停下回報，不要自行換版本或改用別的工具。
3. 確認 `tailscale serve` 可以給出 HTTPS 網址（tailnet 要啟用 MagicDNS 與 HTTPS certificates）；不行就回報，不改用 funnel 或 port forwarding。

## 1. Milestones

每個 milestone 結束都要：單元測試全綠、`git commit`、在報告記下 commit SHA 與測試指令輸出摘要。

### M1 核心 server + cmdr（不含媒體）

- `artcmdr/store.py`：原子寫入（暫存檔 + `os.replace`）、單一 lock、流水號 Q/R/C/L、type 模板驗證（DESIGN §4 骨架，含可選 explore）。
- `types/art.json`、`types/dev.json`。
- `artcmdr/ops.py` + `api.py`：DESIGN §3–§5 的 task/item/stage/log/question（choice、open、gate）/answer/poll/reply/watch/chat/recall；gate 強制（§4 規則 1）；task/item 結束、終止、重開；寫入守門（非 active task、cancelled item 回 409）。
- token 驗證（Bearer 與 cookie）、只聽 127.0.0.1。
- `cmdr`（標準庫）+ `cmdr.cmd`：§6 中與 ai-commander 相同的指令與結束碼。
- 測試：`tests/test_store.py`、`test_ops.py`、`test_api.py`、`test_cmdr.py`（含 wait 0/2/3/4/1 五種結束碼、gate 409）。
- 端到端：`tests/e2e_core.py`：開 task → item → ask → 用 API 模擬回答 → wait 拿到 → 追問 → reply → 再答 → qa gate → stage 推進。

### M2 媒體管線

- `artcmdr/media.py`：`POST /api/media`（原始 body，串流寫入、計算 sha256）、去重、`meta.json`、ffprobe 偵測、背景轉檔佇列（DESIGN §8 表格每一列）、`GET /media/<mid>/<variant>`（Range、ETag、immutable cache、nosniff）。
- `POST /api/media/sprite`、`cmdr media add|sprite`、`cmdr fetch`；`ask --media/--option-media` 支援路徑自動上傳。
- 專案設定讀取 `~/.art-cmdr/projects/*.json`、`GET /api/projects`；`file_roots` 白名單（Windows 路徑正規化、symlink、`..`）。
- 測試：每種輸入各一個 fixture（小 PNG 含透明、GIF、短 MP4、OGG、WAV、GLB、Phaser JSON array sprite），驗衍生檔存在、格式與尺寸正確（ffprobe）；Range 回 206 且位元組正確；白名單拒絕 `..`、別的磁碟、`~/.art-cmdr/token`。fixtures 由測試程式產生或放 `tests/fixtures/`（總大小 < 2 MB）。

### M3 explore 與 acceptance 題型

- question kind `explore`（`cmdr explore`，6–8 張卡、min/max 驗證）與 `acceptance`（`cmdr accept-report`）；回答格式與驗證照 DESIGN §3。
- deliver gate 前置檢查（§4 規則 3）。
- `cmdr wait` 輸出多 `picks`、`notes`、`items`、`media`。
- 測試：picks 數量越界 400、未全部表態 400、退回項無意見 400、deliver 在驗收未全過時 409。

### M4 網頁 PWA（手機優先）

- `web/`：Vite + Preact + TS；頁面照 DESIGN §9 表格；在 375×812 與 412×915 視口可用，不出現橫向捲動（diff 區塊除外）。
- 媒體檢視器：圖片（棋盤格、雙指放大）、影片、音效 A/B（切換時保留播放位置）、sprite（canvas、anim 選擇、0.25×–2×、逐幀）、`<model-viewer>`（npm `@google/model-viewer`）、尺度預覽（依 `scale_ref`）。
- 回答：文字、附件上傳（進度、重試）、錄音（MediaRecorder）。
- manifest、圖示、service worker（DESIGN §9 快取策略）、離線提示。
- 先交 `art-cmdr/docs/mockups/` 的手機版 mockup（假資料 HTML，收件匣、探索頁、驗收頁、決策頁四張），由主 session 送擁有者看；擁有者同意版面後再接真資料。
- 測試：元件單元測試（vitest）至少涵蓋：explore 選取上限、acceptance 全部表態才可送出、sprite 幀計算、音效 A/B 位置同步；Playwright（手機視口）跑一次端到端：收件匣 → 探索頁選 2 張 + 意見 → 送出，wait 拿到 picks。

### M5 Web Push

- `artcmdr/push.py`：首次啟動產生 VAPID（`push/vapid.json`）、`/api/push/key|subscribe|test`、事件觸發（新 question 任何 kind、主 session 在對話回覆）；訂閱失效（404/410）自動刪除。
- 網頁設定頁「開啟通知」「測試通知」；點通知開到該題。
- `requirements.txt`：`pywebpush`（鎖版本）。
- 測試：單元測試 mock 推送端點，驗 payload（task、題目、URL）與失效清理；實機測試留給驗收。

### M6 deliver review

- `artcmdr/review.py`：DESIGN §10 唯讀 git 規則；`cmdr gate ... deliver --review <repo> [--base] [--head] [--push <remote>:<branch>]` 記錄 `~/.art-cmdr/reviews/`；`cmdr push` 只推審查時的 head SHA、遠端分支必須是 base 的祖先、不加 `--force`。
- 網頁審查頁（手機版 diff、逐行意見、Push／退回）。
- 測試：在暫存 repo 做 base/head，驗 diff API、ref 移動後 push 拒絕、遠端多出 commit 時拒絕。

### M7 Windows 部署與 session 整合

- `ops/install.ps1`、`ops/uninstall.ps1`、`ops/README.md`（DESIGN §11）；Task Scheduler 登入啟動；`tailscale serve` 步驟。
- `launch.py`：`cmdr resume` 與網頁「在背景重啟 session」（`claude --bg --permission-mode plan`）；`cmdr sessions`（解析 `claude agents --json`）。Windows 上若 `claude --bg` 或 `claude agents` 不可用，記到 `docs/env-check.md` 並讓功能回 503 與清楚訊息，不硬做。
- `skill/art-cmdr/SKILL.md`：cmdr 用法、explore／accept-report 用法、agent brief 範本（實作 agent、驗收 agent）、「定案寫入專案帳本並附題號」（DESIGN D12）。
- `art-cmdr/README.md`：安裝、啟動、手機加入主畫面與開通知的步驟。
- 筆電 `~/.claude/CLAUDE.md` 要加的幾行規則寫成 `ops/claude-md-snippet.md`，**不直接改**擁有者的 CLAUDE.md；由主 session 請擁有者確認後再加。

## 2. 不做

- 不改 snowbell-town repo（它的 AGENTS.md §1-1 改寫另開 task 處理）。
- 不複製其他 repo 的原始碼（DESIGN D1）。
- 不使用 `tailscale funnel`、不對公網開 port。
- DESIGN §12 列出的 v2 功能。

## 3. 回報

- 每個 milestone 用 `cmdr`（若 art-cmdr 還沒能用，先用主 session 指定的方式）回報 build stage；完整報告寫 `art-cmdr/docs/build-report.md`：每個 milestone 的 commit、測試指令與實際輸出摘要、未做與已知缺口。
- 遇到 DESIGN 沒寫到、需要擁有者決定的事：停下問，不自行補完。
