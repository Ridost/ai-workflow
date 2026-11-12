# art-cmdr 設計

art-cmdr 是跑在個人 Windows 筆電上的 AI 任務控制台。筆電上的 Claude Code session 與它派出的 agent 用 `cmdr` 回報進度、提問、附媒體；擁有者用 Android 手機（PWA）從外網看進度、挑選視覺提案、核准 gate、逐項驗收，agent 拿到回答後自動繼續。

- 定案：2026-11-06，ai-commander task `art-cmdr` Q1–Q14（qa gate Q14 核准）
- 第一個使用專案：`Ridost/snowbell-town`（雪鈴鎮，1920×1080 桌面同機雙人遊戲，Phaser 4 + React + Vite，2D PNG／sprite sheet／OGG）
- 本文件是目標狀態。實作與本文件不一致時，以本文件為準並回報差異；要改設計，先改本文件（經擁有者同意）。

## 1. 決策紀錄

| # | 問題 | 決定 |
|---|---|---|
| D1 | 程式來源 | 依本文件的資料模型、API 與 `cmdr` 合約重新撰寫；不複製其他 repo 的原始碼 |
| D2 | 技術棧 | server：Python 3.11+（標準庫 `http.server` + `pywebpush`）；網頁：Vite + Preact + TypeScript，建置成 PWA，由 server 提供靜態檔 |
| D3 | 外網存取 | Tailscale。server 只聽 `127.0.0.1`，用 `tailscale serve` 提供 `https://<筆電>.<tailnet>.ts.net`（HTTPS 是 service worker 與 Web Push 的前提）；仍要 token |
| D4 | 媒體方向 | 雙向：agent 可附在問題、選項、候選卡、驗收項目；擁有者可從手機上傳圖、影片、錄音當回答。圖上標註延到 v2 |
| D5 | 媒體效能 | 原檔不直接給手機看：縮圖 WebP、影片轉 720p H.264 + 封面、GIF 轉 MP4、WAV 轉 Opus；支援 HTTP Range；service worker 快取看過的媒體 |
| D6 | 流程骨架 | `qa* → explore*（可選）→ plan* → build… → accept… → deliver*`（`*` = 擁有者 gate） |
| D7 | 通知 | PWA Web Push（Android Chrome），VAPID 金鑰在筆電本機產生 |
| D8 | 開發位置 | 設計與計畫在別處完成並 push 到本 repo；實作與驗收在筆電的 Claude Code 上進行 |
| D9 | explore v1 | 候選牆：6–8 張卡，複選 `min..max` 張（預設 2–3），每張可附一句意見。並排比較、多輪淘汰、混搭延到 v2 |
| D10 | 驗收 | 驗收 agent 跑 playtest，錄 1920×1080 影片與截圖；手機逐項顯示「預期／實際（文字＋媒體）／agent 判定 PASS・FAIL」，擁有者逐項同意或退回並附意見 |
| D11 | 媒體檢視器 | 圖片（透明棋盤格、雙指放大）、GIF／影片、音效（含 A/B 切換試聽）、sprite sheet 播放、尺度預覽（專案設定的比例尺）、glTF/GLB（`<model-viewer>`） |
| D12 | 與專案帳本分工 | dashboard 負責問答與進度；每個定案由 session 照該專案規矩寫進專案 repo 的帳本（snowbell：`ledger/owner-decisions.md`），並附題號，例如 `via art-cmdr <task>/Q12`。server 不寫專案 repo |
| D13 | 多專案 | 通用系統；專案專屬設定（比例尺、playtest 指令、帳本位置、檔案白名單）放在 `~/.art-cmdr/projects/<project>.json` |
| D14 | 狀態儲存 | `~/.art-cmdr/`（Windows：`%USERPROFILE%\.art-cmdr`），不進 git |
| D15 | 交付 | diff 審查頁 + Push 按鈕；按下只記決定，由 agent 執行 `cmdr push <task>/Q<n>` 推送到 GitHub，不加 `--force` |

## 2. 架構

