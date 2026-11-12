# art-cmdr U1 任務狀態一目了然 — 改版提案（待 QA 定案）

- 起因（2026-11-12，擁有者）：介面看不出**現在到底有多少任務在執行**，也不清楚**每個任務目前進度如何**。
- 狀態：**提案**。依 DESIGN 開頭的規則，要改設計先經擁有者同意；本提案的每個「待定」都對應 `plans/art-cmdr-U1-qa.md` 的一題，在 art cmd task `cmdr-u1` 問答定案後，才改 `art-cmdr/DESIGN.md` 並寫實作／驗收計畫。
- mockup：`art-cmdr/docs/mockups/u1-status.html`（截圖 `u1-inbox.png`、`u1-task.png`、`u1-agents.png`）。

## 1. 問題拆解（從現行 DESIGN 推出來的原因）

| # | 現象 | DESIGN 裡的原因 |
|---|---|---|
| P1 | 不知道「有多少在跑」 | 收件匣（§9）只有三個清單區塊，沒有總數；「任務」在資料模型裡有三層（task → item → agent），畫面沒有說清楚數的是哪一層 |
| P2 | 「進行中」看不出進度 | 收件匣只顯示 task 的「目前 stage」，但一個 task 有多個 item、各在不同 stage，task 層級沒有單一 stage 可顯示 |
| P3 | build 階段是黑盒子 | stage 只有 `state`，沒有子進度；build 可能要跑好幾個 milestone（例：D1 有 M1–M7），中間只看得到「running」 |
| P4 | 分不出「在跑」還是「停住了」 | item `state: running` 是 agent 自己報的；agent 當掉、session 被關、筆電睡眠後仍顯示 running。`cmdr wait` 有 heartbeat，但沒在等的 agent 沒有 |
| P5 | 分不出「卡在我」還是「卡在 agent」 | task `status` 只有 `active|done|cancelled`，「需要你」是另一個區塊，同一個 task 會出現在兩處 |
| P6 | 完成的東西混在裡面 | 沒有封存觀念，task 一多清單就長 |

## 2. 提案

### 2.1 一個詞對一層（解 P1）

| 畫面用詞 | 對應 | 說明 |
|---|---|---|
| **任務** | task | 一件要交付的事（例：`art-cmdr`、`snowbell 冬季地圖`） |
| **工作項** | item | 任務裡的一條線（例：`server`、`web`），各自走 stage |
| **agent** | item.agent | 正在做某個工作項的 Claude session／subagent |

首頁頂端固定一條**計數列**（待定 Q2）：

```
 待你 3 │ 執行中 2 任務 · 4 agent │ 停住 1 │ 今日完成 2
```

每個數字可點，點了直接過濾下面的清單。

### 2.2 task 的「燈號」由 server 推導，不靠 agent 自報（解 P4、P5）

server 依下列優先順序算出每個 task 的單一燈號（`task.light`），收件匣與通知都用它：

| 燈號 | 條件（由上往下，先符合者為準） |
|---|---|
| 🔴 待你 | 有 `open` 的題目（ask／gate／explore／acceptance） |
| 🟠 停住 | 有 item `running`，但其 agent `last_seen` 超過門檻（待定 Q4，預設 15 分鐘） |
| 🟢 執行中 | 有 item `running` 且 agent 活著 |
| 🔵 等 agent 接手 | 你已回答但題目還沒被 `cmdr wait` 讀走（`answered`），或有未處理的 recall／對話留言——代表球在 agent 那邊但還沒人接 |
| ⚪ 閒置 | `active` 但上面都不符合 |
| ✓ 完成／取消 | `done`／`cancelled`（預設不顯示，見 2.6） |

**agent 存活（heartbeat）**：每個 `cmdr` 呼叫（`log`、`stage`、`progress`、`wait` 輪詢……）都帶 `--as <agent>` 或從環境變數 `CMDR_AGENT` 取得 agent 名稱，server 更新 `item.agent.last_seen`。另加 `cmdr beat <task>/<item>`，給長時間跑測試、沒有其他輸出的 agent 用。

### 2.3 進度 = stage 位置 ＋ build 子進度（解 P2、P3）

