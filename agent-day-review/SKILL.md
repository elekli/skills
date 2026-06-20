---
name: agent-day-review
description: |
  跨 agent (Claude Code + Cursor) 增量整理今日 session，append 到當週 cycle log 對應日期下，
  作為晚間 30-40 分鐘 review 的事實底稿。內容是 what + pivot/insight，嚴禁杜撰。
  Triggers: "/agent-day-review", "總結今天", "日總結", "review 今天", "整理今天的 sessions"
allowed-tools: [Read, Bash, Edit, Write, Agent]
user-invocable: true
---

# Agent Day Review

每日跑一到多次（手動 + cron + 手機 dispatch），把當日跨 agent 的 user prompt 整理成
「事實底稿」append 進當週 cycle log。**不取代使用者自己的反思**——是給他基礎讓他加判斷。

---

## 核心原則

1. **增量**：每次只處理「上次跑完之後」的活動。state file 記 `last_processed_ts` per source。
2. **多次 trigger 並存**：18:30 手動、20:00 cron、22:30 手機 dispatch 各自跑各自的，**新建 callout，不 merge**。
3. **嚴禁杜撰**：subagent 只能寫 user prompt 字面直接支持的內容。寧可少不可亂。
4. **記錄 what + pivot/insight**：流水帳沒用，要抓判斷、轉折、blocker。
5. **2000 字硬上限**：跨 callout 內所有 task 加總。

---

## 流程

### Phase 0：解析 args

skill 接受的 slash arg：

| arg | 行為 |
|---|---|
| `/agent-day-review` | 正常跑：寫進 cycle log + 更新 state file |
| `/agent-day-review --dry-run` | 只列印「將要 append 的 callout 內容」+ 「state file 將更新成什麼」，**不寫任何檔案** |
| `/agent-day-review dry-run` | 同上（容錯，不要求 `--`） |
| `/agent-day-review --since=YYYY-MM-DDTHH:MM:SSZ` | 強制覆寫 state file 的 `last_processed_ts`，從指定時間點重新處理（debug / 補跑用） |
| `/agent-day-review --file=<path>` | 指定／重新指定 callout 要寫入的檔案；命中後記住於 state file（見 Phase F1） |

env var 仍支援（給 cron / launchd 使用）：
- `AGENT_REVIEW_DRY_RUN=1` 等同 `--dry-run`
- `AGENT_REVIEW_SINCE=...` 等同 `--since=...`
- `AGENT_REVIEW_FILE=<path>` 等同 `--file=...`

設一個 `DRY_RUN=0/1` flag，在 Phase F (寫檔) 和 Phase G (寫 state) 前 short-circuit。

### Phase A：讀 state file

```bash
STATE_FILE="$HOME/.claude/skill-state/agent-day-review/last_run.json"
mkdir -p "$(dirname "$STATE_FILE")"
```

Schema：
```json
{
  "last_run_iso": "2026-05-02T10:30:00Z",
  "claude_code_last_processed_ts": "2026-05-02T10:30:00Z",
  "cursor_last_processed_ts": "2026-05-02T10:30:00Z",
  "cycle_log_file": "/absolute/path/to/current-cycle-log.md"
}
```

`cycle_log_file`：上次解析定案的目標檔絕對路徑（Phase F1 順序 4 問到後寫入）。不存在代表還沒問過。

時間戳欄位不存在或缺：預設 **今天 00:00 Asia/Taipei** 對應的 UTC（避免第一次跑就回溯一週）。
用 `python3 -c "from datetime import datetime, timezone, timedelta; tz=timezone(timedelta(hours=8)); now=datetime.now(tz); midnight=now.replace(hour=0,minute=0,second=0,microsecond=0); print(midnight.astimezone(timezone.utc).isoformat())"` 取當日 0 點。

### Phase B：增量掃 sessions

兩個 source 平行處理：

#### B1. Claude Code

```python
import json, glob, os
from datetime import datetime, timezone

since = "<claude_code_last_processed_ts>"  # ISO UTC
files = glob.glob(os.path.expanduser("~/.claude/projects/*/*.jsonl"))
files += glob.glob(os.path.expanduser("~/.claude/projects/*/*/subagents/*.jsonl"))

tuples = []  # (timestamp_iso, working_dir, user_text)
for fp in files:
    # working_dir 從目錄名 -Users-<you>-xxx 推回 /Users/<you>/xxx
    proj_dir = os.path.basename(os.path.dirname(fp))
    if proj_dir == "subagents":
        proj_dir = os.path.basename(os.path.dirname(os.path.dirname(fp)))
    cwd = "/" + proj_dir.lstrip("-").replace("-", "/")
    
    with open(fp) as f:
        for line in f:
            try:
                o = json.loads(line)
            except:
                continue
            ts = o.get("timestamp", "")
            if not ts or ts <= since:
                continue
            if o.get("type") != "user":
                continue
            c = o.get("message", {}).get("content", "")
            text = ""
            if isinstance(c, list):
                for x in c:
                    if isinstance(x, dict) and x.get("type") == "text":
                        text = x.get("text", "")
                        break
            elif isinstance(c, str):
                text = c
            if not text or _is_noise(text):
                continue
            tuples.append((ts, cwd, text))
```

