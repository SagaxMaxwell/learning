---
date created: 2026-10-25T14:34
date updated: 2026-11-06T18:31
---

# 爬虫与网页自动化

## 工具选择

| 工具 | 适合场景 |
| --- | --- |
| `httpx` + 解析库 | 静态页面、公开 API、简单抓取 |
| Scrapy | 大规模抓取、调度、去重和数据管道 |
| [[learning/testing/playwright\|Playwright]] | 现代浏览器自动化、动态页面和 E2E |
| [[learning/testing/selenium\|Selenium]] | 传统浏览器自动化或已有企业项目 |

先判断数据是否来自接口。能直接请求接口就不要启动浏览器；只有 JS 渲染、复杂登录态或需要真实用户行为时才使用浏览器自动化。

## Scrapy 核心对象

- Spider：定义入口 URL、解析逻辑和后续请求。
- Selector：用 CSS/XPath 取数据。
- Item 或 dataclass：结构化数据。
- Pipeline：清洗、去重和保存。
- Middleware：代理、UA、重试、限速等公共逻辑。

```python
import scrapy


class QuotesSpider(scrapy.Spider):
    name = "quotes"
    start_urls = ["https://quotes.toscrape.com/"]

    def parse(self, response):
        for quote in response.css(".quote"):
            yield {
                "text": quote.css(".text::text").get(),
                "author": quote.css(".author::text").get(),
            }

        next_page = response.css("li.next a::attr(href)").get()
        if next_page:
            yield response.follow(next_page, self.parse)
```

浏览器工具的安装、等待、定位、窗口切换和 pytest 写法分别见 [[learning/testing/playwright|Playwright]] 与 [[learning/testing/selenium|Selenium]]，这里不重复维护示例。

## 合规与稳定性

- 遵守网站条款、`robots.txt` 和访问频率限制。
- 对登录态、验证码和个人数据保持谨慎，不绕过访问控制。
- 抓取任务需要重试、超时、限速、断点续跑和日志。
- 数据保存前做 schema 校验，避免脏数据进入后续流程。

## 参考

- Scrapy 官方文档：<https://docs.scrapy.org/en/latest/>
- Playwright Python 文档：<https://playwright.dev/python/docs/intro>
- Selenium 官方文档：<https://www.selenium.dev/documentation/>