- **stage 步進條**：每個工作項一列，`qa · explore · plan · build · accept · deliver` 六格，已完成實心、目前那格閃動、gate 格有鎖頭圖示。
- **build 子進度**：新增 `cmdr progress <task>/<item> --done 3 --total 7 --step "M4 網頁 PWA"`，寫入 `stages.build.progress = {done, total, step, at}`。實作計畫裡的 milestone 就是 total；畫面顯示「M4 網頁 PWA · 3/7」。
- **task 進度百分比**（待定 Q3）：預設公式 = 各工作項（已完成 stage 數 + 目前 build 子進度比例）÷ stage 總數，取平均。只當粗略指標，旁邊一定同時寫「現在在做什麼」。
- **最後動態**：每列右側「3 分鐘前」，超過門檻變橘色（同 Q4）。

### 2.4 task 卡片的一句話「下一步」（解 P5）

task 卡片固定兩行文字，回答「現在在幹嘛」「接下來輪到誰」：

```
art-cmdr  ███████░░░ 68%                       🟢 執行中
  web：build · M4 網頁 PWA 3/7 · 2 分鐘前
  下一步：agent — 交 mockup 給你看（預計問 Q15）
```

`下一步` 由主 session 用 `cmdr next <task> "<一句話>"` 維護；沒寫時 server 用燈號自動產生（「等你回答 Q12」「等 web agent 回報」）。

### 2.5 新增「agent」頁籤（解 P1、P4）

底部導覽改成四個頁籤：**收件匣｜任務｜agent｜設定**。agent 頁列出所有登記過、未結束的 agent：名稱、所屬任務／工作項、目前 stage、`last_seen`、是否在 `cmdr wait`（在等哪一題）。停住的排最上面，並提供「叫回主 session」按鈕（沿用 recall，不新增停止／重跑，DESIGN §12 仍不做）。

### 2.6 任務管理方式（解 P6；細節待定 Q5–Q7）

- **排序**：收件匣只放 🔴 待你（依題目送出時間）；任務頁依燈號分組，同組內依優先度、再依最後動態。
- **優先度**：task 新增 `priority: P0|P1|P2`（預設 P1），`cmdr task new --priority`、網頁可長按改。
- **同時執行上限（WIP）**（待定 Q6）：可設定「同時 🟢 執行中的 task 上限」，超過時 `cmdr stage ... build running` 回 409，提示先完成或暫停一個。目的：避免開太多條線後全部半完成。
- **暫停**：task 新增 `status: paused`，暫停中不算 WIP、不推播（例：等素材、等別的 task）。
- **封存**：`done`／`cancelled` 預設不顯示；任務頁底部「已完成（n）」折疊（待定 Q7）。
- **每日摘要**（可選，待定 Q8）：每天固定時間推播一則「昨日完成 n、今日待你 n、停住 n」。

## 3. 資料模型與 API 變更（定案後寫進 DESIGN §3、§5、§6）

```
task  + priority: P0|P1|P2, status 加 paused, next?: {text, at},
        light（server 推導，不存檔）, progress（server 推導，0–1）
item.agent + last_seen, waiting: bool
item.stages.<id> + progress?: {done, total, step, at}
```

| 新增 | 用途 |
|---|---|
| `cmdr progress <task>/<item> --done n --total m --step "<t>"` | build 子進度 |
| `cmdr beat <task>/<item>` | 純 heartbeat |
| `cmdr next <task> "<一句話>"` | 下一步 |
| `cmdr task pause|resume <task>`、`--priority` | 任務管理 |
| `GET /api/tasks` 回應加 `counts: {mine, running_tasks, running_agents, stalled, done_today}` 與每個 task 的 `light, progress, next, items[].{stage, progress, last_seen}` | 首頁一次拿齊（符合 P1 perf 的「收件匣單一請求」） |
| `GET /api/agents` | agent 頁 |

## 4. 不改

- 流程骨架與 gate 強制（DESIGN D6、§4）不變。
- 不新增停止／重跑 agent（DESIGN §12）。
- P1 速度目標不變：計數列與燈號都在 `GET /api/tasks` 一次回傳，不多發請求。
