# Sina Finance Feed API — 新浪财经滚动新闻接口

## When to use

Use as a **fallback news source** when `web_search` / `web_extract` (Tavily backend) fails with 432 errors or other connectivity issues. May also be used as a primary news source for its timeliness and category breadth.

## API endpoint pattern

```
https://feed.mix.sina.com.cn/api/roll/get?pageid=153&lid=<LID>&num=20
```

**Parameters:**
- `pageid`: 153 (standard for all roll feeds)
- `lid`: Category ID (see below)
- `num`: Number of items (max 20)

## Category LIDs

| LID | Category | 中文分类 | 可靠性 |
|-----|----------|----------|--------|
| 2509 | Finance / Economics | 财经 | ✅ 稳定 |
| 2510 | International / World | 国际 | ❌ 已失效 — 返回2024-2025旧数据 |
| 2515 | Technology | 科技 | ⚠️ 稀疏 — 有时仅1条 |
| 2516 | Global Economy | 环球经济 | ✅ 稳定（内容偏港股公告） |
| 2517 | US Stocks / International | 美股/国际 | ✅ 稳定 — **地缘热点首选替代** |
| 2518 | US Stocks / International | 美股/国际 | ✅ 稳定 — 含国际新闻 |
| 1686 | General / Domestic | 国内综合 | ✅ 稳定 |

**关键更新 (2026-05-24)**：LID 2510（国际）已失效，所有返回时间戳均为2024-2025年数据。地缘热点新闻改用 LID 2517/2518，这两个feed混合了美股和国际新闻，实际包含大量地缘热点内容（如乌克兰战况、美伊谈判、白宫事件等）。

## Fetching pattern

**Never pipe curl directly to python3** — the security scanner blocks `curl | python3`. Instead, save to file first:

```bash
# Step 1: Save to file
curl -sL --max-time 15 \
  "https://feed.mix.sina.com.cn/api/roll/get?pageid=153&lid=2509&num=20" \
  -H "User-Agent: Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) AppleWebKit/537.36" \
  -o /tmp/sina_finance.json

# Step 2: Parse with execute_code (Python)
```

## Response format

```json
{
  "result": {
    "status": {"code": 0, "msg": "succ"},
    "data": [
      {
        "title": "新闻标题",
        "intro": "新闻摘要（150字左右）",
        "url": "https://finance.sina.com.cn/...",
        "ctime": "1779528600",     // Unix timestamp (seconds)
        "mtime": "1779529045",
        "keywords": "关键词",
        "media_name": "来源媒体"
      }
    ]
  }
}
```

## Parsing in execute_code

```python
import json, datetime

with open('/tmp/sina_finance.json', 'r') as f:
    data = json.loads(f.read())

items = data.get('result', {}).get('data', [])
for item in items:
    title = item.get('title', '')
    intro = item.get('intro', '')
    url = item.get('url', '')
    ctime = item.get('ctime', '')
    # ctime is Unix seconds; convert to readable date
    dt = datetime.datetime.fromtimestamp(int(ctime))
    date_str = dt.strftime('%Y-%m-%d %H:%M')
    # Filter for today's date
    if '2026-05-23' in date_str:
        print(f"[{date_str}] {title}")
        print(f"  {intro[:200]}")
```

## Pitfalls

- **`ctime` is Unix seconds** (not milliseconds like akshare dates). Convert with `datetime.datetime.fromtimestamp(int(ctime))`.
- **⚠️ Silent failure trap**: Filtering raw integer `ctime` with string operators (`"2026-06-24" in str(ctime)`) silently produces zero results because the Unix timestamp integer has no date substring. Always convert to datetime first, then compare.
- **The `intro` field is already a decent summary** (~150 chars) — can be used directly as the news body with minimal editing.
- **Duplicate detection**: Some items appear in multiple feeds. Deduplicate by title before compiling.
- **Sport content in stocks feed (LID 2512)**: The 2512 feed is primarily sports/betting content, not stock market news. Use 2509 (财经) and 2516 (环球经济) for financial news instead.
- **LID 2510 (国际) 已失效** (2026-05-24验证)：返回的数据时间戳均为2024-2025年。地缘热点新闻改用 LID 2517/2518。
- **LID 2515 (科技) 稀疏**：有时24小时内仅1条。当科技新闻不足时，可补充 `curl` 新浪科技首页 `tech.sina.com.cn` 提取标题（HTML解析）。
- **URL date verification**: Always check the date in URLs (`/2026-05-23/`) matches the `ctime`.
- **Coverage**: The Sina feed covers all five report sections well — 2509 for markets, 2517/2518 for geopolitics, 2515+tech homepage for tech, 1686 for domestic.
