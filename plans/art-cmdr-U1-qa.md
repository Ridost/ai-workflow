# art-cmdr U1 QA 開題手冊（給筆電上的主 session）

擁有者要求（2026-11-12）：在 art cmd 開一個任務，跟擁有者 QA 討論「看不出有多少任務在執行、每個任務進度不清楚」的介面與任務管理改版。

- 提案：`plans/art-cmdr-U1-status-proposal.md`（先讀完再開題）
- mockup：`art-cmdr/docs/mockups/u1-status.html`、`u1-inbox.png`、`u1-task.png`、`u1-agents.png`
- 本手冊在雲端 session 寫成（連不到筆電上的 art cmd），所以由**筆電上的主 session** 照下面步驟開題。
- 下面的 `cmdr ask` 旗標照 DESIGN §6；若實際 `cmdr ask --help` 的旗標名不同（例如 context 用 `--context` 而不是 `--context-file`），照實際的改（`--kind open`、`--recommend` 也一樣：推薦選項要對應到 option 的 `recommended`），題目內容不要改。

## 0. 前置

1. `git pull`，確認本 repo 有上面的檔案。
2. 若 `~/.art-cmdr/projects/ai-workflow.json` 不存在，建立它（`file_roots` 讓 mockup 圖可以附上）：

   ```json
   {"id": "ai-workflow", "title": "AI 工作流程", "repo": "<本 repo 絕對路徑>", "file_roots": ["<本 repo 絕對路徑>"]}
   ```

   建好後重啟 art-cmdr server（或照實作說明重新載入專案設定）。

## 1. 開 task 與 item

```sh
cd <本 repo 絕對路徑>
M=art-cmdr/docs/mockups
P=plans/art-cmdr-U1-status-proposal.md

cmdr task new cmdr-u1 --type dev --project ai-workflow \
  --title "art cmd 介面改版：任務數量與進度一目了然" \
  --link "$P"
cmdr item add cmdr-u1/ui --title "介面與任務管理改版"
cmdr stage cmdr-u1/ui qa running --note "開 QA：Q1–Q10，依提案 plans/art-cmdr-U1-status-proposal.md"
```

## 2. 題目

題目一次全部送出（擁有者在手機上可以挑順序回答），然後每題各開一個背景 `cmdr wait`。每題回答後照 DESIGN §4 規則 5 判斷是否定案；不明確就追問（`cmdr reply`）。

### Q-a 先看你現在卡在哪（open，附截圖請求）

```sh
cmdr ask cmdr-u1/ui --kind open \
  --title "請截一張你現在最看不懂的 art cmd 畫面，並說一句『我想知道但看不出來的是 ___』" \
  --context "我想先確認我推測的原因對不對（提案 §1 P1–P6）。可以從手機直接附截圖或錄一段語音。"
```

### Q-b 「有多少在執行」數的是哪一層？（choice）

```sh
cmdr ask cmdr-u1/ui --title "首頁的「執行中」要數哪一層？" \
  --context "資料模型有三層：任務（task）→ 工作項（item，一條 stage 線）→ agent（正在做工作項的 session）。現在畫面沒有講清楚數的是哪一層。" \
  --media $M/u1-inbox.png \
  --option "A|任務 + agent 兩個都顯示|一眼知道幾件事在動、幾個 agent 在燒|數字比較多" \
  --option "B|只數任務|最簡單|看不出某任務是不是開了很多 agent" \
  --option "C|只數 agent|直接對應資源用量|看不出是幾件事" \
  --recommend A
```

### Q-c 頂端計數列要放哪些？（choice）

```sh
cmdr ask cmdr-u1/ui --title "頂端計數列放哪幾個數字？（每個都可點，點了過濾清單）" \
  --media $M/u1-inbox.png \
  --option "A|待你／執行中／停住／今日完成|涵蓋「輪到誰」與「有沒有東西壞掉」|4 格在手機上略擠" \
  --option "B|只放 待你／執行中|最簡潔|停住的任務要點進去才知道" \
  --option "C|依 stage 分：QA n、實作 n、驗收 n、交付 n|看得到流程堆在哪一段|看不出卡在你還是卡在 agent" \
  --recommend A
```

### Q-d 進度怎麼表示？（choice）

```sh
cmdr ask cmdr-u1/ui --title "每個任務／工作項的進度要怎麼表示？" \
  --context "提案 §2.3：stage 步進條（qa·探索·計畫·實作·驗收·交付）＋實作階段由 agent 回報 milestone 子進度（例：M4 網頁 PWA 3/7）＋任務百分比。百分比只是粗估。" \
  --media $M/u1-task.png \
  --option "A|步進條＋子進度＋百分比|最完整，一條進度條最直覺|百分比是公式算的，可能讓人誤以為很準" \
  --option "B|步進條＋子進度，不要百分比|不會有假精確|任務層級沒有一條總進度條" \
  --option "C|只要一句文字「現在在做什麼」|最省版面|看不出還剩多少" \
  --recommend A
```

