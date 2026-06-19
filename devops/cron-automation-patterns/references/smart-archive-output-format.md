# Smart Archive Output Format — Final Single-Box Layout

## Final Output (user-approved, session 20260612)

```
📋 智能归档检查 (02:43)

┌──────────────────────────────────────────────────────┐
│  🟡 闲置对话（13） [1] 微信会话自动配置························ 2.8h│
│                  [2] 未命名························ 2.8h│
│                  [3] 未命名························ 2.6h│
│                  [4] 未命名························ 2.6h│
│                  [5] 未命名························ 2.5h│
│                  [6] Hermes Agent 桌面浏览器模拟······· 2.3h│
│                  [7] 🧠 智能归档（跨 profile） · Jun 12  1.8h│
│                  [8] 未命名························ 1.6h│
│                  [9] [dispatcher]未命名············ 1.6h│
│                  [10] [dispatcher]未命名··········· 1.3h│
│                  [11] 未命名······················· 1.2h│
│                  [12] 未命名······················· 1.2h│
│                  [13] 未命名······················· 1.2h│
└──────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────┐
│  🟢 活跃对话（4）  [1] 智能归档推送问题修复························ 1m│
│                  [2] 未命名·························· 1m│
│                  [3] 识图功能测试与配置··················· 42m│
│                  [4] Dispatcher WorkBuddy ACP Con 48m│
└──────────────────────────────────────────────────────┘

💡 回复编号归档入队并删除原会话，如：1、1 2、2+
   回复 N+ = 从第 N 个到最后全部归档入队并删除原会话
   回复 0 = 全部归档入队并删除原会话
```

## Key Implementation (make_boxed_table) — Unified make_item, [1] time left-shifted with title

```python
WIDTH = 54
LABEL_W = 18

def make_item(idx, s, fill_extra=0):
    idle = f...
    tag = f"[{s['profile']}]" if s['profile'] not in ("default","") else ""
    idx_str = f"[{idx}] "
    fill_w = (WIDTH - LABEL_W + fill_extra) - len(idx_str) - 1 - len(idle)
    title = (tag + s['title'])[:fill_w].ljust(fill_w)
    return idx_str + title + " " + idle

# Rendering
left_label = f"  {icon} {label}"

# [1] line: 13-char label + 36-char item = 49, row().ljust(54) pads 5 trailing
line1 = left_label.ljust(LABEL_W - 5) + make_item(1, sessions[0])
lines.append(row(line1))

# [2]+ line: 18 blanks + 36-char item = 54
for idx, s in enumerate(sessions[1:], 2):
    lines.append(row(" " * LABEL_W + make_item(idx, s)))
```

## Filler Strategy

Use **`·` middle-dot filler** for visible spacing between title and time in the final user-facing layout. Earlier implementations used plain spaces plus `ljust()`, but several desktop/chat renderers can make whitespace alignment hard to inspect. Middle dots make the layout stable and visually auditable.

| Situation | Problem | Current recommended fix |
|-----------|---------|-------------------------|
| Title/time gap | Plain spaces may be visually collapsed or hard to count | Fill the gap with `·` before the time value |
| [1] row offset | `[1]` is intentionally left-shifted compared with `[2]+` | Preserve the `LABEL_W - 5` first-row offset |
| [2]+ rows | Need stable right-edge alignment | Pre-size item text to `WIDTH` and avoid relying on invisible trailing spaces |

If the live script still uses plain spaces, treat that as an implementation detail to verify against screenshots; this reference records the preferred final display contract.

## CJK Width

| String | Python len | Display cells |
|--------|-----------|---------------|
| `"  🟡 闲置对话（13）"` | 12 | ~18 (CJK + emoji double-width) |
| Consequence | Label overflows 13-char left border | Acceptable — user prioritizes `[1]` offset |

## User Preference Summary

| Aspect | Rule |
|--------|------|
| Box type | Single box, NO column separator (single `┌───┐`, not `┌───┬───┐`) |
| Label row | `label + [1]` on same continuous row, NO middle `│` column split |
| `[1]` vs `[2]+` | INTENTIONALLY NOT aligned — `[1]` is 5 chars LEFT of `[2]+` |
| Time alignment | [1] time 5 cols left of right edge; [2]+ time right-aligned to `│` |
| Icons | 🟡 idle / 🟢 active |
| No column split | Row 1 is a single continuous cell |
| No truncation | Never truncate the label — offset via LABEL_W=18 |
| Whitespace survival | Use `·` filler to prevent trailing-whitespace stripping |