`_is_noise`：丟掉以下：
- `<local-command-caveat>` 開頭
- `<system-reminder>` 開頭
- 純 slash command 無實質內容（`/clear`, `/exit`, `/warmup`, `/handoff` 後沒接 prompt 的）
- tool_result 訊息

#### B2. Cursor-cli

```python
import re
files = glob.glob(os.path.expanduser("~/.cursor/projects/*/agent-transcripts/*/*.jsonl"))
TS_RE = re.compile(r"<timestamp>([^<]+)</timestamp>")
QUERY_RE = re.compile(r"<user_query>(.*?)</user_query>", re.DOTALL)

# Cursor timestamps are like "Tuesday, Apr 28, 2026, 11:09 AM (UTC+8)"
def parse_cursor_ts(s):
    s = s.strip()
    # 去掉 (UTC+N) 後面，然後 strptime
    m = re.match(r"(.+?) \(UTC([+-]\d+)\)", s)
    if not m: return None
    dt_part, tz_part = m.group(1), int(m.group(2))
    # "Tuesday, Apr 28, 2026, 11:09 AM"
    dt = datetime.strptime(dt_part, "%A, %b %d, %Y, %I:%M %p")
    return dt.replace(tzinfo=timezone(timedelta(hours=tz_part))).astimezone(timezone.utc)

since_dt = datetime.fromisoformat(cursor_since.replace("Z", "+00:00"))

for fp in files:
    proj_name = fp.split("/agent-transcripts/")[0].split("/")[-1]
    # working_dir：先掃同 session 的 assistant tool_use 抓 working_directory
    cwd_guess = None
    with open(fp) as f:
        lines = f.readlines()
    for line in lines:
        try:
            o = json.loads(line)
            if o.get("role") == "assistant":
                for x in o.get("message", {}).get("content", []):
                    if isinstance(x, dict) and x.get("type") == "tool_use":
                        wd = x.get("input", {}).get("working_directory")
                        if wd:
                            cwd_guess = wd; break
                if cwd_guess: break
        except: pass
    if not cwd_guess:
        cwd_guess = f"~/cursor/{proj_name}"  # fallback
    for line in lines:
        try:
            o = json.loads(line)
            if o.get("role") != "user": continue
            for x in o.get("message", {}).get("content", []):
                if not (isinstance(x, dict) and x.get("type") == "text"): continue
                text_blob = x.get("text", "")
                tm = TS_RE.search(text_blob)
                qm = QUERY_RE.search(text_blob)
                if not (tm and qm): continue
                dt = parse_cursor_ts(tm.group(1))
                if not dt or dt <= since_dt: continue
                user_text = qm.group(1).strip()
                if _is_noise(user_text): continue
                tuples.append((dt.isoformat(), cwd_guess, user_text))
        except: pass
```

### Phase C：分桶

按 `working_dir` 分桶。同 cwd 的 tuples 進一個 subagent。

如果一個 cwd 有 > 30 條 prompts，先在 parent 截斷成最近 30 條（subagent context 顧慮）。

### Phase D：Subagent fan-out（最多 3 並行）

每 batch 最多 3 個 cwd 平行處理。cwd 多就分批跑。

**Subagent prompt 模板**（嚴格不變）：

```
你會收到單一 working_dir 下的 user prompts 序列。

WORKING_DIR: {cwd}
PROMPTS:
{每行 "{taipei_time} | {text截到200字}"}

任務：

1. **聚類成 1-N 個 coherent task**。task 邊界看主題連續性 + 時間鄰近。
   不是 1 prompt = 1 task。同主題不同時段也要合在一起。
   
2. **每個 task 寫 1-3 句中文（80-150 字）**：
   - **what**: 在做什麼具體工作（檔案、feature、決策）
   - **pivot/insight**: 從 user 措辭可見的判斷、轉折、blocker
   
3. **嚴格禁止杜撰**：
   - 只寫 user prompt 字面直接支持的內容
   - 不確定就省略；不要從「常見模式」推測
   - 不要把 user 沒說的「為什麼」補進去
   - 寧可少不可亂
   
4. **輸出格式**：
   ```
   - **{cwd basename}** — {task 描述}
   - **{cwd basename}** — {task 描述}
   ```
   
5. **字數預算**：本 cwd 上限 {budget} 字。

6. 如果這個 cwd 全部都是雜訊（slash-only、空訊息），輸出單行：
   ```
   (本 cwd 無實質互動)
   ```
```

