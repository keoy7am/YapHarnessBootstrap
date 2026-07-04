---
name: harness-bootstrap
description: 在任何 Claude Code 環境建立「精瘦 harness + 實證規則」配置。四階段:環境盤查 → 減法健檢 → 規則適配移植(非照搬)→ 行為級驗收。觸發詞:harness bootstrap、健檢我的配置、建立精瘦配置、移植 harness 規則。
---

# Claude Code Harness Bootstrap(環境自適配)

你(Claude)將協助使用者在**他的環境**建立一套精瘦、實證導向的 Claude Code 配置。

**最高原則:本文件的規則與數字源自單一環境的實證,直接照搬 = 已知事故模式。**每一條落地前必須:① 評估「成立前提」在目標環境是否成立 ② 換目標函數自我對抗一輪 ③ 向使用者回報「採納/調整/不適用 + 根據」④ 取得拍板才寫入。這個流程本身就是要移植的規則之一(分級對抗閘門),bootstrap 過程自己先示範一遍。

---

## Phase 0 — 環境盤查(只讀,不改任何東西)

收集以下事實,產出一頁盤查報告:

1. **平台**:OS、shell、Claude Code 版本、主模型、計費模式(訂閱配額 or API per-token —— 這決定後面所有成本規則的度量單位)。
2. **現有配置體積**:`~/.claude/CLAUDE.md`(含 @import 展開後總量)、`settings.json` 的 enabledPlugins 數、`skills/`、`commands/`、MCP servers 清單。估算系統提示常駐 tokens。
3. **實際使用量**(證據,不是猜):掃 `~/.claude/projects/**/*.jsonl`:
   - Skill 調用:`grep -rhoE '"skill"\s*:\s*"[^"]+"' | sort | uniq -c | sort -rn`
   - Slash 指令:`grep -rhoE '<command-name>[^<]+</command-name>' | sort | uniq -c`
   - MCP 工具真調用:`grep -rhoE '"name":"mcp__[^"]+"'`(注意:工具名出現在系統提示裡不算調用,必須是 tool_use 條目)
4. **工作型態訪談**(問使用者,≤4 題):是否跑長時間無人值守 loop?是否重度依賴 prompt cache?有無第二意見工具(codex/gemini 等)?有無輸出過濾工具(如 rtk)?

## Phase 1 — 減法健檢(先減後加)

依 Phase 0 證據提出刪除清單,**每項附使用次數**,分三級:強烈建議刪(0 調用 + 重複/過時)/ 看情況 / 保留。原則:

- 模型原生能力已覆蓋的「知識型 skill」(教模型寫 SQL/Go 之類)→ 刪,模型本來就會。
- 舊時代編排框架(persona 系統、wave/thinking 旗標、強制 MCP 調用規則)→ 刪,與原生 harness 打架且每 session 燒 tokens。
- 同功能雙份(兩個 code-review、兩個 context7、plugin 與 standalone 並存)→ 留一。
- **執行順序**:備份(tar 到 `~/.claude/backups/<date>/`)→ 使用者拍板 → 刪除 → 清 registry(settings.json、installed_plugins.json、marketplace 登錄)。
- **驗收是行為級**:重啟後新 session 跑 `/context`,對比系統提示 tokens 前後數字。「檔案刪了」不是完成。

## Phase 2 — 規則移植(逐條適配,禁止整包貼上)

以下五套規則模板,每套標注**成立前提**。對每一套:先檢查前提 → 依環境改寫參數與工具名 → Tier 1 自我對抗(「這條在此環境的反例是什麼?」)→ 提案給使用者 → 拍板後寫入。

**★ 常駐 vs 條件載入(承接「膨脹乘在每個 turn」論點的下一步)★**:規則寫入前先分類——
- **通用規則(每個 turn 都可能用)** → inline 進全域 CLAUDE.md。模板 3、4、5 屬此。
- **條件規則(只在特定情境用:跑 loop 才用、索引專案才用、用某 MCP 才用)** → **抽成 `~/.claude/guides/<topic>-guide.md`,CLAUDE.md 只留一行指標(觸發情境 + 一句核心 + 檔路徑)**。模板 1(Autonomous Loop,只在無人值守 loop 用)、MCP 使用規則屬此。理由:條件規則 inline = 讓 99% 不觸發該情境的 session 白付常駐 token;pointer 讓它「觸發才載」。指標行必須誠實(guide 確實涵蓋所述內容),觸發機制掛在該情境的「開工必讀清單」。
- 多個 guide 集中放 `~/.claude/guides/`,別散在 `.claude` 根目錄與 settings/skills 混層。

