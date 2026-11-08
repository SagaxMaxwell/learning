---
date created: 2024-07-29 11:55
date updated: 2026-11-06T18:26
---

# Flask

Flask 是基于 WSGI 的轻量 Web 框架，内置路由、上下文、Jinja 模板和测试客户端。ORM、表单和认证按需接入扩展，不必为了“完整”一次安装全部工具。

需要 API 契约和类型校验时可对比 [[fastapi|FastAPI]]；API、UI 与业务层的职责见 [[api|Web API 与应用]]。

## 安装与启动

在 uv 项目中安装并启动开发服务：

```bash
uv add flask
uv run flask --app app run --debug
```

默认地址为 `127.0.0.1:5000`。指定监听地址与端口：

```bash
uv run flask --app app run --host 0.0.0.0 --port 8080
```

`0.0.0.0` 会对外监听，调试模式只能用于可信开发环境；生产环境使用 WSGI Server，并关闭调试。

```python
from flask import Flask

app = Flask(__name__)


@app.get("/health")
def health():
    return {"status": "ok"}
```

## 路由、请求与响应

`@app.get()`、`@app.post()` 或 `@app.route(..., methods=[...])` 将 URL 映射到视图函数。业务逻辑放服务层，视图负责协议转换。

| 数据来源 | 读取方式 |
| --- | --- |
| 路径参数 | `/users/<int:user_id>` 对应函数参数 |
| 查询参数 | `request.args.get("keyword")` |
| 表单 | `request.form.get("name")` |
| JSON | `request.get_json()` |
| Header | `request.headers.get("User-Agent")` |
| 文件 | `request.files["file"]` |

返回字典时会转换为 JSON；`(数据, 状态码)` 可指定状态码。不要把未经转义的用户输入拼进 HTML。

```python
from flask import request


@app.post("/api/items")
def create_item():
    payload = request.get_json()
    if not isinstance(payload, dict):
        return {"error": "JSON object required"}, 400
    name = payload.get("name")
    if not isinstance(name, str) or not name.strip():
        return {"error": "name is required"}, 400
    return {"id": 1, "name": name.strip()}, 201
```

本例只演示请求与响应，没有实现保存。常用状态码：创建用 `201`，无效输入用 `400`，未登录用 `401`，无权限用 `403`，不存在用 `404`，内部故障用 `500`。


### URL 生成

`url_for()` 根据 endpoint 和参数生成 URL，不是注册路由。重定向使用 `redirect(url_for("目标函数名"))`。

```python
from flask import Flask, url_for

app = Flask(__name__)

@app.route('/hello')
def hello():
    return 'Hello, World!'

@app.route('/user/<username>')
def profile(username):
    return f'Profile page for {username}'

with app.test_request_context():
    print(url_for('hello'))  # 输出: /hello
    print(url_for('profile', username='john'))  # 输出: /user/john
```

## 模板、静态文件与局部更新

Jinja 模板放在 `templates`，通过 `render_template("index.html", value=...)` 渲染；静态资源放在 `static`，模板里用 `url_for("static", filename="style.css")` 引用。

模板用于把数据填入 HTML；AJAX 用于不重新加载整页就请求数据、更新部分内容。现代浏览器直接使用 `fetch()`，不需要额外引入旧式回调扩展。

### fetch 与 JSON

- 现代浏览器直接使用内置 `fetch()`，后端返回字典或 `jsonify()`。
- 发送 JSON 时要设置 `Content-Type: application/json`，后端用 `request.get_json()` 或 `request.json` 解析。

```python
from flask import Flask, jsonify, render_template, request

app = Flask(__name__)

@app.get("/")
def index():
    return render_template("index.html")

@app.post("/api/greet")
def greet():
    payload = request.get_json(silent=True) or {}
    name = str(payload.get("name") or "world").strip()
    return jsonify(message=f"Hi {name}")
```

```html
<button id="greet">Greet</button>
<output id="result"></output>

<script>
const result = document.querySelector("#result");

document.querySelector("#greet").addEventListener("click", async () => {
  const response = await fetch("/api/greet", {
    method: "POST",
    headers: {"Content-Type": "application/json"},
    body: JSON.stringify({name: "Flask"}),
  });
  const data = await response.json();
  result.value = data.message;
});
</script>
```


## 应用组织与上下文

### 蓝图

Blueprint 把同一业务的路由组织在一起，再注册到应用；它不是独立服务器，也不是完整的业务分层。

```python
from flask import Blueprint, Flask

books = Blueprint("books", __name__)


@books.get("/books")
def list_books():
    return {"items": []}


def create_app():
    application = Flask(__name__)
    application.register_blueprint(books)
    return application
```

