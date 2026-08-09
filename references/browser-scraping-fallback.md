# Browser-Based News & Data Scraping Fallback

Use when Tavily (web_search/web_extract) AND Sina Feed API are both unavailable, or when API data is stale and needs cross-verification.

## Market Data Verification

### A-Share Closing Data
Navigate to `https://finance.sina.com.cn/stock/` — the page header shows all major indices in real-time:

- 上证综指, 深证成指, 北证50, 创业板指, 科创综指, 沪深300
- Each shows: price, absolute change, percentage change
- **This is authoritative** — use it to cross-verify stale/deep shares daily API data

### HK Market Data
Navigate to `https://finance.sina.com.cn/stock/hkstock/`:
- 恒生指数 price and change % shown in header (15-min delayed)
- Page also shows hot stock movers and key news headlines
- Search for "港股收评" in page content for the day's summary article

### US Market (not available live during Beijing daytime)
- Sina stock page shows previous close in the "美国" tab
- yfinance remains the primary US data source

## News Headline Extraction

### Method 1: browser_console JavaScript (fastest)

After navigating to a news homepage, extract headlines directly:

```javascript
// General pattern for Sina pages
Array.from(document.querySelectorAll('h2, h3, .news-item a, .list-news a, .feed-card-item a, .m-list a, .blk a'))
  .slice(0, 80)
  .map(e => e.textContent.trim())
  .filter(t => t.length > 10)
  .join('\n')
```

For tech-specific pages (`tech.sina.com.cn`):
```javascript
Array.from(document.querySelectorAll('.tech-news h3 a, .seo-data-list a, h3 a, .list-item h3 a'))
  .slice(0, 30)
  .map(e => e.textContent.trim())
  .filter(t => t.length > 8)
  .join('\n')
```

### Method 2: browser_snapshot (context-aware)

Use `browser_navigate(url)` + `browser_snapshot(full=true)` to get a text snapshot of the page accessibility tree. This captures:
- Article titles and links
- Time stamps
- Section headers

**Limitation**: Snapshots are often truncated at ~8000 chars; use browser_scroll + re-snapshot for full pages.

## Category-Specific Pages

### Sina (primary Chinese source)
| Category | URL | What it provides |
|----------|-----|-----------------|
| A-Share market | `finance.sina.com.cn/stock/` | Index data + stock news |
| HK market | `finance.sina.com.cn/stock/hkstock/` | HSI data + HK news |
| Financial news | `finance.sina.com.cn/` | All finance categories |
| Tech news | `tech.sina.com.cn/` | Technology headlines |
| World/Geopolitics | `news.sina.com.cn/world/` | International news with timestamps |
| China domestic | `finance.sina.com.cn/china/` | Macro, policy, domestic economy |

### 财联社 (cls.cn) — rich Chinese financial telegraph

| URL | What it provides |
|-----|-----------------|
| `https://www.cls.cn/telegraph` | Real-time telegraph feed with timestamps, covers all five news categories (markets, macro, tech, geopolitics, domestic). Each item is labeled with time (HH:MM:SS), source tag (e.g. 环球市场情报, 盘面直播), read/view counts. Items are chronologically ordered, newest first. |
| `https://www.cls.cn/depth?id=all` | Curated long-form articles (深度) with richer analysis. Shows a ranked hot list in the sidebar. **Best for weekend editions** — the \"周末要闻汇总\" article aggregates all key developments from Friday and Saturday into one structured piece. Also lists trending topics (1-13) in sidebar for quick story discovery across all categories. |

**Extraction pattern for telegraph** (browser_console JS):
```javascript
Array.from(document.querySelectorAll('[class*="telegraph"]'))
  .filter(el => el.textContent.includes('财联社5月'))
  .map(el => el.textContent.trim().substring(0, 300))
  .filter(t => t.length > 30)
  .slice(0, 50)
```

**Extraction for depth page** — the page loads content dynamically; `browser_snapshot` captures the hot list sidebar (ranked 1-13) with clickable article links. Click each link to read full articles, or use the snapshot sidebar list directly for headline discovery across all categories.

Each result contains: timestamp + source tag + full telegraph body. The page shows items tagged with date like `财联社5月27日电`.

**⚠️ 财联社 date caveat**: The page header shows the *last update time* (e.g. `2026.05.26 星期二 17:07`), but individual telegraph items may be from the next calendar day (e.g. items say `财联社5月27日电`). This is because CLS starts pre-loading next-day items in the evening. Trust the **item-level date**, not the page header.

### Reuters — global news (limited sections)
| URL | Status | What it provides |
|-----|--------|-----------------|
| `https://www.reuters.com/world/` | ✅ Works | Geopolitics, international affairs, conflicts |
| `https://www.reuters.com/technology/` | ❌ DataDome | Blocked after first page load |
| `https://www.reuters.com/markets/` | ❌ DataDome | Blocked after first page load |

> ⚠️ **Reuters DataDome**: Reuters applies aggressive anti-bot (DataDome) after 1–2 page loads in the same browser session. The `world/` section usually works on first visit, but subsequent navigations to `technology/` or `markets/` will be intercepted. Strategy: load `world/` first for geopolitics content, then switch to other sources for remaining categories.

### TechCrunch — reliable tech news
| URL | What it provides |
|-----|-----------------|
| `https://techcrunch.com/` | AI, startups, space, security, transportation, climate tech. Articles show category tags + timestamps ("X hours ago"). No anti-bot observed. |

**Extraction pattern** (browser_console JS):
```javascript
Array.from(document.querySelectorAll('h3 a, h2 a'))
  .map(a => a.textContent.trim())
  .filter(t => t.length > 20)
  .slice(0, 30)
```

## Priority Order for News Gathering

1. **Sina Feed API** (`feed.mix.sina.com.cn/api/roll/get`) — fastest, structured JSON
2. **web_search + web_extract** (Tavily) — only if API is responding (not 432)
3. **Browser scraping** (this doc) — when both above fail
4. **Jina AI** (`r.jina.ai/URL`) — for extracting full articles from known URLs found via browser

## Timestamp Verification

Sina news pages show publication time in format "今天 HH:MM" or "MM月DD日 HH:MM". Use these to verify articles are within the current day's 24h window. The world news page (`news.sina.com.cn/world/`) is particularly good for this — every article shows its exact timestamp.
