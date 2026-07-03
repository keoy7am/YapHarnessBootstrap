# YapHarnessBootstrap

> 在任何 Claude Code 環境建立「精瘦 harness ＋ 實證規則」配置的自適配技能。
> A self-adapting skill that audits and rebuilds your Claude Code configuration — lean, evidence-based, and tailored to *your* environment (not copy-pasted from someone else's).

## 為什麼需要它

一次真實健檢的前後對比:

| | 健檢前 | 健檢後 |
|---|---|---|
| 系統提示常駐成本 | ~30k tokens/session(SuperClaude 框架 + 百餘個 skill) | ~5k tokens/session |
| Skill 實際調用率 | 大量 skill 幾週內 **0 調用** | 只留有使用證據的 |
| 規則品質 | 過時編排框架與原生功能打架 | 五套實證規則(每條附成立前提) |

核心發現:**配置膨脹的每一分都乘在每個 session 的每一個 turn 上**。刪掉沒人用的東西,比加任何新東西都更有效。

## 這不是配置檔,是流程

直接複製別人的配置是已知事故模式(本專案規則中稱為「45% 事故」:把他人環境的實測數字當通則搬運,被三方對抗審查否證)。所以這個 skill 不給你配置檔,而是讓你的 Claude 執行四階段流程:

1. **Phase 0 環境盤查** — 計費模式、現有配置體積、transcripts 用量證據、工作型態
2. **Phase 1 減法健檢** — usage-based 刪除提案(附使用次數)→ 備份 → 你拍板 → 刪除 → 行為級驗收
3. **Phase 2 規則移植** — 五套規則模板逐條適配:檢查成立前提 → 自我對抗 → 提案 → 你拍板才寫入
4. **Phase 3 驗收** — `/context` 前後對比 + 一週後回訪(規則沒被遵循就修或刪)

### 五套規則模板

| 模板 | 解決的痛點 | 成立前提 |
|---|---|---|
| Autonomous Loop 檔案化狀態 | 長 loop 的待辦決策被 compaction 靜默吞掉(實測 78 次壓縮摘要 0 次攜帶待決項) | 你跑無人值守 loop |
| Subagent 紀律 | 預設 fan-out 燒 N 倍旗艦配額;小任務外包反而更貴(快取經濟學) | 依計費模式調整度量 |
| 驗收紀律 | 「測試綠」被 AI 當成「完成」,零件全對但組起來是錯的東西 | 通用 |
| 假設≠指令 ＋ 分級對抗閘門 | AI 把你的「感覺」當事實直接執行;外部文章的主張未經驗證直接入規 | 通用 |
| Compact instructions | 壓縮時丟失驗收條件與 GAP 清單 | 通用 |

## 安裝與使用(三選一)

### 方式一:裝成 skill(推薦)

```bash
git clone https://github.com/keoy7am/YapHarnessBootstrap.git
# macOS / Linux / Git Bash:
cp -r YapHarnessBootstrap/skills/harness-bootstrap ~/.claude/skills/
```

然後在 Claude Code 裡說:

```
執行 harness bootstrap,為我健檢並重建配置
```

### 方式二:直接當 prompt

把 [`skills/harness-bootstrap/SKILL.md`](skills/harness-bootstrap/SKILL.md) 的內容整段貼進 Claude Code 對話。

### 方式三:只做健檢不動配置

```
依照 harness bootstrap 的 Phase 0 + Phase 1,只給我盤查報告與刪除建議,先不要動任何東西
```

## 重要警語

- **所有數字(閾值、百分比)源自單一環境實證,不可直接照搬** —— skill 內建的分級對抗閘門會強制 Claude 對每條規則評估你環境的前提,這是設計的一部分,別跳過。
- Phase 1 的刪除一律先備份;拍板權永遠在你。
- 個人工具(rtk、codex 等)在 skill 中皆為「如有則用」,無硬依賴。

## 貢獻

Agentic Coding 的痛點永遠比規則多。歡迎:

- **回報痛點 Issue**:你在長 loop / subagent 委派 / AI 假完成 / 配置膨脹上踩過的坑
- **提交 PR**:新規則模板、既有模板的失效案例、其他環境的實測數據

詳見 [CONTRIBUTING.md](CONTRIBUTING.md)。

## License

MIT — 見 [LICENSE](LICENSE)。
