# Cron Job Anti-Truncation Prompt Pattern

## Problem

DeepSeek v4-pro (and similar models) in cron contexts:
1. Generates full report content in intermediate tool turns (verified in session JSON)
2. Final assistant turn output gets truncated (likely hitting `max_tokens` limit, ~4096)
3. On next turn with 100% cached context, model outputs a **summary** ("日报完成。…") instead of continuing the truncated content
4. Cron delivers this summary verbatim → user sees useless one-liner

Observed 2026-05-17: session `cron_e478ee0fe4e1_20260517_080341`, 12 API calls, final `response_len=79`, content = "日报完成。所有行情数据均来自..."

## Root Cause

Model treats its own truncation as a "task complete" signal rather than an incomplete delivery. When context is fully cached, it skips to conclusion instead of continuing output.

## Fix Pattern (3 guards)

### 1. Front-load explicit warning

```
## ⚠️ 关键警告：你的最终回复就是日报本身
你的 final response 会**直接发送给 Tommy**，没有中间步骤。
✅ 正确：以「汪汪！📰 全球新闻日报」开头的完整日报全文
❌ 错误：「日报完成。」「所有数据已获取。」—— 这都是废话，不是日报
```

### 2. Force write_file → read_file pipeline

```
## 工作流程
1. 先用 write_file 把完整日报写入 /tmp/daily_news_YYYYMMDD.md
2. 然后用 read_file 读回该文件
3. 将读取到的完整内容作为 final response 输出
```

This ensures the full report exists on disk before output, preventing truncation loss.

### 3. Anti-summary rule

```
如果日报内容太长被截断，在下一个 turn 中必须继续输出未完成的部分，
**绝对不允许**用一句总结替代剩余内容。
```

### 4. Self-check gate

```
输出前确认：
✅ final response 是否以「汪汪！📰 全球新闻日报」开头？
✅ 五个板块是否全部完整输出，每板块5条？
✅ 每条是否有加粗标题 + 来源 + 日期？
**如果你的回复不是日报全文，Tommy 会什么也看不到。**
```

## Applied to

- Morning cron: `e478ee0fe4e1` (早间全球新闻日报, 08:03 daily)
- Evening cron: `144402bd8703` (晚间全球新闻日报, 17:35 daily)
- Both updated 2026-05-17

## Verification

After applying, manually trigger `cronjob(action='run', job_id='...')` and check:
1. `agent.log`: last turn `response_len` > 200 (not 79)
2. `cron/output/{job_id}/` latest `.md`: Response section contains full report
3. `gateway.log`: delivery sent with actual content, not just metadata