```
ai-workflow/art-cmdr/          （本 repo）
  server.py                    入口：python server.py [--port 8765]
  artcmdr/                     server 套件
    store.py                   原子寫入 JSON、流水號、type 模板驗證
    ops.py                     業務邏輯（task/item/stage/question/recall/chat/gate 強制）
    api.py                     HTTP 路由、token 驗證、JSON 錯誤
    media.py                   上傳、內容定址儲存、ffmpeg 轉檔佇列、Range 回應
    push.py                    VAPID、訂閱、送通知（pywebpush）
    review.py                  唯讀 git diff 審查（deliver）
    files.py                   路徑白名單、檔案片段讀取
    launch.py                  背景重啟 claude session（M7）
  cmdr / cmdr.cmd              CLI（Python 標準庫；Windows 用 cmdr.cmd 包裝）
  web/                         Vite + Preact + TS 原始碼；build 輸出到 web/dist（不進 git）
  types/*.json                 任務類型 stage 模板
  ops/                         Windows 安裝、Task Scheduler 自動啟動、tailscale serve 說明
  tests/                       unittest（server、cmdr）+ web 單元測試 + 端到端腳本
  skill/art-cmdr/SKILL.md      Claude Code skill：cmdr 用法、agent brief 範本
  docs/                        本文件以外的說明、mockup

~/.art-cmdr/
  token                        網頁與 cmdr 共用；cmdr 用 Authorization: Bearer，網頁登入後存 HttpOnly cookie
  config.json                  {port, claude, ffmpeg, ffprobe, public_url}
  projects/<project>.json      專案設定（見 §7）
  tasks/<task>.json            task 狀態；tasks/<task>.log.jsonl 活動紀錄（只追加）
  media/<sha[0:2]>/<sha>/      原檔 orig.<ext>、meta.json、衍生檔（thumb.webp、preview.mp4、poster.webp、audio.opus）
  push/vapid.json              VAPID 金鑰（權限只限本人）
  push/subscriptions.json      手機訂閱
  reviews/<task>-<Q>.json      cmdr 本機的 review 紀錄（push 時比對）
  server.log
```

資料流：

- agent → `cmdr ...` → `http://127.0.0.1:8765/api/...`（筆電本機，不經 Tailscale）→ server → `~/.art-cmdr/`
- 手機 → `https://<筆電>.<tailnet>.ts.net/`（tailscale serve → 127.0.0.1:8765）→ 網頁／API
- 新問題、gate、驗收、主 session 回覆 → server 送 Web Push → 手機通知 → 點開到該題
- 擁有者回答 → 背景的 `cmdr wait` 結束 → Claude Code 喚醒該 session；沒人在等時記 recall → 主 session 的 `cmdr watch` 接手

## 3. 資料模型

```
task     { id, title, type, project, status: active|done|cancelled, created,
           links: [{label, url|path}], closed?, main_session?: {session_id, cwd, at},
           items: [item], questions: [question], recalls: [recall], chat: [chat], launches: [launch] }

item     { id, title, agent: {name, session_id, cwd, agent_id}, stage,
           state: todo|running|waiting|blocked|done|cancelled, cancelled?,
           stages: { <stage id>: {state, at, note, evidence: ["<path>[:a-b]"], media: [mid]} } }

question { id: "Q<n>", item|null, stage|null, kind: choice|open|gate|explore|acceptance,
           title, context_md, evidence: [{label, path, lines}], media: [mid],
           options: [{key, text, pros, cons, recommended, media: [mid]}],     // choice
           explore?: {min, max, cards: [{key, title, text, media: [mid], link}]},   // explore
           acceptance?: {items: [{key, title, expect, actual_md, verdict: PASS|FAIL, media: [mid]}]},
           review?: {repo, base, head, base_sha, head_sha, remote, branch},   // deliver gate
           status: open|followup|answered|consumed|cancelled, cancel_reason,
           decision: {choice, text, picks?, notes?, items?, at},
           thread: [{who: me|agent, at, text, choice, media: [mid], picks?, notes?, items?, comments?}] }

media    { id: "M<sha 前 12 碼>", sha256, name, mime, kind: image|gif|video|audio|sprite|model|other,
           size, width?, height?, duration?, derived: {thumb?, preview?, poster?, audio?},
           sprite?: {sheet: mid, frames_json: mid, anims_json?: mid},
           status: ready|processing|failed, error?, uploaded_by: me|agent, at }
```

- `explore` 的回答：`picks`（選中的 key，數量在 `min..max`）、`notes`（`{key: 一句意見}`）、`text`。
- `acceptance` 的回答：`items: {<key>: {ok: true|false, text}}`，每一項都要表態；任一項 `ok: false` 時 `choice = reject`，否則 `choice = approve`。退回項目沒有 `text` 回 400。
- gate 核准看 `decision.choice == 'approve'`；explore gate 有合法 `picks` 即視為核准。
- 狀態轉換（所有 kind 相同）：`open` → 擁有者決定 → `answered` → `cmdr wait` 讀走 → `consumed`。只打字不決定（追問）→ `followup` → `cmdr reply` → 回到 `open`（或 `consumed`）。

