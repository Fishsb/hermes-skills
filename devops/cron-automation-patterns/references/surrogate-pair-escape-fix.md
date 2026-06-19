# Surrogate Pair Escape Fix in smart-archive.py

## Discovery Date

2026-06-17, while running `smart-archive.py archive 1`

## Symptom

```
UnicodeEncodeError: 'utf-8' codec can't encode characters in position 0-1: surrogates not allowed
```

at line 789:
```python
print(f"\ud83d\udce4 [{profile}] 正在导出会话 {sid} ...")
```

## Root Cause

The Python source file contained surrogate pair escape sequences such as `\ud83d\udce4`, `\u26a0\ufe0f`, and `\ud83d\ude80` in string literals.

- `\ud83d\udce4` — the UTF-16 surrogate pair representation of U+1F4E4 (📤)
- `\u26a0\ufe0f` — U+26A0 (⚠) plus variation selector U+FE0F
- `\ud83d\ude80` — the UTF-16 surrogate pair representation of U+1F680 (🚀)

In Python 3, surrogates (U+D800–U+DFFF) are valid codepoints inside string objects but **cannot be encoded to UTF-8**. When `print()` tries to encode them for stdout, it raises `UnicodeEncodeError`.

This happens regardless of `PYTHONIOENCODING=utf-8` because surrogates are not valid UTF-8 by design (UTF-8 encodes only Unicode scalar values U+0000–U+10FFFF excluding surrogates).

## Fix

Replace each surrogate pair with the actual Unicode character or its `\U` escape:

| Before (broken) | After (fixed) |
|---|---|
| `\ud83d\udce4` | `📤` or `\U0001f4e4` |
| `\u26a0\ufe0f` | `⚠️` |
| `\ud83d\ude80` | `🚀` or `\U0001f680` |
| `\u274c` | `❌` (single — was fine) |
| `\u2705` | `✅` (single — was fine) |

The three-step fix:

1. **Identify all surrogate escapes** in the file:
   ```bash
   python -c "import re;f=open('smart-archive.py','rb').read();print(len(re.findall(rb'\\x5c\x75[0-9a-f]{4}',f)))"
   ```
   This counts all `\uXXXX` sequences.

2. **Replace surrogate pairs** (two consecutive `\uD8xx\uDCxx` sequences) with their real Unicode:
   - `\ud83d\udce4` → 📤 (U+1F4E4)
   - `\ud83d\ude80` → 🚀 (U+1F680)
   - `\ud83d\udccb` → 📋 (U+1F4CB)
   - `\ud83d\udce6` → 📦 (U+1F4E6)
   - `\ud83d\udce5` → 📥 (U+1F4E5)
   - Also check `\u26a0\ufe0f` (⚠️, surrogate+variation selector pattern)

3. **Verify** by running the script again.

## Why Not Use `errors="surrogatepass"` or `"surrogateescape"`

- `surrogatepass` only works for reading data, not for `print()` to stdout.
- `surrogateescape` only applies to bytes→str decoding errors, not encoding.
- The correct fix is to not use surrogate codepoints in Python source.

## Detection

Run the script directly from the terminal:
```bash
python smart-archive.py archive 1
```
If you see `UnicodeEncodeError: surrogates not allowed`, the script has surrogate pair escapes.

Alternatively, grep for `\\ud[89ab]` in the source file (the surrogate range in lowercase hex).

## Lesson

When writing emoji in Python source code:
- ✅ Use the literal character directly: `"📤"`
- ✅ Use `\U0001F4E4` (8-hex-digit `\U` escape)
- ❌ Do NOT use `\ud83d\udce4` (4-hex-digit `\u` surrogate pair — invalid in UTF-8)
