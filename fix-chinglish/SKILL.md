---
name: fix-chinglish
description: |
  修正 Claude 生成內容中「中文夾雜英文」的情況：把不必要的英文常用詞、動詞、形容詞改回中文，
  只保留真正的術語、專有名詞、產品名、檔名、code identifier。
  當使用者說「修中英夾雜」、「fix chinglish」、「/fix-chinglish」、「把這份的英文修一下」、
  「太多英文了幫我清一下」時使用此 skill。
allowed-tools: [Read, Write, Edit, Bash, Glob, Grep]
user-invocable: true
---

# fix-chinglish

## 核心原則

1. **預設一律翻成中文**。能找到通用中文譯詞的英文常用詞，就翻；不要為了「保險」或「省 token」留英文。
2. **僅在沒有通用中文譯詞時保留原文**——專有名詞、產品名、code identifier、繁中技術寫作裡大家都直接用英文的概念（如 `text-to-SQL`、`token` 等）。
3. **目標是傳達生成內容的完整意思，不是壓縮字數**。中文需要多幾個字才能講清楚，就多幾個字。

這個 skill 不是翻譯整篇文章，是**逐句判斷哪個英文詞該留、哪個該改**，並補上中文需要的賓語、虛詞、語法調整，讓人類同事讀得順。

## 用途

把 Claude 生成內容中過度的中英夾雜，改寫成符合使用者語感的中文：

> 「Keep only true proper nouns in English (product names, file names, technical concepts
> like `git`, `hook`, `API`). Common nouns, verbs, and adjectives should be in Chinese.」
> — `~/.claude/CLAUDE.md`

## 何時用

- 使用者明確說「/fix-chinglish」、「修中英夾雜」、「把英文修一下」
- 使用者貼出一份 plan / handoff / HTML 報告，抱怨太多英文
- 工作夥伴要看的對外文件、需要使用者一字一句修過後分享給人類讀者

## 不該用

- 對方要的是**翻譯**整篇英文 → 為對方尋找替代方案
- 對方要的是 markdown → PDF / EPUB 轉檔 → 用 `pdf-tools` / 為對方尋找替代方案
- 程式碼註解、commit message、技術文件本來就慣用英文 → 不動

---

## 輸入語法

**檔案模式**：
```
/fix-chinglish <path> [--keep term1,term2,...] [--hint "free-text"]
```

**對話模式**（無 path）：
```
/fix-chinglish [--keep term1,term2,...] [--hint "free-text"]
```

`--keep` 與 `--hint` 為選用：

- `--keep`：明確指定要保留為英文的詞，逗號分隔。會 **加到** baseline keep-list，不取代。
- `--hint`：自由文字，給 skill 額外脈絡。例：`--hint "這是給非技術主管看的，技術詞越少越好"`。

範例：

- `/fix-chinglish ~/.claude/plans/example-plan.html`
- `/fix-chinglish ~/notes/draft.md --keep MCP,FastMCP,RAG`
- `/fix-chinglish` ← 修正當前對話中 Claude 最近一次的長輸出
- `/fix-chinglish ~/notes/plan.md --hint "讀者是非技術背景的同事"`

---

## 工作流程

### 步驟 1：判斷模式

- 有 `<path>` 引數 → **檔案模式**
- 無 `<path>` 引數 → **對話模式**，目標是當前對話中 Claude 最近一次的長段輸出
  （若不確定要修哪一段，反問使用者「要修哪一段？貼最後幾個字給我」）

### 步驟 2：蒐集脈絡

四個來源都盡量用上：

1. **當前對話脈絡**：你已經有完整對話歷史，知道前後在討論什麼領域、有哪些技術選型。
2. **檔案本身**：完整 `Read` 一次，掃描全文中重複出現的英文詞、產品名、code identifier，這些幾乎都該保留。
3. **同目錄關聯文件**：用 `ls <dir>` 與 `Glob` 找鄰近的 `*.md` / `CLAUDE.md` / `PRD` / `README`，必要時 `Read` 抓專案術語表與既有翻譯習慣。
4. **使用者 hint**：解析 `--keep` 與 `--hint`。

### 步驟 3：建立 keep-list

把以下三類整合成一份「**保留為英文**」清單：

**Baseline keep-list**（自動套用，不需使用者指定）：

- **專有名詞 / 產品名 / 服務名**：`ollama`、`OpenRouter`、`Anthropic`、`Claude`、`Linear`、`Tailscale`、`FastMCP`、`HackMD`、`GitHub`、`Cursor`、`Obsidian`、`Toggl`、`DuckDB`、`Vault`、`Keychain`、`Beancount`、`Readwise`、`Notion`、`Figma`、`Vercel`、`Cloudflare`
- **標準縮寫**：`LLM`、`API`、`SQL`、`HTTP`、`HTTPS`、`URL`、`MCP`、`RAG`、`ADR`、`PRD`、`DAG`、`JSON`、`CSV`、`PDF`、`EPUB`、`ETL`、`SaaS`、`IDE`、`CLI`、`CI`、`UI`、`UX`、`AWS`、`GCP`、`SSH`、`TLS`、`DNS`、`SDK`、`MVP`、`OKR`、`KPI`
- **檔名 / 副檔名 / 路徑 / 指令**：`.env`、`.md`、`.html`、`pnpm-lock.yaml`、`git`、`bash`、`zsh`、`brew`、`pnpm`、`npm`、`deno`、`bun`、`tsc`、`curl`
- **CLAUDE.md 明文列為「technical concepts」的詞**：`git`、`hook`、`API`
- **LLM / agent 領域已穩定為英文的技術概念**：`token`、`prompt`、`embedding`、`chunk`、`context`、`agent`、`hook`、`skill`、`frontmatter`、`repo`、`branch`、`commit`、`PR`、`issue`、`pull request`、`merge`、`rebase`

