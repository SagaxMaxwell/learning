---
date created: 2024-03-19 17:22
date updated: 2026-11-06T18:30
---

# 交付与部署

DevOps 是开发、测试、运维协作和自动化交付的方式，不是某个工具。开发模型与项目文档见 [[architecture|软件过程与架构]]。

## CI/CD

最小流水线：

1. 安装依赖。
2. 静态检查：Ruff、类型检查。
3. 单元测试。
4. 构建产物。
5. 集成测试或 E2E。
6. 部署到测试环境。
7. 人工或自动门禁后发布。

失败要尽早发生。格式、lint 和单元测试放前面，浏览器 E2E 放后面。

## Docker 与 Compose

容器把应用与依赖打包，用于一致的开发环境、测试环境和部署。镜像可移植不代表容器内状态能自动持久化；数据需要卷或外部存储。

| 工具 | 作用 | 适合场景 |
| --- | --- | --- |
| Dockerfile | 定义镜像构建过程 | 固定运行时与依赖 |
| Docker Compose | 描述多个服务的配置和网络 | 本地应用、数据库、缓存联调 |
| 容器编排平台 | 调度、扩缩容与服务恢复 | 多节点部署；简单项目不必引入 |

构建产物与配置分离，不在镜像中写入密钥。为服务设置健康检查、资源限制、日志和明确版本。

## Linux 部署流程

1. 安装依赖并准备配置。
2. 配置防火墙、服务用户和目录权限。
3. 启动应用及 Web 服务器。
4. 用健康检查和业务请求验证部署。
5. 保留日志、监控和回滚方式。

Nginx 的反向代理和 Caddy 配置见 [[learning/web/networking#Nginx|网络服务]]。Python Web 应用可由合适的 WSGI/ASGI Server 承载，再由 Nginx 做反向代理。

## 测试门禁

关键业务的端到端测试放在较后的流水线阶段；单元测试和静态检查尽早运行。测试范围、替身和用例设计统一见 [[learning/testing/methods|测试方法]]。

浏览器测试独立准备数据，使用稳定定位，失败保留截图、trace 和日志。工具使用见 [[learning/testing/selenium|Selenium]] 与 [[learning/testing/playwright|Playwright]]。

## 参考

- [GitHub Actions](https://docs.github.com/actions)
- [Docker 文档](https://docs.docker.com/)
