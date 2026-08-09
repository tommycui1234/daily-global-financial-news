---
name: daily-global-financial-news
description: "Cross-agent daily global financial news digest. Fetches A-share/HK/US market data, searches and validates news across 6 sections (24h priority, 36h fallback), outputs a structured Chinese-language report. Works with Hermes, Claude Code, Codex, OpenClaw."
version: 1.3.0
author: tommycui1234
platforms: [macos, linux]
tools:
  - web_search
  - web_extract
  - terminal
tags: [finance, daily-report, news, chinese, cross-asset]
---

# Daily Global Financial News Digest
## 全球新闻日报 — 跨 Agent 金融日报

## Cross-Agent Tool Mapping

本 Skill 涉及的工具在不同 Agent 中对应关系如下。执行时根据当前环境选择正确工具：

| 操作 | Hermes | Claude Code | Codex | OpenClaw |
|------|--------|-------------|-------|----------|
| 网页搜索 | `web_search` | `web_search` / `WebSearch` | `web_search` | `web_search` |
| 网页抓取 | `web_extract` | `WebFetch` | `fetch_url` | `web_extract` |
| 执行脚本 | `execute_code` / `terminal` | `Bash` | `terminal` / `run` | `terminal` |
| 浏览器操作 | `browser_navigate` + `browser_console` | `Browser` (Playwright) | `browser` | `browser_navigate` |
| 并行子任务 | `delegate_task` | `Task` | `delegate_task` | `delegate_task` |
| 文件读写 | `read_file` / `write_file` | `Read` / `Write` | `read` / `write` | `read_file` / `write_file` |

> ⚠️ 本 Skill 默认以 Hermes 工具命名。使用其他 Agent 时，请按上表映射。

---

## When to Use

Use this skill when asked to generate a daily global financial news report. The report covers six sections:

1. 📈 金融市场
2. 🏛️ 宏观经济
3. 💻 科技前沿
4. 🌍 地缘热点
5. 🏠 国内市场
6. 🇭🇰 中国香港

---

## Critical Time Rules

| Edition | Time (Beijing) | A/HK Data Rule | US Data Rule |
|---------|---------------|----------------|--------------|
| Morning | 08:03 | **Yesterday's close** — label 「昨日收盘」 | Last night's close (EST previous day) |
| Evening | 17:35 | **Today's close** — must be post-market, 2 decimal places | Same morning's close (EST same day) |
| Weekend | 17:35 Sat/Sun | **Friday's close** — banner: `> ⚠️ 今日A股/港股休市，以下为上周五（M月D日）收盘数据` | Friday EST close |

**Holiday rule:** If A/HK markets are closed, top banner must state `> ⚠️ 今日A股/港股休市，以下为上周五收盘数据`. Never pass off a prior day's close as today's without labeling.

**US time mapping:** US market closes at 16:00 EST = 04:00 next-day Beijing. At 08:03 Beijing, "last night" US data is the EST session that ended at 04:00 same-day Beijing.

---

## Data Fetching

### A-Shares

Primary: Sina JS API (fast, no Python dependency)

```bash
curl -sL "https://hq.sinajs.cn/?list=sh000001,sz399001,sz399006,sh000688" \
  -H "Referer: https://finance.sina.com.cn" | iconv -f gbk -t utf-8
```

Field order (verified 2026-07-13): `name, open, prev_close, current, high, low`

Python fallback: `akshare.stock_zh_index_daily(symbol="sh000001")`

### HK Indices

Primary: Sina JS API `rt_hkHSI`, `rt_hkHSTECH` — then validate with web search.
Python fallback: `yfinance` — `^HSI` and `HSTECH.HK`

### US Indices

Primary: Yahoo Finance (`^DJI`, `^GSPC`, `^IXIC`)
Fallback: Futunn or Sina Finance US stock pages via web search → web_extract.

> 🔴 **yfinance cross-verification required:** yfinance may return stale US close data without error. Always verify against web search results. If they disagree by >0.1%, use web search values.

---

## News Gathering & Validation

### Primary: Sina Feed API

Fast, structured JSON with timestamps. See `references/sina-feed-api.md`.

### Supplement: Google News RSS

English-language headlines via `curl` to `news.google.com/rss`. See `references/google-news-rss-fallback.md`.

### Fallback: web_search + web_extract

When Sina Feed API is unavailable, search with Chinese keywords for domestic/HK, English keywords (Reuters, Bloomberg, FT) for international.

### Fallback: Browser Scraping

Navigate to `finance.sina.com.cn/stock/`, `cls.cn/telegraph`, `reuters.com/world/`, `cnbc.com/technology/`. See `references/browser-scraping-fallback.md`.