**從脈絡擷取**：

- 從檔案內容、對話、鄰近文件中辨認出的專案名稱、code identifier、function name、env var name。
- 若某詞在程式碼區塊（` ``` `）、inline code（`` ` ``）、HTML 的 `<code>` 標籤內出現，就視為 code identifier，自動列入 keep-list。

**使用者 `--keep`**：直接加入。

### 步驟 4：套用核心原則

**預設一律翻成中文**。能找到中文表達的詞就翻——不要為了「保險」或「省 token」留英文。

僅在以下情況保留原文：

1. **在 keep-list 中**（步驟 3 建立的清單：專有名詞、產品名、code identifier、LLM/agent 領域穩定為英文的技術概念）
2. **沒有通用中文譯詞**：在繁體中文技術寫作裡找不到被廣泛接受的譯法。判斷標準——若多數技術部落格、書籍、官方文件都使用同一個中文譯詞（例：`server` → 伺服器、`provider` → 供應商、`frontier model` → 前沿模型），就翻；若大家都直接寫英文（例：`text-to-SQL`），則保留。
3. **譯成中文會明顯失準**：API 名稱、產品功能名、規格代號、論文題目等。

**判斷順序**：

1. 是 keep-list 中的詞 / 專有名詞 / code identifier？→ **留英文**
2. 有通用中文譯詞？→ **用中文**
3. 兩可的詞？→ **預設往中文靠**（第一次只處理明確該翻的；兩可詞留待使用者第二輪指示再轉，見「注意事項」）

### 步驟 5：傳達完整語意，不壓縮字數

**目標是讓人類同事讀得順，不是讓字數最小。** 中文表達某些概念本來就比英文需要更多字。**不要為了減少字數犧牲語意。**

具體規則：

- **動詞名詞化要補賓語**：「跑 generation」單字直譯成「跑生成」缺賓語，要補成「跑生成內容」或「執行內容生成」。
- **抽象動詞要補主受詞**：英文常省略的主受詞，中文要補回。「server 持 token」中「持」的意思在英文裡含糊（持有？消耗？支付？），中文必須在脈絡中選明確動詞，並補完整：「伺服器消耗 token 會把生成內容的責任也吃進來」。
- **連接詞與虛詞要補**：英文用逗號或介系詞帶過的轉折，中文要寫出來。
- **複合名詞要拆解**：英文 noun pile 在中文必須展開為短句，不要硬塞成一長串名詞。
- **單字英文不夠用就改用詞組**：例如「scope 太大」直接寫「範圍太大」；「沒有 tradeoff」寫「沒有明顯的取捨」。

範例對照：

| 原句 | 不夠好的修法 | 正確修法 |
|------|--------------|----------|
| 本機 ollama 跑 LLM generation | 本機 ollama 跑 LLM 生成 | 本機 ollama 跑 LLM 生成內容 |
| server 持 token 會把責任也吃進來 | 伺服器持 token 會把責任也吃進來 | 伺服器消耗 token 會把生成內容的責任也吃進來 |
| Client 端決定 provider | 客戶端決定 provider | 客戶端決定模型供應商 |
| 做 generation 量級不夠 | 做生成量級不夠 | 跑生成，模型量級不夠 |

判斷補什麼賓語要靠脈絡：先讀文件其他部分，確認「在這個情境下，是要生成什麼？是消耗什麼？」再下手。脈絡不足以判斷 → 走步驟 6 標 AMBIGUOUS。

### 步驟 6：歧義處理

當你不確定一句話的意思（不只是用字，是**語意**不確定），**不要硬猜**：

1. 把該句原樣保留
2. 在輸出檔末尾加一個 `<!-- AMBIGUOUS -->` / `> [AMBIGUOUS]` 區塊，列出：
   - 行號或位置
   - 原句
   - 你想到的兩三種可能解讀
3. 在 step 8 回報時，把這幾處列出來，請使用者一句一句確認

> 範例（步驟 6 觸發時機）：「server 持 token 會把責任也吃進來」這句歧義很大——「持」可以是「持有/保管」也可以是「消耗/支付」，「責任」可以是法律責任也可以是技術責任。要在脈絡中找答案；找不到就標 AMBIGUOUS。

### 步驟 7：寫輸出

**檔案模式**：