### Q-e 多久沒回報算「停住」？（choice）

```sh
cmdr ask cmdr-u1/ui --title "agent 多久沒有任何 cmdr 回報，就標成 🟠 停住？" \
  --context "現在 running 是 agent 自己報的，agent 當掉、session 關掉、筆電睡眠後仍會顯示 running。提案改成由 server 看 heartbeat（每次 cmdr 呼叫都算）。跑很久的測試可用 cmdr beat 續命。" \
  --option "A|15 分鐘|很快發現問題|跑長測試的 agent 要記得 beat，否則誤報" \
  --option "B|30 分鐘|誤報少|最多晚半小時才知道" \
  --option "C|60 分鐘|幾乎不誤報|發現太慢" \
  --option "D|每個專案自己設（預設 15）|snowbell playtest 很長可以設長一點|多一個設定" \
  --recommend D
```

### Q-f 首頁版面（choice）

```sh
cmdr ask cmdr-u1/ui --title "首頁要長什麼樣子？" \
  --media $M/u1-inbox.png --media $M/u1-task.png \
  --option "A|收件匣＝計數列＋待你＋任務總覽卡（如 mockup ①）|打開就看到全部|待你很多時任務卡被擠到下面" \
  --option "B|收件匣只放待你；任務總覽在「任務」頁籤|收件匣最乾淨|要切頁才看得到進度" \
  --option "C|看板：欄＝stage，卡＝工作項，橫向捲動|一眼看出東西堆在哪一段|手機上要橫向捲，與 DESIGN 手機原則衝突" \
  --recommend A
```

### Q-g 要不要獨立的 agent 頁籤？（choice）

```sh
cmdr ask cmdr-u1/ui --title "要不要新增「agent」頁籤（誰在跑、誰停住、誰在等你）？" \
  --media $M/u1-agents.png \
  --option "A|要（如 mockup ③）|停住的 agent 一目了然，可直接叫回|多一個頁籤" \
  --option "B|不要，agent 資訊放在任務頁每個工作項旁邊|頁籤少|跨任務看不到全部 agent" \
  --recommend A
```

### Q-h 同時進行的任務要不要設上限？（choice）

```sh
cmdr ask cmdr-u1/ui --title "同時「執行中」的任務數要不要設上限（WIP）？" \
  --context "目的：避免同時開太多條線，結果每條都半完成、你每天要回一堆題。另外提案新增「暫停」狀態，暫停的不算、不推播。" \
  --option "A|不設限，只顯示數字|最自由|容易越開越多" \
  --option "B|軟上限（例 3）：超過時首頁顯示提醒|有提醒但不擋|可能被忽略" \
  --option "C|硬上限：超過時 cmdr 拒絕把新任務推進實作|強制收斂|急件要先暫停別的" \
  --recommend B
```

### Q-i 完成的任務與通知（choice）

```sh
cmdr ask cmdr-u1/ui --title "完成的任務放哪？通知要不要加？" \
  --option "A|完成的折疊在底部；推播加「agent 停住」|清單乾淨，壞掉會被通知|多一種推播" \
  --option "B|A＋每天早上一則摘要（昨日完成 n、待你 n、停住 n）|每天有全貌|多一則固定推播" \
  --option "C|完成的折疊在底部；推播維持現狀（只有新題目）|推播最少|停住要自己打開才知道" \
  --recommend A
```

### Q-j 還有沒說到的？（open）

```sh
cmdr ask cmdr-u1/ui --kind open \
  --title "還有什麼「想知道但看不出來」或任務管理上的困擾，提案沒有提到？" \
  --context "例如：想看每個任務花了多久、想手動排優先順序、想在電腦上用寬版畫面……沒有的話回「沒有」。"
```

## 3. 收尾

1. 全部定案後，把每題的決定整理成表，用 `cmdr gate cmdr-u1/ui qa` 送 qa gate（標題：「U1 QA 定案，核准後改 DESIGN 並寫實作計畫」），context 列出：每題的決定、對提案的修改。
2. gate 核准後：
   - 改 `art-cmdr/DESIGN.md`（§3 資料模型、§5 API、§6 cmdr、§9 網頁），決策紀錄加 D16 起的新列，註明 `via art-cmdr cmdr-u1/Q<n>`。
   - 寫 `plans/art-cmdr-U1-status-impl.md` 與 `plans/art-cmdr-U1-status-accept.md`（格式照 P1）。
   - `cmdr stage cmdr-u1/ui plan running`，送 plan gate。
3. 與 P1 速度改版的關係：U1 的計數列、燈號、進度都放在 `GET /api/tasks` 一次回傳，不能讓收件匣多發請求（P1 驗收 B5）；兩者都改到收件匣時，先合 P1 再做 U1。
