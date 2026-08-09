# Automated Self-Check Validation Pattern

Use this `execute_code` regex-based validation after writing the digest to `/tmp/daily_news_YYYYMMDD.md` and before delivering as final response. Catches format violations before delivery.

## When to Use

Every news digest generation. Run after `write_file`, before output.

## Validation Script

```python
import re
import sys

filepath = sys.argv[1] if len(sys.argv) > 1 else '/tmp/daily_news_YYYYMMDD.md'

with open(filepath, 'r') as f:
    content = f.read()

checks = {}

# 1. Opening line — must start with 汪汪！📰 全球新闻日报 or 📰 全球新闻日报
checks['starts_with_opening'] = (
    content.strip().startswith('汪汪！📰 全球新闻日报') or
    content.strip().startswith('📰 全球新闻日报')
)

# 2. Count bold titles (each news item: 序号. **bold title**：)
# Matches both Chinese ：and English : after bold title
bold_titles = re.findall(r'\d+\.\s+\*\*[^*]+\*\*[：:]', content)
checks['bold_title_count'] = len(bold_titles)

# 3. Count sections (6 required, handles both ##-prefixed and plain emoji headers)
# Plain format: 📈 一、金融市场 (no ## prefix)
# Markdown format: ## 📈 一、金融市场
emoji_list = '[📈🏛💻🌍🏠🇭🇰]'
plain_sections = re.findall(rf'^{emoji_list}\s+[一二三四五六]、', content, re.MULTILINE)
md_sections = re.findall(rf'##\s+{emoji_list}', content)
checks['section_count'] = max(len(plain_sections), len(md_sections))

# 4. Count source+date lines
source_date_lines = re.findall(r'来源：[^|]+\|\s*日期：\d{4}-\d{2}-\d{2}', content)
checks['source_date_count'] = len(source_date_lines)

# 5. Market data present (both A-share and US indices)
checks['has_a_share_data'] = '上证指数' in content and '深证成指' in content
checks['has_hk_data'] = '恒生指数' in content or '恒生科技' in content
checks['has_us_data'] = '纳斯达克' in content or '道琼斯' in content or '标普500' in content

# 6. Per-item bold check — every numbered item must have **...** in its title
all_numbered = re.findall(r'^(\d+)\.\s+(.+?)$', content, re.MULTILINE)
bold_failures = [(n, line[:60]) for n, line in all_numbered if '**' not in line[:80]]

# 7. Deduplication checks (cross-section repetition prevention)
dedup_issues = []

# HK section should not repeat index moves already in 金融 section
hk_start = content.find('🇭🇰')
hk_content = content[hk_start:] if hk_start > 0 else ''
pre_hk = content[:hk_start] if hk_start > 0 else content
if '恒指' in hk_content and '恒指' in pre_hk:
    dedup_issues.append('恒指 appears in both 金融 and 🇭🇰 sections')
if '恒生科技' in hk_content and '恒生科技' in pre_hk:
    dedup_issues.append('恒生科技 appears in both 金融 and 🇭🇰 sections')

# Domestic (🏠) should not repeat A-share index moves
domestic_start = content.find('🏠')
domestic_content = content[domestic_start:hk_start] if domestic_start > 0 and hk_start > 0 else ''
if '上证指数' in domestic_content or '沪指' in domestic_content:
    dedup_issues.append('A-share index move in 🏠 section (should be in 金融 only)')

# 8. Per-section item count
section_names = ['📈 一、金融市场', '🏛️ 二、宏观经济', '💻 三、科技前沿',
                 '🌍 四、地缘热点', '🏠 五、国内市场', '🇭🇰 六、中国香港']
section_item_counts = {}
for i, name in enumerate(section_names):
    start = content.find(name)
    end = content.find(section_names[i+1]) if i < len(section_names)-1 else len(content)
    sec = content[start:end]
    section_item_counts[name] = len(re.findall(r'^\d+\.\s', sec, re.MULTILINE))

# --- Output ---
print("=== SELF CHECK RESULTS ===")
for k, v in checks.items():
    print(f"  {k}: {v}")

print(f"\n  Per-section item counts: {section_item_counts}")
print(f"  Bold titles found: {checks['bold_title_count']}")
print(f"  Source+date lines: {checks['source_date_count']}")

if bold_failures:
    print(f"\n❌ {len(bold_failures)} ITEMS WITHOUT BOLD TITLE:")
    for num, preview in bold_failures:
        print(f"  Item {num}: {preview}...")
else:
    print("\n✅ All items have bold titles")

if dedup_issues:
    print(f"\n❌ DEDUP ISSUES ({len(dedup_issues)}):")
    for issue in dedup_issues:
        print(f"  - {issue}")
else:
    print("\n✅ No cross-section dedup violations")

# Pass threshold
passed = all([
    checks['starts_with_opening'],
    checks['bold_title_count'] == 30,
    checks['section_count'] == 6,
    checks['source_date_count'] == 30,
    checks['has_a_share_data'],
    checks['has_hk_data'],
    checks['has_us_data'],
    len(bold_failures) == 0,
    len(dedup_issues) == 0,
    all(v == 5 for v in section_item_counts.values()),
])

print(f"\n{'✅ PASS' if passed else '❌ FAIL'} — {'ready to deliver' if passed else 'fix before output'}")
```

## Pass Criteria

- `starts_with_opening` = True
- `bold_title_count` = 30 (6 sections × 5 items)
- `section_count` = 6
- `source_date_count` = 30
- `has_market_table` = True
- `bold_failures` = 0

## Integration

1. Write digest to `/tmp/daily_news_YYYYMMDD.md`
2. Run the validation script via `execute_code`
3. If PASS → read the file and output as final response
4. If FAIL → fix the violations, re-validate, then output