### Freshness Validation

1. **Priority: 24 hours**
2. **Fallback: 36 hours** — only when a section has <5 items within 24h. Label: 「本板块24小时内资讯有限，已放宽至36小时」
3. Verify article date in URL, title, and body
4. Discard anything >36h old
5. Macro policies older than 24h: tag `[政策延续]`

---

## Report Structure

```
📰 全球新闻日报（YYYY年MM月DD日）

---

📈 一、金融市场
1. **加粗短标题**：正文约150字。来源：XXX | 日期：YYYY-MM-DD
...

🏛️ 二、宏观经济
...

💻 三、科技前沿
...

🌍 四、地缘热点
...

🏠 五、国内市场
...

🇭🇰 六、中国香港
...
```

**Formatting rules:**
- Each item: `序号. **加粗标题**：正文` — bold short headline, colon, then body
- Body: ~150 Chinese characters, factual only, no predictions
- Every item ends with `来源：XXX | 日期：YYYY-MM-DD`
- All six sections, exactly 5 items each

---

## Cross-Section Deduplication 🔴

When an event fits multiple sections, assign to the **highest priority** section:

| Priority | Section | Strategy |
|----------|---------|----------|
| 1 | 📈 金融市场 | Market moves, index changes, fund flows |
| 2 | 🏛️ 宏观经济 | Monetary policy, inflation, GDP |
| 3 | 🌍 地缘热点 | International conflicts, sanctions, diplomacy |
| 4 | 💻 科技前沿 | Tech companies, AI, semiconductors |
| 5 | 🏠 国内市场 | A-share policy, domestic regulation, real estate |
| 6 | 🇭🇰 中国香港 | HK local policy, property, non-index economic data |

Mantra: **先到先得，高优先级说了算。被抢走的板块自动找替补。**

---

## Pre-Output Checklist

1. [ ] All market data from actual API calls (not guessed)
2. [ ] A/HK data timing matches edition rule
3. [ ] Every news item within 24 hours (or 36h with label)
4. [ ] Every item has **bold short headline**
5. [ ] All six sections complete, 5 items each
6. [ ] Cross-section deduplication verified
7. [ ] Missing data labeled `[数据待核实]` — never fabricated

---

## Hermes-Specific: Cron Job Anti-Truncation

When running as a Hermes cron job, DeepSeek v4-pro may truncate output. Mitigation:
1. Write report to `/tmp/daily_news_{edition}_{date}.md` via `write_file`
2. Read back with `read_file` and output as final response
3. Add anti-summary guard: "If your response does not start with 汪汪！📰, you failed"

See `references/cron-anti-truncation.md`.

---

## Hermes-Specific: Parallel News Gathering

When `web_search`/`web_extract` are available, spawn `delegate_task` subagents in parallel:
- 3 subagents: 金融+宏观, 科技, 地缘+国内+香港
- Each returns structured JSON with title/summary/source/date
- Note: max 3 concurrent children

⚠️ Flash model trap: `delegate_task` subagents may exhaust iteration budget. If any return `max_iterations`, fall back to direct `web_search`.

---

## Pitfalls

- **yfinance stale data**: US close data can be silently wrong. Always cross-verify.
- **Sina Feed ctime**: Unix timestamp (integer), not date string. Convert with `datetime.fromtimestamp(ctime, tz=timezone(timedelta(hours=8)))`.
- **akshare timeout**: `execute_code` with akshare often times out. Use `terminal` with `curl` to Sina JS API instead.
- **HSTECH ticker**: Use `HSTECH.HK`, NOT `^HSTECH`.
- **Portal date traps**: Sina/Sohu push "on this day last year" articles. Verify year in URL.
- **`curl | python3` blocked**: Some agents block piping curl to python. Save to temp file with `-o` first.
- **Weekend A/HK**: Must check if markets are closed. Use `date` command to determine day of week.
- **US holiday Memorial Day**: US markets closed last Monday of May. Label post-holiday data as previous Friday.

---

## References

- `references/sina-feed-api.md` — Sina Feed API endpoint URLs and category LIDs
- `references/google-news-rss-fallback.md` — Google News RSS parsing code
- `references/browser-scraping-fallback.md` — Per-source extraction patterns
- `references/jina-ai-fallback.md` — Jina AI curl pattern
- `references/cron-anti-truncation.md` — Hermes cron truncation fix
- `references/self-check-validation.md` — Automated format validation script
- `references/html-news-dashboard.md` — Bloomberg-style HTML dashboard generation