## 4. 流程與 gate 強制

類型模板 `types/<type>.json`：`{"stages": [{"id", "name", "phase", "gate"}]}`。`phase` ∈ `qa|explore|plan|build|accept|deliver`；骨架順序固定為 `qa* → explore*（可省略）→ plan* → build…（≥1）→ accept…（≥1）→ deliver*`，server 啟動時驗證，模板壞了就不啟動。

內建類型：

| type | stages |
|---|---|
| `art` | qa*（問答）、explore*（探索）、plan*（計畫）、build（實作）、accept（驗收）、deliver*（交付） |
| `dev` | qa*、plan*、build、accept、deliver* |

強制規則：

1. `cmdr stage` 把 item 推到某 stage 時，該 stage 之前（含）的每個 gate 都必須有核准的 gate 問題，否則回 409，訊息附上要送的指令。
2. explore gate 用 `cmdr explore` 送（kind `explore`）；qa／plan 用 `cmdr gate`；deliver 用 `cmdr gate ... deliver --review`。
3. deliver gate 送出前，該 item 最新一題 `acceptance` 必須所有項目 `ok: true`，否則 409。
4. 驗收 agent 與實作 agent 必須是不同 agent；驗收 agent 只拿驗收計畫與成果位置。
5. 回答只寫「TEST」或語意不明不算定案，session 要再問（session 規則，server 不判斷）。

## 5. API（JSON，全部要 token）

| 方法 路徑 | 用途 |
|---|---|
| `GET /api/tasks`、`GET /api/tasks/<t>` | 摘要列表／單一 task |
| `POST /api/tasks`、`POST /api/tasks/<t>/status` | 開 task／done・cancel・reopen |
| `POST /api/tasks/<t>/items`、`POST /api/tasks/<t>/items/<i>/status`、`.../agent` | 加 item／cancel・reopen／登記 agent |
| `POST /api/tasks/<t>/items/<i>/stage` | 更新 stage（gate 強制） |
| `POST /api/tasks/<t>/log` | 活動紀錄 |
| `POST /api/tasks/<t>/questions` | ask／gate／explore／acceptance（依 `kind`） |
| `POST /api/tasks/<t>/questions/<q>/answer` | 擁有者決定或追問（網頁用） |
| `POST /api/tasks/<t>/questions/<q>/poll` | `cmdr wait` 輪詢 + heartbeat |
| `POST /api/tasks/<t>/questions/<q>/reply` | agent 回覆追問 |
| `POST /api/watch` | `cmdr watch`：回 `{recalls, chats}` |
| `POST /api/tasks/<t>/chat`、`.../chat/say`、`.../chat/done` | task 對話框 |
| `POST /api/tasks/<t>/recalls`、`.../recalls/<r>/done` | 叫回 agent |
| `POST /api/media`（原始 body，標頭 `X-Filename`、`Content-Type`） | 上傳，回 media 物件；同 sha 直接回既有物件 |
| `POST /api/media/sprite` | `{sheet, frames_json, anims_json?}` 組成 sprite media |
| `GET /media/<mid>/<variant>` | `orig|thumb|preview|poster|audio`；支援 `Range`、`ETag`、`Cache-Control: private, max-age=31536000, immutable` |
| `GET /api/file?path=&lines=` | 白名單內的證據片段（≤1 MB） |
| `GET /api/review/<t>/<q>` | deliver 審查的 commit 列表與 diff |
| `GET /api/push/key`、`POST /api/push/subscribe`、`POST /api/push/test` | Web Push |
| `GET /api/projects` | 專案設定（網頁顯示比例尺用） |

錯誤：4xx/5xx 一律 `{"error": "<原因>"}`；cmdr 原樣印出。

## 6. cmdr 指令

與 ai-commander 用法相同的部分（語意見 §3、§4）：`task new|done|cancel|reopen`、`item add|cancel|reopen|agent`、`stage`、`log`、`ask`、`gate`、`wait`、`reply`、`watch`、`say`、`chat`、`chat done`、`recall`、`recall done`、`push`、`show`、`brief`、`resume`、`sessions`。