`{budget}` = `floor(2000 / cwd_count) + 30`（給有多 task 的 cwd 餘裕）。
`cwd basename` = `os.path.basename(cwd)` 或 `cwd.split("/")[-2:]` 拼接（取最後兩段更可讀，例如 `my-vault/0 Inbox`）。

### Phase E：合併 + 字數預算

把所有 subagent 回傳的 bullet 合併。如果加總 > 2000 字，按比例 trim 每個 cwd 的最末 task。**不在 parent 重寫內容**（保訊息保真）。

### Phase F：解析目標檔 + insert callout

#### F1. 解析要寫入的檔案

callout 要 append 到哪個檔，按以下順序解析（**第一個命中為準**）：

| 順序 | 來源 | 說明 |
|---|---|---|
| 1 | `--file=<path>` arg | 明確指定／重新指定。命中後寫回 state file |
| 2 | `AGENT_REVIEW_FILE` 環境變數 | 給 cron / launchd 用（等同「明確指定」） |
| 3 | state file 的 `cycle_log_file` | 上次記住的答案 |
| 4 | **互動詢問使用者** | 前三者皆無時，問使用者要寫去哪個檔。**得到答案後寫回 state file**，下次起走順序 3 |
| 5 | 非互動且前四者皆無 | **報錯退出，不靜默寫檔**（見「失敗模式」） |

```python
import os

target = None
if file_arg:                                    # 來自 --file=
    target = file_arg
elif os.environ.get("AGENT_REVIEW_FILE"):
    target = os.environ["AGENT_REVIEW_FILE"]
elif state.get("cycle_log_file"):
    target = state["cycle_log_file"]

if target:
    target = os.path.abspath(os.path.expanduser(target))
# else → 走 F1b
```

**F1b（`target` 仍為 None）**：

- **互動執行**：問使用者「這次的 review 要 append 到哪個檔案？給我絕對路徑」，取得後設給 `target`。
- **非互動執行**（cron / `--no-interactive` / 手機 dispatch 無人應答）：印出明確錯誤後**退出，不寫任何檔**：
  ```
  agent-day-review: 無 target 檔案。請用 --file=<path>、設 AGENT_REVIEW_FILE 環境變數，
  或先互動跑一次以記住路徑。
  ```

凡經由 **arg / env / 互動詢問**取得的 `target`，都要**寫回 state file 的 `cycle_log_file`**（見 Phase G），使後續（含自動）執行沿用，直到使用者再以 `--file=` 重新指定。

> 注意：記住的是**單一檔案**，不會隨雙週自動換檔。進入新 cycle、要寫新檔時，使用者需以 `--file=<新檔>` 重新指定一次（之後再記住新值）。

#### F2. 確保檔案存在

```python
if not os.path.exists(target):
    parent = os.path.dirname(target)
    if parent:
        os.makedirs(parent, exist_ok=True)
    # 建最小骨架：只放一個 Log 標題，當天 heading 交給 F3 補。
    # 絕不覆寫既有檔（這個分支只在檔不存在時進入）。
    with open(target, "w") as f:
        f.write("# Log\n")
```

#### F3. Find / create day heading

找 `## YYYY-MM-DD (ddd)` heading，例如 `## 2026-05-02 (Sat)`（`ddd` 為 Asia/Taipei 當天的星期縮寫）。

找不到（新檔、或當天首次寫入——現在是常態，不是例外）→ append 一個新的 day heading 到檔末，再於其下插 callout。若檔內已有其他 `## ` day heading，插在日期順序的合適位置；判斷不了就直接 append 檔末。

#### F4. Insert callout

從 today heading 開始，找該段落「最末尾」（下一個 `## ` 或 `# ` heading 前一行，皆無則檔末）。在那位置 append（前面留一行空行隔開既有內容）：

```
>[!Note] Agent 協作回顧 ({HH:MM})
>
>{合併後的 bullet 內容，每行前面加 `>` prefix}
>{空行用 `>` 不留空行}
```

`HH:MM` = Asia/Taipei 當下時間。

**多次 trigger**：每次跑都新建一個 callout，視覺上多個 callout 連在一起，各自時間戳區隔。

### Phase G：寫 state file

```json
{
  "last_run_iso": "<now UTC iso>",
  "claude_code_last_processed_ts": "<now UTC iso>",
  "cursor_last_processed_ts": "<now UTC iso>",
  "cycle_log_file": "<Phase F1 解析到的 target 絕對路徑>"
}
```

