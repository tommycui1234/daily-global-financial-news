# HTML News Dashboard via Claude Code

Generate a Bloomberg Terminal-style single-page HTML dashboard from the morning/evening news digest. This is a **supplementary visual layer** — the text report remains the primary deliverable.

## When to Use

When the user wants a visual dashboard from the existing daily news report data. Do NOT re-search or re-fetch — extract data from the already-generated cron output.

## Workflow

### 1. Extract data from cron output

```bash
ls -lt ~/.hermes/cron/output/e478ee0fe4e1/  # morning
ls -lt ~/.hermes/cron/output/144402bd8703/   # evening
```

Read the latest `.md` file, extract:
- Market data (A-share, HK, US indices with prices and change%)
- All 5 sections × 5 news items (headline + ~60-char body + source + date)

### 2. Write prompt for Claude Code

Save a `prompt.md` in `~/claudeProjects/YYYY-MM-news-dashboard/` containing:
- All market data
- All 25 news items with real text (NOT placeholders)
- Design requirements: Bloomberg dark theme, sticky header, scrolling ticker, left market cards, right 5×5 news grid
- Output: `index.html`

### 3. Run Claude Code

```bash
cd ~/claudeProjects/YYYY-MM-news-dashboard && \
  claude -p "$(cat prompt.md)" --output-format text
```

> **Note**: `claude-design` is a Claude Code skill, NOT a Hermes skill. Use the `claude` CLI directly — do NOT try to load it as a Hermes skill.

### 4. Inject real news (if needed)

If Claude Code generates placeholder cards, use Python to do targeted replacement:
```python
# Read index.html, regex-replace placeholder cards with real news
# Key: match exact whitespace and HTML structure
```

**Avoid using `read_file` from `execute_code`** for reading the HTML — it may embed line numbers into the content. Use `terminal` with Python directly.

### 5. Open & verify

```bash
open ~/claudeProjects/YYYY-MM-news-dashboard/index.html
```

## Design Spec

| Element | Detail |
|---------|--------|
| Theme | Bloomberg Terminal: #0C0C0C bg, #00FF88 green, #FFB000 amber, monospace |
| Header | Title + live BJT clock + market status indicators |
| Ticker | 9 indices scrolling, pause on hover |
| Left panel | 3 market group cards (A股/港股/美股), each index: price + change% + sparkline |
| Right panel | 5 sections × 5 news cards, each: bold amber headline + dim body + muted source/date |
| Responsive | Desktop primary, tablet ok |
| Dependencies | None — pure HTML/CSS/JS single file |

## Pitfalls

- **Don't re-search**: News data is already in the cron output. Re-searching wastes tokens and the user will correct you.
- **Use real news, not placeholders**: Claude Code may default to "新闻加载中..." cards. Always inject real data.
- **File corruption**: `execute_code` `read_file` may embed `|` characters in content. Use `terminal` + Python for file reads when doing string replacement.
- **Project directory**: Must be under `~/claudeProjects/` per CLAUDE.md convention.