新增或改變的：

```sh
cmdr task new <task> --type art|dev --project <project> --title "<t>" [--link <url>]
cmdr media add <path> [--name <n>]                        # 印 M<id>；之後指令可用 M<id> 或路徑
cmdr media sprite <sheet.png> <frames.json> [<anims.json>] # 印 M<id>（kind sprite）
cmdr ask ... --media <M|path> ... --option "A|text|pros|cons" --option-media A=<M|path> ...
cmdr explore <task>/<item> --title "<t>" --context-file c.md --min 2 --max 3 \
     --card "k1|標題|說明" --card-media k1=<M|path> [--card-link k1=<url>] ...   # 6–8 張，印 Q<n>
cmdr accept-report <task>/<item> --file acceptance.json                         # 印 Q<n>
     # acceptance.json: {"title", "context_md", "items": [{"key","title","expect","actual_md","verdict","media":[path|M]}]}
cmdr wait <task>/Q<n> [--max 8h]                          # JSON 多 picks / notes / items / media
cmdr fetch <M>  [--out <dir>]                             # 下載擁有者上傳的媒體給 agent 看，印本機路徑
```

`cmdr wait` 結束碼：0 已決定、2 追問、3 逾時（題目不變）、4 題目被取消、1 連不到 server 或參數錯。

`cmdr` 對路徑參數先自動上傳（等同 `cmdr media add`），所以 agent 可以直接給本機檔案路徑。

## 7. 專案設定 `~/.art-cmdr/projects/<project>.json`

```json
{
  "id": "snowbell",
  "title": "雪鈴鎮",
  "repo": "D:/snowbell-town/snowbell-town",
  "file_roots": ["D:/snowbell-town/snowbell-town"],
  "ledger": "ledger/owner-decisions.md",
  "scale_ref": {"label": "狗本體", "px": 76, "media": "public/sprites/gold/gold.png", "frame": "walk_down_00"},
  "viewport": {"w": 1920, "h": 1080},
  "playtest": "npm run playtest"
}
```

- `file_roots`：證據與 `cmdr media add` 可讀的根目錄（再加上 `~/.art-cmdr/media`）；`.` 開頭的目錄與 `~/.art-cmdr` 其餘部分不開放。
- `scale_ref`：尺度預覽時，在素材旁畫出同像素比例的比例尺；`media` 指定時用該圖（sprite 取 `frame`）當比例尺。
- `ledger`：session 把定案寫入的帳本路徑（server 不寫）。

## 8. 媒體管線

| 輸入 | 偵測 | 衍生檔 | 手機顯示 |
|---|---|---|---|
| PNG/JPG/WebP | Pillow 不需要；`ffprobe` 取尺寸 | `thumb.webp`（長邊 480） | 縮圖 → 點開原檔，透明處棋盤格，雙指放大 |
| GIF | ffprobe（多影格） | `preview.mp4`（H.264，`-movflags +faststart`）、`poster.webp` | `<video loop muted playsinline>` |
| MP4/WebM/MOV | ffprobe | 長邊 > 1280 或非 H.264/VP9 → `preview.mp4` 720p；`poster.webp` | `<video>` + Range 串流 |
| OGG/Opus/MP3/M4A | ffprobe | 無 | `<audio>`；同一題的音效可 A/B 切換（同步播放位置） |
| WAV/FLAC | ffprobe | `audio.opus`（96 kbps） | 同上 |
| GLB/glTF（單檔 .glb；.gltf 需內嵌資源） | 副檔名 + magic | 無 | `<model-viewer camera-controls>` |
| sprite（sheet + Phaser JSON array/hash + 選用 anims JSON） | `cmdr media sprite` | `thumb.webp`（第一幀） | canvas 播放，選 anim、調速度（0.25×–2×）、逐幀 |

- 轉檔：背景佇列（同時 1 個 ffmpeg），media `status: processing` 期間網頁顯示原檔縮圖或佔位；失敗記 `error`，原檔仍可下載。
- 上限：單檔 500 MB（可設定）；內容定址，重複上傳不佔空間。
- 手機上傳：檔案選擇（相簿／相機）、錄音（`MediaRecorder`，WebM/Opus）；上傳時顯示進度，失敗可重試。

## 9. 網頁（PWA，手機優先）