### 上下文本地对象

| 对象 | 用途 | 生命周期 |
| --- | --- | --- |
| `request` | 当前 HTTP 请求 | 请求上下文 |
| `session` | 当前用户的会话数据 | 当前请求读写，跨请求保存由会话机制实现 |
| `current_app` | 当前应用配置与实例代理 | 应用上下文 |
| `g` | 当前上下文临时复用数据，如连接 | 应用上下文，通常随一次请求结束 |

这些是上下文本地代理，不是所有请求共享的全局变量。请求处理时自动建立上下文；脚本或初始化操作可用 `with app.app_context():`。脱离上下文访问会报错。

跨请求状态应保存到会话、缓存或数据库，不放进 `g`。应用上下文也不是贯穿整个进程生命周期的一份全局容器。

### 请求处理

1. WSGI Server 将请求交给 Flask。
2. Flask 建立上下文、匹配路由并执行请求前处理。
3. 调用视图，将返回值转换成响应。
4. 执行响应后处理，结束时运行清理回调并弹出上下文。

`after_request` 用于处理响应；`teardown_request` 用于清理，异常场景也应能安全执行。


## Session 与安全

### 默认存储机制

- Flask 默认使用签名 Cookie 保存 Session 数据。
- 处理流程：序列化 Session 数据，使用 `SECRET_KEY` 签名防篡改，再编码后写入客户端 Cookie。
- 优点是无状态、部署简单，不需要服务端会话存储。
- 缺点是容量受 Cookie 限制，数据默认可被客户端读取，不能存敏感信息；`SECRET_KEY` 泄露会带来安全风险。
- 非永久 Session 通常随浏览器会话结束；永久 Session 的生命周期由 `PERMANENT_SESSION_LIFETIME` 控制。

### 读写会话

- `from flask import session`

```python
# 在请求之间保存数据可以使用 session
import os

from flask import Flask, session
app = Flask(__name__)
app.config["SECRET_KEY"] = os.environ["SECRET_KEY"]

@app.route("/use_session")
def use_session():
    if "songs" not in session:
        session["songs"] = {"title": "Tapestry", "singer": "Bruno Major"}
    return session.get("songs")

@app.route("/delete_session")
def delete_session():
    session.pop("songs", None)
    return "removed song from session"
```

### 第三方登录

OAuth 2.0 / OpenID Connect 可评估 Authlib。注册客户端、配置回调地址与 scopes，校验 `state`，按流程启用 PKCE，并验证身份提供方返回的数据。认证、权限与 CSRF 保护不是同一件事。

Flask-Login 偏登录会话管理，Flask-JWT-Extended 偏 Token；选择取决于认证方式，不能只靠一个扩展名称就认为权限已完整实现。


## 数据库与扩展

持久化分层、驱动、连接池与主键选择统一见 [[learning/data/orm|数据持久化与 ORM]]，SQL 与索引原理见 [[learning/data/sql|SQL]]。Flask 可以使用 `sqlite3`、数据库驱动或 Flask-SQLAlchemy，使用 Redis、MongoDB 时再接入对应客户端。

Flask-SQLAlchemy 的基本流程是配置连接 → 定义模型 → 初始化扩展 → 在事务中读写 → 提交或回滚。实际项目用迁移维护表结构。

### Flask-SQLAlchemy 索引

```python
class User(db.Model):
    id = db.Column(db.Integer, primary_key=True)
    username = db.Column(db.String(80), unique=True, nullable=False, index=True)
    email = db.Column(db.String(120), unique=True, nullable=False)

    __table_args__ = (
        db.Index("ix_user_email", "email"),
        db.UniqueConstraint("username", "email", name="uq_user_username_email"),
    )
```

> 在字段中设置 `index=True` 自动添加索引
> 在 `__table_args__` 可手动设置其他索引，参数为元组

### 按需选择扩展

| 需求 | 工具 | 边界 |
| --- | --- | --- |
| 关系数据库 ORM | Flask-SQLAlchemy | 不代替 SQL、事务和迁移知识 |
| 表单与 CSRF | Flask-WTF | 模板必须包含 CSRF 字段 |
| 管理后台 | Flask-Admin、Flask-AppBuilder | 对外提供前先配置认证和权限 |
| 邮件 | Flask-Mail 或标准库 | 凭据用外部配置，不通过公开接口任意发送 |
| API schema 与文档 | APIFlask、Pydantic 等 | 先明确 schema 与 HTTP 层的边界 |

