# 📰 Daily Global Financial News Digest

> 跨 Agent 全球金融新闻日报 Skill — 支持 Hermes / Claude Code / Codex / OpenClaw

自动获取 A 股、港股、美股三大市场行情数据，搜索并验证六大板块新闻（24h 优先，36h 回退），输出结构化中文日报。

---

## 🚀 跨平台使用

| Agent | 安装方式 | 触发方式 |
|-------|---------|---------|
| **Hermes** | `hermes skills tap add tommycui1234/daily-global-financial-news` | 说「生成早间/晚间新闻日报」 |
| **Claude Code** | 复制 `SKILL.md` 到 `~/.claude/skills/` | `/daily-global-financial-news` |
| **Codex (OpenAI)** | 复制 `SKILL.md` 到 `~/.codex/skills/` | 自然语言触发 |
| **OpenClaw** | `openclaw skills install tommycui1234/daily-global-financial-news` | `@agent 早间新闻` |
| **通用 CLI** | `git clone` + 按 SKILL.md 手动执行 | 将 SKILL.md 作为 system prompt 注入 |

---

## 📋 日报结构

| 板块 | 内容 |
|------|------|
| 📈 金融市场 | A股/港股/美股指数涨跌、资金流向 |
| 🏛️ 宏观经济 | 货币政策、通胀数据、GDP、美联储 |
| 💻 科技前沿 | AI、半导体、科技公司动态 |
| 🌍 地缘热点 | 国际冲突、制裁、外交事件 |
| 🏠 国内市场 | A股政策、国内监管、房地产 |
| 🇭🇰 中国香港 | 港股策略、楼市、香港经济 |

每板块 5 条新闻，每条 ~150 字，附来源和日期。

---

## 📊 数据源

| 市场 | 主数据源 | 回退方案 |
|------|---------|---------|
| A 股 | AkShare / Sina JS API | Yahoo Finance |
| 港股 | AkShare / Yahoo Finance | RTHK / AASTOCKS |
| 美股 | Yahoo Finance | Futunn / Sina Finance |
| 新闻 | Sina Feed API / Google News RSS | web_search / 浏览器抓取 |

---

## 🔑 Zero-Config — 零 API Key 依赖

**不需要注册任何账号，不需要申请任何 API Key。** 所有数据源均为公开端点或 Agent 内置能力：

| 数据源 | 类型 | 需要 API Key？ |
|--------|------|:---:|
| Sina Feed API | 公开 HTTP | ❌ |
| Sina JS API（行情） | 公开 HTTP | ❌ |
| Google News RSS | 公开 RSS | ❌ |
| Yahoo Finance (`yfinance`) | 开源库 | ❌ |
| AkShare | 开源 Python 库 | ❌ |
| 浏览器抓取 | Agent 内置 | ❌ |
| `web_search` / `web_extract` | Agent 内置 | ❌ |

唯一前提：你的 AI Agent 支持 `web_search` + `web_extract`（或等效工具），这是几乎所有主流 Agent 的标配能力。即装即用，零配置。

---

## 📁 项目结构

```
daily-global-financial-news/
├── README.md              # 本文件
├── SKILL.md               # Agent 操作指南（核心）
└── references/
    ├── sina-feed-api.md           # 新浪 Feed API 接入
    ├── google-news-rss-fallback.md # Google News RSS 回退
    ├── browser-scraping-fallback.md # 浏览器抓取回退
    ├── jina-ai-fallback.md        # Jina AI 回退
    ├── cron-anti-truncation.md    # Cron 防截断策略
    ├── self-check-validation.md   # 自动化格式验证
    └── html-news-dashboard.md     # HTML 可视化仪表盘
```

---

## ⚡ 快速开始

```bash
# Hermes
hermes skills tap add tommycui1234/daily-global-financial-news
hermes skills install daily-global-financial-news

# 其他 Agent：复制 SKILL.md 到 skills 目录后直接使用
```

---

## 📝 License

MIT
