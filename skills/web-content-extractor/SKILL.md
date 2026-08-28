---
name: web-content-extractor
description: >
  从网页 URL 提取正文内容并结构化。可抓取文章正文、标题、作者/日期等元信息，去掉导航广告等噪音；把长文章转成纯文本/Markdown 供模型阅读；处理登录墙、重定向、分页。适合让 agent"读这篇文章"、"抓这个页面内容"、"总结这个网页"。
  当用户给一个 URL 要求读/总结/抓取内容，或需要获取某网页正文用于分析时触发。
license: MIT
compatibility: 无本地依赖也可用(直接 requests/浏览器)；如需正文清洗建议 python3 + trafilatura/readability-lxml
---

# 网页正文提取

把 URL 的网页抓下来并提炼成可读、可喂给模型的正文。核心目标：**去噪音、保正文、带元信息。**

## When to use

- 用户丢一个链接说"读一下""总结""这文章讲什么"
- 需要抓取网页正文做下一步分析（翻译、转发、做报告）
- 页面含大量导航/广告/推荐，直接抓 HTML 又乱又占 token

## 依赖检查

```bash
python3 -c "import requests" 2>/dev/null && echo "requests OK" || echo "need: pip install requests"
python3 -c "import trafilatura" 2>/dev/null && echo "trafilatura OK" || echo "need: pip install trafilatura (可选,正文清洗最稳)"
```

## 最简抓取（不带清洗，适用多数情况）

```bash
python3 - <<'EOF'
import requests, re
url = "https://example.com/文章"
r = requests.get(url, timeout=15, headers={"User-Agent": "Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7)"})
r.encoding = r.apparent_encoding   # 防中文页乱码
html = r.text
# 去掉 script/style 等噪音
html = re.sub(r"<(script|style|noscript)[^>]*>.*?</\1>", " ", html, flags=re.S)
text = re.sub(r"<[^>]+>", " ", html)          # 去标签
text = re.sub(r"\s+", " ", text).strip()     # 合并空白
print(text[:4000])
EOF
```

## 高质量正文清洗（推荐 trafilatura）

trafilatura 自动识别正文、去掉导航广告、保留段落结构，中文效果好：

```bash
python3 - <<'EOF'
import trafilatura
downloaded = trafilatura.fetch_url("https://example.com/文章")
text = trafilatura.extract(downloaded, include_comments=False,
                           include_tables=True, with_metadata=True,
                           output_format="markdown")
print(text[:6000])
EOF
```

需要结构化元信息（标题/作者/日期）：

```python
import trafilatura
d = trafilatura.fetch_url("文章URL")
md = trafilatura.extract_metadata(d)
print(md)   # 含 title / author / date / sitename
body = trafilatura.extract(d, output_format="markdown")
```

## 处理反爬 / 登录墙 / 重定向

- **带 headers**：很多站拒绝默认 UA，换上浏览器 UA 常用（见上代码块）
- **重定向**：requests 默认跟随，看 `r.url` 确认最终落点
- **登录墙**：无头浏览器（Playwright/浏览器工具）先登录再取正文；纯 requests 拿不到的告知用户需要登录态
- **限速/超时**：设 `timeout`，别无限等；失败重试 1-2 次
- **403/429**：减速并加 Referer，仍不行就换浏览器自动化抓

## 截断与分页

长文章分页则抓全部页再拼：

```python
import requests, re
def clean(url):
    r = requests.get(url, timeout=15, headers={"User-Agent":"Mozilla/5.0"})
    html = re.sub(r"<(script|style)[^>]*>.*?</\1>", " ", r.text, flags=re.S)
    return re.sub(r"\s+", " ", re.sub(r"<[^>]+>", " ", html)).strip()

pages = [clean(f"文章URL?page={i}") for i in range(1, 6)]   # 按需确认页数
full = "\n\n".join(pages)
print(full[:8000])
```

## 交给模型回答

- 把清洗后的正文（按上下文限制截断）喂给模型，让模型读并回答
- 带元信息（标题/作者/日期）能帮模型判断来源与时效
- 页面是列表/表格类，把对应结构原文给模型，别丢信息
- 抓取失败也如实告诉用户（"该页需登录/返回403"），别编内容

## 常见坑

- **中文乱码**：一定设 `r.encoding = r.apparent_encoding`，或从 `charset` meta 读
- 深色/JS渲染页：requests 拿不到 DOM，需浏览器工具（等待渲染后再取）
- 换行丢失：正文挤一行行太长，用 `split("\n")` 按段分发或多页处理
- 隐私/版权：只抓用户明确要的内容，别爬接口、别抓需要凭证的页
- 超大正文：先给模型标题+目录+前若干段，别一次灌爆

## 原则

**抓得到就给全文让模型读，抓不到就如实说明，绝不编造页面内容。** 清洗的价值是省 token、不误导，不是改写原文。