Jinja2 是模板引擎，`session` 是 Flask 提供的会话对象，不把它们列成需要额外安装的 Flask 扩展。


## Flask-WTF 表单

```bash
uv add flask-wtf email-validator
```

- 适用于表单处理，数据校验并提供 CSRF 验证的功能扩展
- 能够使表单免受 CSRF（跨站请求伪造） 的攻击
- 可提供 `FileField` 处理文件上传，在表单提交后，自动从 `flask.request.files` 中抽取数据

```python
import os

from flask import Flask, render_template, flash, redirect, url_for
from flask_wtf import FlaskForm
from wtforms import StringField, SubmitField
from wtforms.validators import DataRequired, Email

app = Flask(__name__)
app.config["SECRET_KEY"] = os.environ["SECRET_KEY"]

# 定义表单类
class MyForm(FlaskForm):
    name = StringField("Name", validators=[DataRequired()])
    email = StringField("Email", validators=[DataRequired(), Email()])
    submit = SubmitField("Submit")

# 定义路由和视图函数
@app.route("/", methods=["GET", "POST"])
def index():
    form = MyForm()
    if form.validate_on_submit():
        # 处理表单数据
        flash("收到来自 {} 的邮件地址 {}".format(form.name.data, form.email.data))
        return redirect(url_for("index"))
    return render_template("index.html", form=form)

if __name__ == "__main__":
    app.run(debug=True)
```

```html
<!-- index.html -->
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <title>Flask-WTF 表单示例</title>
</head>
<body>
    <h1>请填写以下表单</h1>
    <form method="post">
        {{ form.hidden_tag() }}
        <p>
            {{ form.name.label }}<br>
            {{ form.name(size=20) }}
        </p>
        <p>
            {{ form.email.label }}<br>
            {{ form.email(size=20) }}
        </p>
        <p>{{ form.submit() }}</p>
    </form>
    {% with messages = get_flashed_messages() %}
      {% if messages %}
        <ul>
        {% for message in messages %}
          <li>{{ message }}</li>
        {% endfor %}
        </ul>
      {% endif %}
    {% endwith %}
</body>
</html>
```

> `FileField.data` 属于 `Werkzeug` 对象

## Flask-Mail 邮件

```bash
uv add flask-mail
```

```python
import os

from flask import Flask
from flask_mail import Mail, Message

app = Flask(__name__)

# 配置邮件服务器信息
app.config["MAIL_SERVER"] = "smtp.example.com"
app.config["MAIL_PORT"] = 587
app.config["MAIL_USE_TLS"] = True
app.config["MAIL_USERNAME"] = os.environ["MAIL_USERNAME"]
app.config["MAIL_PASSWORD"] = os.environ["MAIL_PASSWORD"]

mail = Mail(app)

@app.route("/mail")
def email():
    msg = Message("Hello Message", sender="admin@test.com", recipients=["to@test.com"])
    mail.send(msg)
    return "Email sent!"

if __name__ == "__main__":
    app.run(debug=True)
```

## 错误处理与测试

已知业务错误转换为稳定的错误响应，预期的 HTTP 错误保留原有状态码。不要用无差别捕获 `Exception` 把所有 `404` 都变成 `500`。

```python
@app.errorhandler(404)
def page_not_found(error):
    return {"error": "not found"}, 404
```

自定义业务错误可继承 `werkzeug.exceptions.HTTPException`，设置状态码后注册处理器。错误页也可用 `render_template()` 渲染，但不要向客户端暴露异常栈。

Flask 自带测试客户端，不需要真实监听端口：

```python
def test_health():
    with app.test_client() as client:
        response = client.get("/health")

    assert response.status_code == 200
    assert response.json == {"status": "ok"}
```

## 调试与部署

```bash
uv run flask --app app routes
uv run flask --app app shell
uv run python -c "import secrets; print(secrets.token_hex(32))"
```

阅读 Flask 源码时可克隆官方仓库并可编辑安装：

```bash
git clone https://github.com/pallets/flask
cd flask
uv venv
uv pip install -e ".[dev]"
uv run python -m pytest
```

Nginx、Caddy 和 WSGI/ASGI 的部署边界见 [[networking|网络服务]]，交付流程见 [[learning/engineering/deployment|交付与部署]]。

## 参考

- [Flask 官方文档](https://flask.palletsprojects.com/)
- [Flask JavaScript、fetch 和 JSON](https://flask.palletsprojects.com/en/stable/patterns/javascript/)
- [Flask-WTF 文档](https://flask-wtf.readthedocs.io/)
