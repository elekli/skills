# skills

Claude Code 個人技能集。每個技能一個目錄，內含 `SKILL.md`。

| 技能 | 用途 |
|---|---|
| [`agent-day-review`](agent-day-review/SKILL.md) | 跨 agent（Claude Code + Cursor）增量整理當日 session，append 進當週 cycle log，作為晚間 review 的事實底稿。 |
| [`fix-chinglish`](fix-chinglish/SKILL.md) | 修正生成內容裡不必要的中英夾雜，只保留真正的術語、產品名、code identifier。 |

## 安裝

把目錄放進 `~/.claude/skills/`：

```bash
cp -r agent-day-review fix-chinglish ~/.claude/skills/
```

技能內的路徑（如 cycle log 位置）按各自工作流調整。
