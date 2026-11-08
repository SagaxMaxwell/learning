---
date created: 2026-10-25T14:34
date updated: 2026-11-06T18:19
---

# Web API 与应用

## API 层职责

API 层负责协议和边界，不负责堆业务细节：

- 解析 HTTP 请求、鉴权和限流。
- 用 Pydantic 或 dataclass 校验输入输出。
- 调用内部 service、scheduler 或 domain 对象。
- 统一异常结构、日志和追踪 ID。

服务层负责业务规则。这样 CLI、定时任务、Gradio UI 和 FastAPI 都能复用同一套核心逻辑。

## FastAPI

完整实现见 [[fastapi|FastAPI 教程]]。重点是路由层只处理 HTTP 边界，输入输出模型、异常结构、依赖注入和测试都应有明确约定。

前后端模型保持一致，常见做法有：

- 后端导出 OpenAPI，让前端生成客户端类型。
- 把 JSON Schema 作为契约，由前后端共同校验。
- 在单仓库中共享 schema 定义或生成产物。

## Gradio 与 Streamlit

模型 Demo 可评估 Gradio，数据看板可评估 Streamlit。选型比较、事件处理、缓存与运行示例集中在 [[apps.ipynb|Web 应用实验]]；这里只强调 UI 和业务逻辑分开。

## Reflex

Reflex 用 Python 写全栈 UI，适合希望用 Python 管理前端状态的应用。它与 FastAPI、Gradio 的边界不同：FastAPI 偏 API，Gradio 偏模型或工具 UI，Reflex 偏完整 Web 应用。

## 服务边界

- API schema 不直接暴露数据库 ORM 对象。
- response model 与前端类型保持同源。
- 长任务交给队列或任务调度器，不阻塞请求线程。
- 统一错误码、错误消息和日志字段。
- 复杂业务放在 service 或 domain 层，避免路由函数变成“大杂烩”。

## 参考

- FastAPI 官方文档：<https://fastapi.tiangolo.com/>
- Gradio 官方文档：<https://www.gradio.app/docs/>
- Streamlit 官方文档：<https://docs.streamlit.io/>
- Reflex 官方文档：<https://reflex.dev/docs/>
