# 供在此代码库中工作的编程智能体使用

更改任何内容前，请先阅读本文。以下每一行均已于 2026-11-09 根据实时 API 或 Bright Data CLI 0.4.1 验证。如果 API 参考页面与实时 API 不一致，以此处记录的实时 API 为准。

## 身份验证

- `token()` 先读取 `BRIGHTDATA_API_TOKEN`，然后读取存储在 `brightdata-cli/credentials.json` 中的密钥 `bdata login`（Python SDK 读取的也是同一个文件）。设置该变量或运行一次登录，是人工必须完成的唯一一步；切勿将密钥粘贴到代码中。
- README 中的原始 HTTP 快速入门仅会读取 `BRIGHTDATA_API_TOKEN`，无法获取 CLI 登录信息。

## API 的工作方式

- 一个数据集 `gd_m6gjtfmeh43we6cqc`，仅按 URL 采集。输入：`url`，可选 `country`。
- 产品名称是 Fetch API。不要写成“Crawl API”，也不要使用 Python SDK 中的 `client.crawler.crawl`：Bright Data 的文档会避免使用这两个名称。
- 此软件包使用标准库调用 `https://api.brightdata.com/datasets/v3`，且没有任何依赖项。快照下载使用 `fetch_snapshot`，而不是 `fetch`，因此不会与产品混淆。
- 每次调用都是异步作业：`/trigger`，轮询 `/progress/<id>`，然后执行 `/snapshot/<id>?format=json`。页面处理耗时 7 到 44 秒；未知主机在 147 秒后返回失败。`TIMEOUT` 为 900。
- 如果状态轮询因网络错误失败，则会持续重试，直至达到截止时间。HTTP 错误仍会引发异常。
- 始终发送 `include_errors=true`。404 会作为错误行返回，即 `dead_page`；无法解析的主机会作为错误行返回，其 error_code 为 `proxy`。
- 截至 2026-11-09 的实时行为：
  - 每条记录包含：`markdown`、`html2text`、`page_html`、`page_title`、`url`、`final_url`、`country`；仅当页面存在相关内容时才包含 `ld_json`。
  - `ld_json` 是 JSON 字符串；在 Wikipedia 上，它会解析为列表。
  - `markdown` 是整个页面，包括导航内容。
  - `country`：请以大写形式发送。省略时，记录中的 `country` 为 null。使用 https://ipinfo.io/country 检查后，10 次请求中有 8 次来自请求指定的国家/地区，另有 2 次（JP 一次、FR 一次）来自西班牙，但记录中的 `country` 仍会回显输入值。它不能证明出口所在的国家/地区。
  - 一个受机器人防护的 Zillow 搜索页面在 12 秒内返回。
- 数据结构可能随时更改，切勿硬编码字段列表。
- 完整文档索引，每个条目对应一个 `.md` 页面：https://docs.brightdata.com/llms.txt

## 在此代码库中工作

- `pytest` 可离线运行且不需要令牌。`ruff check .` 必须通过。
- CI 会在空白机器上执行 README 自身的命令，并通过准确的 H2 文本查找各章节。如果重命名标题，也要在 ci.yml 中同步修改。
- 每周工作流会针对真实 API 执行 README 中的每个围栏代码块，每日工作流则运行规模更小的实时检查。每个代码片段都必须完整且可单独粘贴运行，其展示的输出必须真实。
- README 中的字段表位于 `<!-- fields:start -->` 与 `<!-- fields:end -->` 之间，每天由 `.github/scripts/field_tables.py` 根据 https://docs.brightdata.com/scrapers-full.json 重新生成。不要手动编辑。
- README 顶部的“上次验证”徽章行会由每日运行自动重写。不要手动编辑。
- 标题是为搜索而编写的，例如“Python URL 转 Markdown”“如何将 URL 转换为 Markdown……”。添加章节时，请保持此模式。
- 保持精简。除状态轮询外，不要添加重试、去重、调度、数据库、异步示例或并发功能。