| 頁 | 內容 |
|---|---|
| 收件匣（首頁） | 「需要你」：待答問題、gate、explore、驗收；「進行中」：各 task 目前 stage 與最後一則 log；「agent 等待中」 |
| task 頁 | item × stage 進度（手機用直向清單，不用寬矩陣）、問題列表、活動紀錄、與主 session 對話 |
| 決策頁（choice/open/gate） | 上：標題、上下文（markdown）、證據分頁、媒體；下方固定：選項卡（每張可展開媒體）、回覆框（文字＋附件＋錄音）、送出 |
| 探索頁（explore） | 候選牆：卡片格狀（手機 2 欄），每張圖／影片／音效預覽、標題、說明；點選加入（顯示「已選 n / max」）、每張「一句意見」；全螢幕檢視可左右滑動切換卡片 |
| 驗收頁（acceptance） | 逐項卡片：預期、實際（文字＋媒體）、agent 判定；每項「同意」「退回」+ 意見；全部表態後才能送出 |
| 審查頁（deliver review） | commit 列表、逐檔 diff（手機可橫向捲動）、逐行意見、Push／退回 |
| 媒體檢視器（全螢幕） | 依 kind 切換：圖片（棋盤格、放大）、影片、音效（A/B）、sprite、3D、尺度預覽開關 |
| 設定 | 開啟通知（訂閱 Web Push）、測試通知、token 登出 |

- PWA：`manifest.webmanifest`、圖示、`display: standalone`；service worker：app shell（`index.html` 與帶 hash 的 JS/CSS）precache、`/media/*` cache-first（immutable）、`GET /api/tasks` 與 `GET /api/tasks/<t>` stale-while-revalidate（先畫上次的資料並標示「更新中」，拿到新資料再換掉）、其餘 `/api/*` network-only；離線時顯示「筆電離線」與上次的資料（唯讀，不能送出）。
- 載入速度（2026-11-12 加入，見 `plans/art-cmdr-P1-perf-impl.md`）：已安裝的 PWA 再次開啟，1 秒內畫出收件匣（可為快取資料），2 秒內換成最新資料；首次開啟（無快取、4G）3 秒內可操作。首屏 JS（gzip 後）≤ 150 KB，`<model-viewer>`、diff、錄音等非首屏功能延遲載入。
- 通知內容：`<task> · <題目>`（不放媒體），點擊開到該題。主 session 回覆對話也通知。
- 輪詢：前景每 5 秒刷新目前頁（輸入中不打斷、草稿保留）。

## 10. 安全

- server 只聽 `127.0.0.1`；外網只經 `tailscale serve`（HTTPS，僅 tailnet 內可見）。不可用 `tailscale funnel`。
- token：32 bytes 隨機；網頁首次輸入後存 `HttpOnly; Secure; SameSite=Strict` cookie；cmdr 讀 `~/.art-cmdr/token`。
- 檔案讀取只在專案 `file_roots` 與媒體庫內；路徑先 resolve（含 symlink、Windows 磁碟代號大小寫）再比對。
- 上傳檔不執行、不解壓；回應帶 `X-Content-Type-Options: nosniff`；SVG 以 `Content-Disposition: attachment` 或 sandbox iframe 顯示，不 inline。
- deliver review 只跑唯讀 git（`log`、`diff`、`cat-file`），清掉 `GIT_*` 環境變數，加 `--no-ext-diff --no-textconv`、`core.hooksPath` 指向空目錄、`core.fsmonitor=false`。

## 11. Windows 部署

- 需求：Python 3.11+、Node 22（只建置網頁時需要）、ffmpeg/ffprobe（PATH 或 `config.json`）、Tailscale（啟用 MagicDNS 與 HTTPS certificates）、Git for Windows。
- `ops/install.ps1`：建立 `~/.art-cmdr`、產生 token 與 VAPID、`pip install -r requirements.txt`、`npm ci && npm run build`、把 `cmdr.cmd` 所在目錄加入使用者 PATH、建立 Task Scheduler 工作（登入時啟動 server，失敗每 5 分鐘重試）。
- `tailscale serve --bg 8765`（一次性設定，重開機保留）。
- 主控台編碼：所有 I/O 用 UTF-8（`PYTHONUTF8=1`），不依賴 cp950。
- 筆電睡眠時手機會顯示「筆電離線」；這期間 agent 也不在跑，不需要排隊機制。

## 12. 不在 v1

- 圖上／影格標註、並排比較、多輪淘汰、混搭（D4、D9）
- 多使用者與權限分級
- 停止或重跑 agent
