# Google News RSS — 国际新闻轻量级备选源

## When to use

当 Tavily (`web_search`/`web_extract`) 返回 432 错误且不需要启动完整浏览器时，Google News RSS 是获取英文国际新闻标题的**最快备选方案**。单次 `curl` 调用即可获取结构化 XML，覆盖金融市场、宏观经济、科技和地缘热点板块。

比浏览器抓取快 10-20 倍，比 Jina AI 更可靠（无需处理 451 封禁）。

## RSS Endpoints

```bash
# 综合国际新闻（含地缘热点）
curl -sL --max-time 15 \
  "https://news.google.com/rss?hl=en-US&gl=US&ceid=US:en"

# 商业/财经
curl -sL --max-time 15 \
  "https://news.google.com/rss/headlines/section/topic/BUSINESS?hl=en-US&gl=US&ceid=US:en"

# 科技
curl -sL --max-time 15 \
  "https://news.google.com/rss/headlines/section/topic/TECHNOLOGY?hl=en-US&gl=US&ceid=US:en"

# 全球新闻
curl -sL --max-time 15 \
  "https://news.google.com/rss/headlines/section/topic/WORLD?hl=en-US&gl=US&ceid=US:en"
```

## Parsing (in execute_code)

```python
import xml.etree.ElementTree as ET
from hermes_tools import terminal

# Fetch and parse
out = terminal("curl -sL --max-time 15 'https://news.google.com/rss?hl=en-US&gl=US&ceid=US:en'")
root = ET.fromstring(out['output'])

for item in root.findall('.//item'):
    title = item.find('title').text if item.find('title') is not None else ''
    pubdate = item.find('pubDate').text if item.find('pubDate') is not None else ''
    link = item.find('link').text if item.find('link') is not None else ''
    print(f"{title} | {pubdate} | {link}")
```

## Key properties

- **No auth required** — fully public RSS feeds
- **Returns 10-15 items** per feed
- **pubDate in RFC 2822 format** (e.g. `Fri, 29 May 2026 22:48:13 GMT`)
- **Each item** contains: `<title>`, `<link>`, `<pubDate>`, `<description>`, `<source>`
- **Title format**: `Headline text - Source Name` (e.g. `Trump says he will soon decide on Iran deal - Reuters`)
- **Source extraction**: Split title on ` - ` (last occurrence) to get source name

## Coverage mapping to report sections

| RSS Feed | Maps To |
|----------|---------|
| BUSINESS | 📈 金融市场, 🏛️ 宏观经济 |
| TECHNOLOGY | 💻 科技前沿 |
| WORLD + main feed | 🌍 地缘热点 |

## Pitfalls

- **No Chinese-language content** — 仅英文标题。需要配合新浪 Feed API 获取中文新闻。
- **Title-only** — RSS 只提供标题和短摘要，不含完整正文。如需详细内容，配合 Jina AI (`r.jina.ai`) 提取原文。
- **pubDate timezone** — 所有时间为 GMT，需加 8 小时转北京时间。直接比较即可，无需精确转换用于日报。
- **Source deduplication** — Google News RSS 会在标题末尾包含来源名（如 ` - Reuters`），多条目可能来自同一来源报导同一事件。
- **Refresh rate** — RSS 约 30-60 分钟更新一次，存在轻微延迟。