- 輸出路徑：與原檔同目錄，檔名加 `.zh` 後綴在副檔名前。
  - `foo.html` → `foo.zh.html`
  - `bar.md` → `bar.zh.md`
  - `baz` (無副檔名) → `baz.zh`
- 檔名衝突：append `-2`、`-3` …。**永不覆蓋既有 `.zh.*` 檔**。
- 結構保留：
  - HTML：只改文本節點，不動 attribute、`<style>`、`<script>`、`<code>` 內容
  - Markdown：不動 frontmatter、`` ` `` 與 ``` ``` ``` 區塊、URL、image alt 內的識別字串
- 編碼：UTF-8，行尾與原檔一致

**對話模式**：

- 不寫檔。直接把修正後的文字輸出到對話中。
- 若原段落很長（>200 行），分段輸出，並在每段前標 `--- 段 N/M ---` 方便定位。

### 步驟 8：回報

回報必須包含：

1. **輸出位置**（檔案模式）或「修正版見上方」（對話模式）
2. **substitution 統計**：總共改了幾處，列出最常見的 3–5 種替換（如「server → 伺服器：12 次」）
3. **保留的英文詞**：列出實際決定保留的 keep-list，分兩類：
   - baseline（從清單自動套用的）
   - 從脈絡擷取的（如 `peaceful-sparking-ladybug`、`factdb`、`ETL_pipeline`）
4. **AMBIGUOUS 區塊**：若有歧義保留，列出每一處請使用者確認
5. **是否動了結構**：除了詞彙替換外，有沒有重寫整句語法（例如把「跑 generation」改成「跑生成，模型量級不夠」這種補結構的改動），條列清楚

---

## 範例（取自實際 plan）

**輸入**（HTML 中一行）：

```html
<p class="why">本機跑 generation 量級不夠（專業領域要靠 frontier model）；server 持 token 會把責任也吃進來。Client 端決定 provider 把成本與選擇還給使用者。</p>
```

**輸出**：

```html
<p class="why">本機跑生成，模型量級不夠（專業領域要靠前沿模型）；伺服器消耗 token 會把生成內容的責任也吃進來。客戶端決定模型供應商，把成本與選擇還給使用者。</p>
```

**改動說明**（給回報用）：

- `generation` → 「生成」並補主語結構：「跑生成，模型量級不夠」（補逗點、補主語）
- `frontier model` → 前沿模型
- `server` → 伺服器
- `token` → **保留**（LLM 領域穩定技術概念）
- `Client 端` → 客戶端
- `provider` → 模型供應商（脈絡是 LLM 模型供應商，補完整名稱）
- 補語意：「持 token」→ 「消耗 token」，「責任」→ 「生成內容的責任」（依文件其他段落確認語意）

---

## 設計決策

- **原則導向，不窮舉清單**：第一版曾用「必改對照表」列出 server / provider / scope ... 等詞要翻成什麼。改成原則導向後 skill 行為更穩定——任何「在繁中技術寫作有通用譯詞」的英文常用詞都該翻，不依賴對照表是否收錄。對照表會永遠不夠完整，也會讓 skill 把表外的詞當作「不用改」。
- **預設往中文靠，傳達完整語意優先**：能用中文就用中文；中文需要多幾個字才能講清楚，就多幾個字。**不要為了減少 token 用量壓縮中文語意。**
- **純 Claude 推理，不呼叫外部 API**：判斷密集的工作（哪個是術語、哪個該翻、補什麼賓語），用 LLM 翻譯 API 反而會犧牲精準度。對話脈絡與檔案脈絡都是 Claude 本來就能直接讀的。
- **不覆寫原檔**：永遠輸出到 `*.zh.*`。原檔留著供對照、撤銷、debug。即使 skill 出錯，原檔不丟。
- **AMBIGUOUS 而非硬猜**：寧可標出歧義請使用者確認，也不亂改變語意。語意錯比夾英文更糟。
- **補結構是合法操作**：中文文法需要的賓語、虛詞、語序調整都該做。不是「逐詞替換」工具，是「修中文」工具。
- **baseline keep-list 內建在 skill**：避免每次都要使用者列。可被 `--keep` 擴充，但不會被取代——baseline 永遠生效。
- **同目錄掃描**：抓不到 PRD、CLAUDE.md 也沒關係，這只是補充脈絡。對話與檔案本身是主要來源。

---

## 注意事項

- **若內容根本不是中英夾雜，而是英文段落**：這 skill 不會幫你翻譯整段。請改用 `translate-md`。skill 偵測到 >50% 是英文時應主動提醒。
- **CSS / JS / HTML attribute 不要碰**：即使裡面有英文。只動文本節點。
- **Code block 內不要碰**：即使裡面有彆扭中英夾雜的註解，也保留——code 通常是引用，不該被改寫。
- **commit / PR 標題 / log line 不要碰**：這些有自己的格式慣例。
- **若使用者後來說「我要把這篇英文比例再降低」**：可以追加一輪，把上次留下的兩可詞（如 `token`、`prompt`）也轉中文（變成「詞元」、「提示」）。但**第一次預設保留**——避免一次改太多失準。