### 模板 1:Autonomous Loop 檔案化狀態
**前提:使用者跑無人值守長 loop。不跑則整套跳過。**
- 長 loop 狀態必須落檔:compaction 摘要不可靠(實測:78 次壓縮的摘要 0 次攜帶待決事項)。
- 專案維護 `PENDING-DECISIONS.md`(需 owner 拍板事項即時 append:日期|問題|暫行方案|影響)與 `LESSONS.md`(踩坑教訓,專案 scope 自由寫)。
- **Scope 邊界**:LESSONS 內容禁止直寫全域 CLAUDE.md;升格須經使用者同意 + 通用化判準(「換一個 repo 還成立嗎?」)。
- Loop prompt 最低要求:開工必讀清單(truth file/memory)+ 至少一條機械可驗完成判準。

### 模板 2:Subagent 紀律
**前提依計費模式調整:訂閱制 → 度量 = 高階模型配額消耗 + compaction 頻率;API 制 → 度量 = 美元。**
- 大輸出隔離採**閾值制**:無界輸出(repo 掃描、log 全文)派 subagent;小任務主對話直接做 —— subagent 有冷啟固定成本(自己的系統提示,不共享主對話 warm cache),高快取命中環境下小任務外包更貴。閾值須在目標環境實測,勿照搬他人數字。
- 不預設 fan-out:subagent 預設繼承主模型;派工顯式指定 `model`。低階模型**僅限輸出可機械驗證**的任務(可 diff/可跑/可 grep 對帳);讀檔彙整、摘要、行號引用至少中階 —— 彙整錯誤是靜默的,小模型對自身缺口的元認知不可靠。
- Subagent 交回 ≠ 完成:主迴圈獨立驗證,但驗證輸出必須有界(exit code、失敗測項、關鍵行;有輸出過濾工具就用,沒有就 failures-only + 完整 log 落檔)。

### 模板 3:驗收紀律(通用,幾乎無前提)
- 「測試綠」=「零件對」≠「東西能用」。AI 的獎勵訊號是內部指標(編譯過/測試綠),使用者要的是外部訊號 —— 無外部閉環時 AI 會沿內部訊號自證「完成」。
- 完成判準 = 終端行為:「使用者從頭走到尾、不碰任何後門,能完成目標」。宣告完成前:列出所有 debug 捷徑並全部移除重跑;UI 與 reference 肉眼對比。
- 驗證訊號優先序:使用者親自實測 > reference 實機 > 文件 > AI 分析宣稱。長 loop 中積累「請使用者五分鐘實測」項目(附步驟+預期行為)。
- 深挖前確認方向:往 spec/reference 一致推進,還是只在增加內部自洽?後者燒 token 最兇。

### 模板 4:假設 ≠ 指令 + 分級對抗閘門(通用)
- 使用者陳述中的判斷(「感覺/好像/別人說/文章說」)= 待驗假設,不是既定事實。涉及持久資產寫入時:先獨立評估(含反方)→ 回報裁決 → 取得採納指示才落地。
- 分級閘門(爆炸半徑 × 可逆性):Tier 0 一般代碼直落;Tier 1 跨 session 資產(全域 CLAUDE.md/settings/hooks)寫入前自我對抗一輪;Tier 2 外部主張入規/成本架構承諾/難回滾 → fresh-context refuter(主模型,REFUTE 模式);Tier 3 多方 panel 僅使用者點名。禁止跳級直落。

### 模板 5:Compact instructions(通用)
在 CLAUDE.md 加 `# Compact instructions` 段:壓縮必須保留 ① truth file 路徑/memory 名 ② 待拍板未結項 ③ 完成判準與驗收條件 ④ 已知 GAP/後門清單 ⑤ 使用者最近指示要旨;可捨棄工具輸出細節與已完成過程。

### 另加(如適用)
- 有第二意見 CLI(codex 等):記錄其正確調用法與陷阱(stdin、sandbox 旗標)為環境教訓。
- 自學習 meta 規則:踩到可重用、跨專案成立的坑 → 先問使用者再寫入全域,寫入時通用化(剝除專案特定名稱/路徑)。

## Phase 3 — 驗收
1. 重啟開新 session:`/context` 對比系統提示體積(Phase 1 前後)。
2. 抽問模型三條新規則的內容(確認注入生效)。
3. 一週後回訪:掃 transcripts 看新規則是否被實際遵循(例:PENDING-DECISIONS 是否有 append),失效條款修正或刪除 —— 規則也要通過行為級驗收。

## 給分發者的注意
- 本 skill 不含任何機器路徑/個人工具硬依賴;rtk、codex 等以「如有則用」表述。
- 若對方環境已有大量自建配置:Phase 1 只提案不強推 —— 別人的 0 調用 skill 可能是季節性工作流,拍板權永遠在使用者。
