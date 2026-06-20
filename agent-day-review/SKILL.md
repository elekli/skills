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

env var 仍支援（給 cron / launchd 使用）：
- `AGENT_REVIEW_DRY_RUN=1` 等同 `--dry-run`
- `AGENT_REVIEW_SINCE=...` 等同 `--since=...`

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
  "cursor_last_processed_ts": "2026-05-02T10:30:00Z"
}
```

不存在或欄位缺：預設 **今天 00:00 Asia/Taipei** 對應的 UTC（避免第一次跑就回溯一週）。
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

### Phase F：找 cycle log + insert callout

#### F1. Discover cycle log

```python
import glob, re, os
from datetime import datetime, timezone, timedelta, date

# 週號慣例錨點(使用者自訂):W18 起始 = 2026-04-26(週日);雙週一檔、週日起算。
# 檔名週號與 frontmatter 日期皆以此為唯一真相,兩者不得各自為政。
ANCHOR_SUNDAY = date(2026, 4, 26)
ANCHOR_WEEK = 18

def week_range_from_filename(fp):
    """無 frontmatter 時,由檔名 W<lo>-<hi> 經錨點反推 (start, end) 日期。"""
    m = re.search(r"W(\d+)-(\d+)\.md$", fp)
    if not m:
        return None
    lo = int(m.group(1))
    start = ANCHOR_SUNDAY + timedelta(days=(lo - ANCHOR_WEEK) * 7)
    return start, start + timedelta(days=13)

today_taipei = datetime.now(timezone(timedelta(hours=8))).date()
candidates = glob.glob(os.path.expanduser("~/morphe/0 Inbox/*Q*W*-*.md"))
target = None
for fp in candidates:
    with open(fp) as f:
        head = "".join(f.readline() for _ in range(5))
    sm = re.search(r"start-date:\s*(\d{4}-\d{2}-\d{2})", head)
    em = re.search(r"end-date:\s*(\d{4}-\d{2}-\d{2})", head)
    if sm and em:
        s = datetime.fromisoformat(sm.group(1)).date()
        e = datetime.fromisoformat(em.group(1)).date()
    else:
        # fallback:空殼檔(缺 frontmatter)改用檔名週號反推範圍,
        # 避免被略過而觸發 F2 誤建出週號 +2 的重複檔。
        rng = week_range_from_filename(fp)
        if not rng:
            continue
        s, e = rng
    if s <= today_taipei <= e:
        target = fp; break
```

#### F2. Not found → create

```python
if not target:
    # 算今天前一個（含）週日。Python 的 weekday() Mon=0...Sun=6
    days_since_sun = (today_taipei.weekday() + 1) % 7  # Sun=0
    start = today_taipei - timedelta(days=days_since_sun)
    end = start + timedelta(days=13)
    
    # 檔名週號由 start 經錨點推算,與上面的 frontmatter 日期保證一致。
    # (不再掃描舊檔名取最大 W——那會讓任何一次錯標的週號往後每兩週複製一次。)
    week_offset = (start - ANCHOR_SUNDAY).days // 7
    new_lo = ANCHOR_WEEK + week_offset
    new_hi = new_lo + 1
    
    # Quarter from start month
    quarter = (start.month - 1) // 3 + 1
    
    fname = f"{start.year}Q{quarter} W{new_lo:02d}-{new_hi:02d}.md"
    target = os.path.expanduser(f"~/morphe/0 Inbox/{fname}")
    
    # 防覆寫:同名檔已存在(例如先前手建的空殼)→ 直接沿用,交給 F3/F4 append,
    # 絕不 open(target,"w") 清掉既有內容。
    if not os.path.exists(target):
        # Build content: 14 day headings reverse chrono
        lines = [
            "---",
            f"start-date: {start.isoformat()}",
            f"end-date: {end.isoformat()}",
            "---",
            "# Cycle Plan",
            "",
            "",
            "# Cycle Review",
            "",
            "",
            "# Log",
            "",
        ]
        for offset in range(13, -1, -1):
            d = start + timedelta(days=offset)
            # ddd = abbreviated weekday name (Sun, Mon, ..., Sat)
            ddd = d.strftime("%a")
            lines.append("")
            lines.append(f"## {d.isoformat()} ({ddd})")
            lines.append("")
        
        with open(target, "w") as f:
            f.write("\n".join(lines) + "\n")
```

#### F3. Find day heading

`## YYYY-MM-DD (ddd)`，例如 `## 2026-05-02 (Sat)`。

如果 cycle log 內找不到該 heading（理論上不應發生，因為建檔時 14 天都展開）→ 在最後一個 `## ...` heading 之後（或 `# Cycle Review` 之前）補插。

#### F4. Insert callout

從 today heading 開始，找該段落「最末尾」（下一個 `## ` heading 或 `# Cycle Review` 前一行）。在那位置 append（前面留一行空行隔開既有內容）：

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
  "cursor_last_processed_ts": "<now UTC iso>"
}
```

兩個 source 都統一用 `now`（簡化；下次跑時兩個 source 都從這個點往後）。

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
| `AGENT_REVIEW_BUDGET=N` | 2000 | 字數上限

---

## 邊界 / 失敗模式

| 情境 | 行為 |
|---|---|
| state file 不存在 | 從今天 00:00 (Asia/Taipei) 開始 |
| 沒有任何 session 在區間內 | 寫單行 callout `(本時段無實質 agent 協作)` |
| Subagent 全回 `(本 cwd 無實質互動)` | 同上 |
| Cycle log 不存在 | 用 template 建新檔，14 天 heading 完整展開 |
| 跨 cycle 邊界（今天剛跨進新 cycle） | 自動建新 cycle log |
| 跨 quarter 邊界 | 用 start-date 的 quarter |
| 同 cwd 一天內 > 30 prompts | 截近 30 條送 subagent |
| Cursor timestamp parse 失敗 | 跳過該訊息，不終止 |
| 寫 cycle log 衝突（dump 同時跑） | append-only，理論不衝；若 conflict edit 失敗 → 1 秒重試 1 次 |

---

## 跟其他 skill 的關係

- **dump skill**：dump 是使用者主導的 brain dump 整理（task list、project 連結）。本 skill **append 在所有既有內容末尾**，不重排 dump 寫的東西。
- **handoff skill**：寫 `HANDOFF.md`，無 cycle log 衝突。
- **coach skill**：只讀 cycle log。本 skill 寫的 callout 反而能讓 coach 引用「W18-19 你卡在某專案的退款名單」。

---

## Cron / 自動觸發（不在 skill 內，使用者自設）

範例 launchd 或 cron：
```
0 20,23 * * * /usr/local/bin/claude --no-interactive "/agent-day-review" >> ~/.claude/skill-state/agent-day-review/cron.log 2>&1
```

或手動 `/agent-day-review`。

手機 Claude Code dispatch 同樣打 `/agent-day-review`。
