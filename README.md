# 使用 Bright Data Fetch API 在 Python 中将 URL 转换为 Markdown

使用 Python 将任意 URL 转换为供 LLM 和 RAG 使用的 Markdown、文本、HTML 或 JSON-LD。可绕过机器人拦截。无需浏览器，无依赖项。基于 [Bright Data Fetch API](https://docs.brightdata.com/products/scrapers/fetch-api/introduction?utm_source=github&utm_medium=referral&utm_content=fetch-api-scraper) 构建。新账户每月可获得 5,000 个免费额度。

[![实时检查](https://github.com/bright-cn/fetch-api-scraper/actions/workflows/live.yml/badge.svg)](https://github.com/bright-cn/fetch-api-scraper/actions/workflows/live.yml) [![上次验证](https://img.shields.io/badge/last%20verified-9%20Nov%202026-brightgreen)](https://github.com/bright-cn/fetch-api-scraper/actions/workflows/live.yml) <!-- verified: rewritten by the daily run -->

[快速入门](#如何使用-python-将-url-转换为-markdown) · [任意网站](#抓取任何网站包括会拦截机器人的网站) · [CLI](#批量-url-转-markdown-cli) · [格式](#将网站转换为供-llm-使用的-markdown文本html-和-json-ld) · [数据](#fetch-api-数据字段json) · [错误](#fetch-api-抓取工具错误) · [MCP 和 Claude Code](#fetch-api-mcp-服务器和-claude-code-技能) · [免费额度](#fetch-api-免费吗) · [文档](https://docs.brightdata.com/products/scrapers/fetch-api/introduction?utm_source=github&utm_medium=referral&utm_content=fetch-api-scraper) · [支持](#支持)

使用 Python 将任意 URL 转换为供 LLM 使用的 Markdown：发送网页 URL，即可获得页面的 Markdown、纯文本、原始 HTML 和 JSON-LD 结构化数据，以及页面标题和最终 URL。可从任意国家/地区请求并绕过机器人拦截，无需浏览器，也没有依赖项。基于 [Bright Data Fetch API](https://docs.brightdata.com/products/scrapers/fetch-api/introduction?utm_source=github&utm_medium=referral&utm_content=fetch-api-scraper) 构建。

这里还提供：用于处理 URL 列表的单命令 CLI，以及完全不需要 Python 的 [Bright Data CLI](#fetch-api-mcp-服务器和-claude-code-技能)。

## 如何使用 Python 将 URL 转换为 Markdown

需要 Python 3.10 或更高版本，无需安装任何内容：这里使用的是标准库。

```bash
export BRIGHTDATA_API_TOKEN=YOUR_API_KEY
```

从 [Bright Data 控制面板](https://www.bright.cn/cp/setting/users)获取令牌。还没有账户？[创建账户](https://www.bright.cn/cp/start)；新账户每月可获得 [5,000 个免费额度](https://docs.brightdata.com/general/account/billing-and-pricing/free-tier)。

```python
import json
import os
import time
import urllib.request

API = "https://api.brightdata.com/datasets/v3"
HEADERS = {
    "Authorization": f"Bearer {os.environ['BRIGHTDATA_API_TOKEN']}",
    "Content-Type": "application/json",
}


def call(path, body=None):
    data = None if body is None else json.dumps(body).encode()
    request = urllib.request.Request(API + path, data=data, headers=HEADERS)
    with urllib.request.urlopen(request) as response:
        return json.load(response)


page = {"url": "https://en.wikipedia.org/wiki/Web_scraping", "country": "US"}
job = call("/trigger?dataset_id=gd_m6gjtfmeh43we6cqc&include_errors=true", [page])
while call(f"/progress/{job['snapshot_id']}")["status"] not in ("ready", "failed"):
    time.sleep(5)
record = call(f"/snapshot/{job['snapshot_id']}?format=json")[0]
if "error" in record:
    raise SystemExit(record["error"])
print(record["page_title"], "|", len(record["markdown"]), "characters of Markdown")
print(next(line for line in record["markdown"].splitlines() if line.startswith("# ")))
```

```
Web scraping - Wikipedia | 68572 characters of Markdown
# Web scraping
```

[按页面](https://www.bright.cn/pricing/web-scraper)计费。

该代码片段中的两项设置并非可选。

`include_errors=true` 会让 API 将无法抓取的页面、404 页面或不存在的主机报告为包含 `error` 键的记录行。API 默认关闭此选项，关闭后，页面抓取失败看起来会与没有页面相同。

`country` 是可选项。请以大写形式发送；详见[从其他国家/地区获取页面](#从其他国家地区获取页面)。

## 抓取任何网站，包括会拦截机器人的网站

Fetch API 可以抓取任何公开网站，包括会拦截机器人的网站。它会通过 Bright Data 的解锁功能处理请求：你无需使用无头浏览器、配置代理或破解验证码。2026 年 11 月 9 日，它在 12 秒内返回了一个 Zillow 搜索页面，该网站会拦截大多数爬虫工具，并获得了 9,308 个字符的 Markdown 及页面的 JSON-LD。

唯一凭据是 Bright Data API 密钥，通过 `BRIGHTDATA_API_TOKEN` 设置。此软件包及其命令也会查找由 `bdata login` 存储的密钥。

## 批量 URL 转 Markdown CLI

此代码库中的命令会同时对多个 URL 运行快速入门流程，并写入一个 JSON 文件。

```bash
pip install git+https://github.com/bright-cn/fetch-api-scraper
fetch-api-scraper https://example.com https://en.wikipedia.org/wiki/Web_scraping --country US
```

```
Fetching 2 page(s) from US
Usually under a minute each. Billed per page, 5,000 free credits a month.
asking  https://example.com...
got     https://example.com: 'Example Domain', 156 characters of Markdown
asking  https://en.wikipedia.org/wiki/Web_scraping...
got     https://en.wikipedia.org/wiki/Web_scraping: 'Web scraping - Wikipedia', 68,572 characters of Markdown

Saved 2 pages as JSON to pages.json
```

在终端中，`asking` 行会被原地更新的旋转进度指示器和计时器取代，让你可以看到任务正在运行。

```
--country CC   two-letter country to request the pages from, such as US
--out PATH     output file, default pages.json
```

`python -m fetch_api_scraper` 同样有效。该命令会从 `BRIGHTDATA_API_TOKEN` 中查找令牌；如果你运行过一次 `npx -p @brightdata/cli bdata login`，则会从已存储的登录信息中查找，供你以及该终端中的任何编程智能体使用。

也可以导入它而不是直接运行，从而为每个 URL 获取 `ok` 和 `error`，而不是原始记录行。`scrape` 不会因为某个 URL 出错而引发异常；读取 `page` 前，请先检查 `ok`：

```python
from fetch_api_scraper import scrape

for outcome in scrape(["https://example.com", "https://example.com/this-page-does-not-exist"]):
    if outcome.ok:
        print(f"{outcome.url}: {outcome.page['page_title']}")
    else:
        print(f"{outcome.url} failed: {outcome.error}")
```

```
https://example.com: Example Domain
https://example.com/this-page-does-not-exist failed: The navigation resulted in a dead page (404 status code)
```

## 将网站转换为供 LLM 使用的 Markdown、文本、HTML 和 JSON-LD

Fetch API 是一种只有一个输入的抓取工具，具体说明见 [Fetch API 文档](https://docs.brightdata.com/products/scrapers/fetch-api/introduction?utm_source=github&utm_medium=referral&utm_content=fetch-api-scraper)。下面的每个代码片段都完整可用：在完成上述 `pip install` 后，直接按原样粘贴即可。每个片段都会在每周一通过 Actions 运行，每隔一天还会进行规模较小的检查，顶部徽章显示的是最新结果。

| 你的需求 | 操作方式 |
| --- | --- |
| 将页面转换为供 LLM 使用的 Markdown | `markdown(url)` |
| 获取多个 URL 的所有格式 | `pages([url, url])` |
| 从其他国家/地区获取页面 | `pages(url, "DE")` |

`pages` 接受一个 URL 或 URL 列表。一个列表只会创建一个作业，而不是每个 URL 创建一个作业。每个页面对应一条记录，API 无法抓取的页面会作为包含 `error` 键的记录行返回。

### 将网页转换为供 LLM 使用的 Markdown

```python
from pathlib import Path

from fetch_api_scraper import markdown

text = markdown("https://docs.python.org/3/tutorial/index.html")
Path("tutorial.md").write_text(text, encoding="utf-8")
print(len(text), "characters saved to tutorial.md")
heading = next(line for line in text.splitlines() if line.startswith("# "))
print(heading.split("[")[0])  # keep the text, drop the pilcrow link after it
```

```
15117 characters saved to tutorial.md
# The Python Tutorial
```

`markdown` 包含整个页面，包括导航和侧边栏，而不仅仅是文章。API 返回错误行而不是页面时，`markdown` 会引发异常。

### 获取原始 HTML 和最终 URL

```python
from fetch_api_scraper import pages

page = pages("http://example.com")[0]
print("asked for: http://example.com")
print("landed on:", page["final_url"])
print(len(page["page_html"]), "characters of HTML,", len(page["html2text"]), "of plain text")
```

```
asked for: http://example.com
landed on: http://example.com/
577 characters of HTML, 156 of plain text
```

`final_url` 是页面经过重定向后最终到达的位置。`html2text` 是页面的纯文本，`page_html` 是 HTML。

### 从页面提取 JSON-LD 结构化数据

```python
import json

from fetch_api_scraper import pages

page = pages("https://en.wikipedia.org/wiki/Web_scraping")[0]
found = json.loads(page["ld_json"])
article = found[0] if isinstance(found, list) else found
print(article["@type"], "|", article["name"], "|", article["dateModified"][:10])
```

```
Article | Web scraping | 2026-11-01
```

`ld_json` 是页面的 schema.org 数据，以 JSON 字符串形式提供，仅在页面包含相关数据时存在。在 Wikipedia 上，它会解析为对象列表。

### 从其他国家/地区获取页面

```python
from fetch_api_scraper import pages

for country in ("US", "DE", "JP"):
    page = pages("https://ipinfo.io/country", country)[0]
    print(country, "->", page["markdown"].strip())
```

```
US -> US
DE -> DE
JP -> ES
```

https://ipinfo.io/country 会返回请求来源的国家/地区，因此可以显示 Fetch API 从哪里获取页面。2026 年 11 月 9 日，10 次此类请求中有 8 次来自指定的国家/地区。JP 有一次、FR 有一次来自西班牙，而记录中的 `country` 仍会回显指定的国家/地区，因此它不能证明页面实际是从哪里获取的。

### 在一个作业中获取多个 URL

```python
from fetch_api_scraper import pages

urls = ["https://example.com", "https://www.zillow.com/homes/Seattle,-WA_rb/"]
for page in pages(urls):
    print(page["page_title"][:50], "|", len(page["markdown"]), "characters")
```

```
Example Domain | 156 characters
Seattle WA Real Estate - Seattle WA Homes For Sale | 9308 characters
```

第二个 URL 是 Zillow 搜索页面，与 example.com 位于同一个作业中，并成功绕过其机器人拦截完成抓取。

### 立即触发，稍后获取

对于很长的 URL 列表，不要让进程阻塞一小时。触发作业、保存快照 ID，准备就绪后再获取。快照可供下载 30 天。

```python
import time

from fetch_api_scraper import FETCH_API, fetch_snapshot, status, trigger

snapshot = trigger(FETCH_API, [{"url": "https://example.com"}])
print("snapshot:", snapshot)
while (state := status(snapshot)) not in ("ready", "failed"):
    time.sleep(5)
print("status:", state)
print("fetched:", fetch_snapshot(snapshot)[0]["page_title"])
```

```
snapshot: sd_mw9c298n2l0bbx3d47
status: ready
fetched: Example Domain
```

截至 2026-11-09，实时 API 对数据集 `gd_m6gjtfmeh43we6cqc` 接受的输入字段：

| 输入 | 必填 | 作用 |
| --- | --- | --- |
| `url` | 是 | 要获取的页面 |
| `country` | 否 | 用于获取页面的两字母国家/地区代码，使用大写 |

`bdata pipelines info gd_m6gjtfmeh43we6cqc collect_by_url` 会输出 API 验证的输入字段。

## Fetch API 数据字段（JSON）

大多数人希望从页面中获取的字段：

```
page_title  markdown  final_url  html2text  page_html  ld_json  country
```

代码不会硬编码字段列表。API 返回的所有内容都会写入每条记录和命令生成的文件中。

<!-- fields:start -->
此表每天根据已发布的数据结构 [scrapers-full.json](https://docs.brightdata.com/scrapers-full.json) 重新生成，因此不会过时。每条记录包含适用于该记录的字段，以及 `timestamp` 和 `input`；示例文件的第一个页面包含 9 个键。

<details>
<summary>全部 8 个页面字段，<code>gd_m6gjtfmeh43we6cqc</code></summary>

| 字段 | 类型 | 说明 |
| --- | --- | --- |
| `markdown` | html2markdown | 以 Markdown 格式提取内容，保留标题、列表和链接等基本格式，同时移除复杂的 HTML 结构。 |
| `url` | url | url |
| `final_url` | url | 最终 URL |
| `html2text` | html2text | 仅提取页面的原始文本，去除所有 HTML 标签和格式。 |
| `page_html` | html2html | 提取页面的完整 HTML 源代码，包括所有标签、脚本和样式。 |
| `ld_json` | html2ldjson | 以 JSON-LD 格式提取结构化元数据，常用于 SEO 和富媒体摘要（例如 schema.org 数据）。 |
| `page_title` | text | 页面标题 |
| `country` | country | 输入的国家/地区 |

</details>
<!-- fields:end -->

<details>
<summary>来自 <code>fetch-api-scraper https://example.com</code> 的真实输出文件开头部分</summary>

```json
{
  "generated_at": "2026-11-09T14:05:12.630142+00:00",
  "pages": [
    {
      "url": "https://example.com",
      "page": {
        "markdown": "This domain is for use in documentation examples without needing permission. This is not a service; avoid relying on it for testing and monitoring purposes.",
        "url": "https://example.com/",
        "final_url": "https://example.com/",
        "html2text": "This domain is for use in documentation examples without needing permission. This is not a service; avoid relying on it for\ntesting and monitoring purposes.",
        "page_html": "<!doctype html><html lang=en><head><meta charset=utf-8><link rel=icon href=data:,><meta name=viewport content=\"width=device-width,initial-scale=1\"><title>Example Domain</title><style>html{color-scheme:light dark;background:light-dark(#eee,#222)}body{font:16px/1.6 system-ui,sans-serif;max-width:26em;margin:auto;padding:25vh 2em 2em;text-align:center}</style></head><body><p>This domain is for use in documentation examples without needing permission. This is not a service; avoid relying on it for testing and monitoring purposes.</p><script src=/s.js></script></body></html>\n",
        "page_title": "Example Domain",
        "country": null,
        "timestamp": "2026-11-09T14:05:03.000Z",
        "input": {
          "url": "https://example.com"
        }
      }
    }
  ]
  ...
```

包含一个页面及其所有字段的完整文件位于 [examples/sample_output.json](examples/sample_output.json)。

</details>

## Fetch API 抓取工具错误

| 你看到的内容 | 含义 |
| --- | --- |
| `API token required but not found.` | 在发出任何请求前以状态码 2 退出。请设置令牌或运行 `bdata login`。 |
| `failed  URL: The navigation resulted in a dead page (404 status code)` | 以状态码 1 退出。页面不存在。 |
| `failed  URL: Crawler error: Navigation failed ... Could not resolve host` | 以状态码 1 退出。域名不存在。API 用了 147 秒才返回此结果。 |
| `failed  URL: timeout: snapshot ... not ready after 900s` | 以状态码 1 退出。请再次运行，或稍后使用 `fetch_snapshot()` 获取快照。 |
| `failed  URL: HTTP 401: ...` | 已设置令牌，但令牌无效。 |

任何失败都会以状态码 1 退出，因此可以安全地将运行结果用作脚本执行条件。

通过函数调用时，相同情况如下所示：

| 你看到的内容 | 含义 |
| --- | --- |
| `TokenMissing` | 没有 `BRIGHTDATA_API_TOKEN`，也没有 `bdata login`。 |
| `ApiError: HTTP 400: ...validation_error...` | API 不接受的输入，例如格式错误的 URL。 |
| 包含 `error` 键的记录行 | API 对某个 URL 的响应：`dead_page`；如果主机无法解析，则为 `proxy`。其他记录行不受影响。 |
| `TimeoutError` | 作业超过了 900 秒的等待时间。将 `timeout=` 传给 `run`，或者使用 `trigger` 并稍后获取。 |

## Fetch API MCP 服务器和 Claude Code 技能

无需 Python，也无需安装任何内容。粘贴这两行；第一行会打开一次浏览器，也可以通过 SSH 和 CI 使用 `bdata login --device`：

```bash
npx -p @brightdata/cli bdata login
npx -p @brightdata/cli bdata pipelines run gd_m6gjtfmeh43we6cqc collect_by_url "https://example.com" --pretty
```

2026 年 11 月 9 日的运行结果：1 条记录、8 个字段、耗时 26 秒。CLI 会省略值为 null 的字段。

`npx skills add brightdata/skills` 会向 Claude Code、Cursor 和 Codex 说明这些命令及文档，之后即可使用自然语言操作。完整指南：[面向编程智能体的 Bright Data](https://docs.brightdata.com/quickstart-coding-agent)。

要在没有终端的托管式助手中使用？[Bright Data MCP 服务器](https://github.com/bright-cn/brightdata-mcp#which-tool-to-use) 可通过其 `scrapers` 组中的 `search_scrapers` 和 `run_scraper`，运行包括 Fetch API 在内的任何预构建爬虫工具：

    https://mcp.brightdata.com/mcp?token=YOUR_API_TOKEN&groups=scrapers

智能体还可以自行创建账户，无需填写注册表单：[智能体注册](https://brightdata.com/auth.md)。Bright Data 还可连接从 LangChain 到 Zapier 和 n8n 的其他服务：[集成](https://docs.brightdata.com/integrations/introduction)。

## Fetch API 免费吗？

新的 Bright Data 账户每月可获得 [5,000 个免费额度](https://docs.brightdata.com/general/account/billing-and-pricing/free-tier)。Fetch API 按记录计费，每个页面对应一条记录。[定价](https://www.bright.cn/pricing/web-scraper)中列出了费率。

## 支持

此代码库中的错误：[创建 issue](https://github.com/bright-cn/fetch-api-scraper/issues)，[CONTRIBUTING.md](CONTRIBUTING.md) 说明了需要提供的信息。任何与 API、你的账户或额度相关的问题，请联系 [Bright Data 支持团队](https://brightdata.zendesk.com/hc/en-us/requests/new)。

## 许可证

[Apache 2.0](LICENSE)。

## Node.js URL 转 Markdown

Node.js 也提供了相同的 URL 转 Markdown 爬虫工具：[bright-cn/fetch-api-scraper-node](https://github.com/bright-cn/fetch-api-scraper-node)。

## Bright Data 的更多开源爬虫工具

其他以相同方式构建并包含实时验证示例的 Python 抓取工具：

- [Python ChatGPT 爬虫工具](https://github.com/bright-cn/chatgpt-scraper)
- [Python Amazon 抓取工具](https://github.com/bright-cn/amazon-scraper-python)
- [Python LinkedIn 爬虫](https://github.com/bright-cn/linkedin-scraper-python)
