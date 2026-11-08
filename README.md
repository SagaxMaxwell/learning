---
date created: 2026-07-18T11:34
date updated: 2026-11-06T18:26
---

# Learning

按主题组织的知识笔记、可运行实验和练习。

- `.md`：适合阅读、检索和长期维护的知识
- `.ipynb`：需要运行并观察结果的实验

同一主题的笔记与实验放在同一目录；没有运行需求时，不创建 Notebook。

```base
filters:
  and:
    - 'file.inFolder("learning")'
    - '["md", "ipynb"].contains(file.ext)'
    - 'file.name != "README"'
    - '!file.path.contains("/.")'
    - '!file.path.contains("/backups/")'
    - 'file.size > 0'
    - '!file.inFolder("learning/assets")'
    - '!file.inFolder("learning/src")'
    - '!file.inFolder("learning/tests")'
formulas:
  outline: |
    [
      ["python/basics.md","Python 基础"],
      ["python/basics-lab.ipynb","Python 基础"],
      ["python/functions.md","Python 函数与对象"],
      ["python/stdlib.md","Python 常用库"],
      ["python/validation.md","校验与序列化"],
      ["python/validation-lab.ipynb","Pydantic 数据校验"],
      ["python/performance.md","Python 并发、内存与性能"],
      ["python/concurrency.ipynb","并发与任务队列"],
      ["python/scheduling.ipynb","任务调度"],
      ["systems/os.md","操作系统"],
      ["systems/linux.md","Linux"],
      ["systems/network.md","计算机网络"],
      ["systems/setup-linux.md","Linux 配置"],
      ["systems/setup-windows.md","Windows 配置"],
      ["algorithms/data-structures.md","数据结构与 C/C++"],
      ["algorithms/patterns.ipynb","算法模式"],
      ["algorithms/exercises.md","Python 编程练习"],
      ["algorithms/practice.ipynb","Python 练习与测评"],
      ["algorithms/leetcode.ipynb","LeetCode 题库"],
      ["algorithms/problems.ipynb","OD 与解题练习"],
      ["data/sql.md","SQL 基础与查询"],
      ["data/mysql.md","MySQL"],
      ["data/orm.md","数据持久化与 ORM"],
      ["data/redis.md","Redis"],
      ["data/analysis.ipynb","数据分析与机器学习"],
      ["data/quant-models.md","量化模型"],
      ["web/api.md","Web API 与应用"],
      ["web/fastapi.md","FastAPI"],
      ["web/flask.md","Flask"],
      ["web/apps.ipynb","Python Web 应用"],
      ["web/networking.md","网络服务"],
      ["web/scraping.md","爬虫与网页自动化"],
      ["testing/methods.md","测试方法与用例设计"],
      ["testing/cases.md","测试用例设计参考"],
      ["testing/mock.ipynb","Mock 实践"],
      ["testing/playwright.md","Playwright"],
      ["testing/selenium.md","Selenium"],
      ["testing/playwright-lab.ipynb","Playwright 浏览器自动化"],
      ["testing/uds.md","汽车 UDS"],
      ["engineering/architecture.md","软件过程与架构"],
      ["engineering/design-patterns.md","设计模式"],
      ["engineering/design-decisions.md","设计取舍"],
      ["engineering/system-design.md","系统设计"],
      ["engineering/deployment.md","交付与部署"],
      ["engineering/markdown.md","Markdown 与 Obsidian 笔记规范"]
    ]
  position: 'formula.outline.reduce(if(file.path == "learning/" + value[0], index, acc), 999)'
  note: 'file.asLink(if(formula.position < formula.outline.length, formula.outline[formula.position][1], file.name))'
  topic: 'if(file.inFolder("learning/python"), "Python", if(file.inFolder("learning/algorithms"), "算法与练习", if(file.inFolder("learning/data"), "数据与数据库", if(file.inFolder("learning/testing"), "测试", if(file.inFolder("learning/web"), "Web", if(file.inFolder("learning/systems"), "系统与网络", if(file.inFolder("learning/engineering"), "软件工程", "其他")))))))'
properties:
  formula.note:
    displayName: 笔记
  file.ext:
    displayName: 格式
views:
  - type: kanban
    name: 笔记
    groupBy:
      property: formula.topic
      direction: ASC
    groupOrder:
      - Python
      - 系统与网络
      - 算法与练习
      - 数据与数据库
      - Web
      - 测试
      - 软件工程
      - 其他
    order:
      - formula.note
      - file.ext
    sort:
      - property: formula.position
        direction: ASC
      - property: file.path
        direction: ASC
    columnWidth: 260
```
