# Jina AI Fallback: Web Extraction When Tavily Is Down

## When to use

When `web_search` and `web_extract` both fail with Tavily `432` / `5xx` errors (complete Tavily outage), Jina AI can extract full article content from known URLs without going through Tavily at all.

## Pattern

```bash
curl -sL --max-time 15 "https://r.jina.ai/https://www.stcn.com/article/detail/3924389.html" | head -100
```

The prefix `r.jina.ai/` converts any URL to markdown. No API key needed. 20 RPM limit.

## Best for

- Chinese financial news sites (stcn.com, finance.sina.com.cn, cls.cn, wallstreetcn.com)
- Known article URLs found via browser navigation or Google News
- Full article extraction when only headlines are available from other sources

## Not for

- Discovery (you need the URL first — use browser Google News or Sina homepage for discovery)
- Paywalled sites (WSJ, Bloomberg, FT — Jina gets blocked same as curl)
- High-volume parallel extraction (>20 URLs/minute)

## Complementary discovery: Browser + Google News

When `web_search` is down and you need to *discover* URLs:

```
browser_navigate → https://news.google.com/search?q=china+stock+market+2026
```

Google News returns source, headline, and age — then use Jina AI to extract the full article from the discovered URL.

## Combined workflow (Tavily-down scenario)

1. **Browser** navigate to Google News → discover URLs + headlines
2. **Browser** navigate to Sina Finance → discover Chinese headlines + URLs
3. **Jina AI** (`curl r.jina.ai/URL`) → extract full article content from known URLs
4. **Jina AI** for stcn.com article detail pages → get complete stories with dates and sources