兩個 source 都統一用 `now`（簡化；下次跑時兩個 source 都從這個點往後）。

`cycle_log_file`：把 Phase F1 經 arg / env / 互動詢問取得的 `target` 寫回（**保留**既有值，除非這次有新指定）。state file 若已有舊值而這次走順序 3 沿用，原樣寫回即可。

---

## Callout 視覺範例

```markdown
## 2026-05-02 (Sat)

- [ ] 範例待辦事項
- [[範例筆記]]

>[!Note] Agent 協作回顧 (18:30)
>
>**my-vault** — 詢問跨 session 整理：今天用了哪些 agent、做了什麼。設計階段，pivot：input source 不是 git/HANDOFF 而是 user prompts。
>
>**my-shop** — 預購→兌換頁全棧：CSV 對齊金流平台模版、建多語 callback 頁、UTM 參數、deploy 兩版。

>[!Note] Agent 協作回顧 (20:00)
>
>**blog** — Astro 遷移 Phase 1：寫 scripts/convert.ts 處理舊 CMS 殘留 card，turndown 預設會丟孤兒腳註要寫自訂規則。
```

---

## 環境變數 / args

優先順序：slash arg > env var > 默認

| arg / env | 默認 | 用途 |
|---|---|---|
| `--dry-run` / `AGENT_REVIEW_DRY_RUN=1` | off | 不寫檔，只印出將要 append 的 callout |
| `--since=ISO_TS` / `AGENT_REVIEW_SINCE` | 從 state file 讀 | 強制覆寫起始時間（debug / 補跑） |
| `--file=PATH` / `AGENT_REVIEW_FILE` | 從 state file 讀；無則互動詢問 | callout 寫入的目標檔（見 Phase F1） |
| `AGENT_REVIEW_BUDGET=N` | 2000 | 字數上限

---

## 邊界 / 失敗模式

| 情境 | 行為 |
|---|---|
| state file 不存在 | 時間戳從今天 00:00 (Asia/Taipei) 開始；`cycle_log_file` 視為未設，走 F1 解析 |
| 沒有任何 session 在區間內 | 寫單行 callout `(本時段無實質 agent 協作)` |
| Subagent 全回 `(本 cwd 無實質互動)` | 同上 |
| **無法解析 target 且非互動執行** | **報錯退出，不寫任何檔**（見 F1b） |
| 目標檔不存在 | 建最小骨架（只含 `# Log`），不覆寫既有檔（F2） |
| 目標檔缺當天 day heading | append 新 heading 再插 callout（F3，常態） |
| target 來自互動詢問 / arg / env | 寫回 state file `cycle_log_file`，後續沿用 |
| 同 cwd 一天內 > 30 prompts | 截近 30 條送 subagent |
| Cursor timestamp parse 失敗 | 跳過該訊息，不終止 |
| 寫目標檔衝突（dump 同時跑） | append-only，理論不衝；若 conflict edit 失敗 → 1 秒重試 1 次 |

---

## 跟其他 skill 的關係

- **dump skill**：dump 是使用者主導的 brain dump 整理（task list、project 連結）。本 skill **append 在所有既有內容末尾**，不重排 dump 寫的東西。
- **handoff skill**：寫 `HANDOFF.md`，無 cycle log 衝突。
- **coach skill**：只讀 cycle log。本 skill 寫的 callout 反而能讓 coach 引用「W18-19 你卡在某專案的退款名單」。

---

## Cron / 自動觸發（不在 skill 內，使用者自設）

**自動執行（`--no-interactive`）必須先有可解析的 target**——它無法互動詢問。兩種做法擇一：
1. 在 cron/launchd 環境設 `AGENT_REVIEW_FILE=<path>`（最穩，明確）。
2. 先互動跑一次（會問你要寫哪個檔並記住），之後自動執行沿用 state file 的 `cycle_log_file`。

兩者皆無 → 自動執行會**報錯退出、不寫檔**（刻意：不靜默寫去錯的地方）。

範例 launchd 或 cron：
```
0 20,23 * * * AGENT_REVIEW_FILE="$HOME/path/to/current-cycle-log.md" /usr/local/bin/claude --no-interactive "/agent-day-review" >> ~/.claude/skill-state/agent-day-review/cron.log 2>&1
```

或手動 `/agent-day-review`（首次會問檔案，之後記住）。進入新 cycle 要換檔時：`/agent-day-review --file=<新檔>`。

手機 Claude Code dispatch 同樣打 `/agent-day-review`（需 state file 已有 `cycle_log_file`，否則無人應答會報錯）。
