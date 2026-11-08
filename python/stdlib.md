---
date created: 2024-07-01 13:13
date updated: 2026-11-06T18:30
---

# Python 常用库

按任务查找标准库与少量常用扩展，不为每个短小工具单独建文件。并发工具的运行示例集中在 [[concurrency.ipynb|并发与任务队列]]。

## 文件与目录

路径操作优先用 `pathlib`；复制和目录处理用 `shutil`；环境变量、底层系统接口用 `os`。

```python
from pathlib import Path
import shutil

source = Path("source.txt")
destination = Path("target.txt")
shutil.copy2(source, destination)
destination.unlink()
```

删除目录可用 `shutil.rmtree()`，调用前确认路径和范围，不能直接信任外部输入。

### 遍历目录

```python
from pathlib import Path

for filename in Path("notes").rglob("*"):
    print(filename)
```

`iterdir()` 只列当前层，`rglob()` 递归匹配；需要分别处理子目录与文件时用 `os.walk()`，需要目录项属性时可用 `os.scandir()`。

### 大文件读取

逐行处理文本或按块读取二进制，避免一次加载整个文件。`seek()` 可调整读取位置；二进制模式按字节定位更明确。

```python
chunk_size = 2 * 1024 * 1024

with open("large_file.bin", "rb") as file:
    for chunk in iter(lambda: file.read(chunk_size), b""):
        process(chunk)
```

`process()` 代表实际的数据处理函数。

## 邮件

`email.message.EmailMessage` 组织内容，`smtplib` 连接 SMTP 服务器发送。凭据从环境读取，配置超时并启用 TLS。

```python
import os
import smtplib
import ssl
from email.message import EmailMessage

message = EmailMessage()
message["From"] = os.environ["SMTP_USERNAME"]
message["To"] = "receiver@example.com"
message["Subject"] = "Test Email"
message.set_content("This is a test email sent by Python.")

with smtplib.SMTP("smtp.example.com", 587, timeout=10) as server:
    server.starttls(context=ssl.create_default_context())
    server.login(os.environ["SMTP_USERNAME"], os.environ["SMTP_PASSWORD"])
    server.send_message(message)
```

服务器若要求连接时即加密，使用 `SMTP_SSL` 和对应端口，不再调用 `starttls()`。

## 随机数与抽样

`random` 用于模拟、打乱与抽样；密码、验证码和 Token 使用 `secrets`。

```python
import random

mu = 0
sigma = 1

print(random.random())              # [0, 1) float
print(random.uniform(1, 3))          # [1, 3] float
print(random.randrange(1, 3))        # [1, 3) int
print(random.randint(1, 3))          # [1, 3] int
print(random.normalvariate(mu, sigma))  # 正态分布的浮点数
```

- `random.shuffle(x)` 函数可以实现对序列 `x` 的原地随机乱序
- `random.choice(seq)` 函数可以从非空序列中取出一个随机元素
- `random.choices(population, weights=None, *, cum_weights=None, k=1)` 函数可以从总体中随机抽取（有放回抽样）出容量为 `k` 的样本并返回样本的列表，可以通过参数指定个体的权重，如果没有指定权重，个体被选中的概率均等
- `random.sample(population, k)` 函数可以从总体中随机抽取（无放回抽样）出容量为 `k` 的样本并返回样本的列表

> `random` 模块提供的函数除了生成均匀分布的随机数外，还可以生成其他分布的随机数，例如 `random.gauss(mu, sigma)` 函数可以生成高斯分布（正态分布）的随机数

## 模块与导入

模块通常是包含函数、类和变量的 `.py` 文件。普通包使用 `__init__.py`，可在其中初始化包或定义 `__all__`；命名空间包可以没有这个文件。

| 环境变量 | 用途 |
| --- | --- |
| `PYTHONPATH` | 向模块搜索路径补充目录；不是系统 `PATH` |
| `PYTHONHOME` | 修改 Python 标准库位置前缀，虚拟环境中通常不设置 |
| `PYTHONSTARTUP` | 交互式解释器启动时执行的脚本，不用于普通脚本启动 |
| `PYTHONCASEOK` | 部分平台允许导入时忽略大小写；不建议依赖 |

项目依赖和运行环境优先交给 uv，不靠全局修改这些变量修复导入问题。

## 序列化与类型转换

- `pickle.dump()` / `load()` 通过二进制流保存和恢复 Python 对象；`dumps()` / `loads()` 处理 `bytes`。不是所有对象都支持序列化，且绝不能加载不可信的 pickle 数据。
- JSON、Pydantic 与 msgspec 见 [[validation|校验与序列化]]。
- Python 3 的整数统一使用 `int`；`int("1010", 2)` 返回 `10`。非法字符串抛出 `ValueError`，不支持的类型可能抛出 `TypeError`。

## 子进程

一般命令用 `subprocess.run()`；需要流式通信、持续运行或细控生命周期时用 `Popen()`。优先传参数列表，不把用户输入拼进 Shell 命令。

```python
import subprocess
import sys

result = subprocess.run(
    [sys.executable, "--version"],
    check=True,
    capture_output=True,
    text=True,
    timeout=10,
)
print(result.stdout.strip())
```

`check=True` 使非零退出码抛出 `CalledProcessError`；超时抛出 `TimeoutExpired`。`pty` 用于需要伪终端的场景，不是普通子进程的默认选择。

## 正则表达式

### 常用方法

| 方法 | 用途 |
| --- | --- |
| `re.match()` | 从字符串开头匹配 |
| `re.search()` | 搜索第一个匹配位置 |
| `re.fullmatch()` | 检查整个字符串是否匹配 |
| `re.findall()` | 返回所有匹配结果 |
| `re.sub()` | 替换匹配内容 |
| `re.split()` | 按匹配位置分割 |

反复使用同一规则时，可以用 `re.compile()` 保存正则对象。Python 默认 Unicode 模式下，`\d`、`\w` 不只匹配 ASCII 字符；限制英文和数字时明确写范围或使用 `re.ASCII`。

### 语法速查

| 字符类      | 描述                                 |
| -------- | ---------------------------------- |
| `.`      | 匹配除换行符之外的任何 `1` 个字符                |
| `[abc]`  | 匹配方括号内的任意 `1` 个字符                  |
| `[^abc]` | 匹配不在方括号内的任意 `1` 个字符                |
| `\d`     | 匹配一个 Unicode 十进制数字字符；ASCII 模式为 `[0-9]` |
| `\D`     | 匹配 `\d` 之外的字符；ASCII 模式等价于 `[^0-9]` |
| `\w`     | 匹配 Unicode 字母数字或下划线；ASCII 模式为 `[A-Za-z0-9_]` |
| `\W`     | 匹配 `\w` 之外的字符 |
| `\s`     | 匹配任何空白字符，包括空格、制表符、换行符等等            |
| `\S`     | 匹配任何非空白字符                          |

| 量词       | 描述                     |
| -------- | ---------------------- |
| `*`      | 匹配前面的子表达式零次或多次         |
| `+`      | 匹配前面的子表达式一次或多次         |
| `?`      | 匹配前面的子表达式零次或一次         |
| `{n}`    | `n` 是一个非负整数，恰好匹配 `n` 次 |
| `{n,}`   | 至少匹配 `n` 次             |
| `{n,m}` | 至少匹配 `n` 次且最多匹配 `m` 次  |

| 定位符  | 描述                 |
| ---- | ------------------ |
| `^`  | 匹配输入字符串的开始位置       |
| `$`  | 匹配输入字符串的结束位置       |
| `\b` | 匹配 `\w` 与 `\W` 之间或字符串首尾的单词边界 |
| `\B` | 非单词边界匹配            |

| 分组和引用     | 描述                             |
| --------- | ------------------------------ |
| `(exp)`   | 匹配 `exp` 并捕获文本到自动命名的组里         |
| `(?:exp)` | 匹配 `exp` 但不捕获匹配的文本             |
| `\1`      | 引用第一个捕获组；`\n` 本身表示换行符 |

| 选择符  | 描述                   |
| ---- | -------------------- |
| `\|` | 匹配两个或多个分支选择的任意 `1` 个 |

| 反义符      | 描述         |
| -------- | ---------- |
| `[^…]` | 表示不在括号中的字符 |

| 断言         | 描述                        |
| ---------- | ------------------------- |
| `(?=exp)`  | 正向先行断言，匹配 `exp` 前面的位置     |
| `(?!exp)` | 负向先行断言，匹配后面跟的不是 `exp` 的位置 |

### 贪婪与非贪婪

| 贪婪量词     | 非贪婪量词     | 描述                                      |
| -------- | --------- | --------------------------------------- |
| `*`      | `*?`      | 匹配前面的元素零次或多次，非贪婪版本会尽可能少地匹配              |
| `+`      | `+?`      | 匹配前面的元素一次或多次，非贪婪版本会尽可能少地匹配              |
| `?`      | `??`      | 匹配前面的元素零次或一次，非贪婪版本会尽可能少地匹配              |
| `{m,n}` | `{m,n}?` | 匹配前面的元素至少 `m` 次但不超过 `n` 次，非贪婪版本会尽可能少地匹配 |

固定文本替换用 `str.replace()`；按规则替换用 `re.sub()`，不必所有替换都引入正则。

## 日期与时区

存储时间通常使用带时区的 UTC，展示时再转换到用户时区。

```python
from datetime import UTC, datetime
from zoneinfo import ZoneInfo

now = datetime.now(UTC)
shanghai = now.astimezone(ZoneInfo("Asia/Shanghai"))
text = shanghai.strftime("%Y-%m-%d %H:%M:%S %Z")
parsed = datetime.strptime("2026-07-10", "%Y-%m-%d").replace(tzinfo=UTC)
```

## 命名元组与不可变数据

`namedtuple` 为元组字段提供名字，也可以通过子类增加方法。`_asdict()` 转成字典，`_replace()` 返回替换了字段的新对象。

```python
from collections import namedtuple

Card = namedtuple("Card", ("suit", "rank"))
card = Card("红桃", 13)
assert card.suit == "红桃"
assert card._replace(suit="方块") == Card("方块", 13)
assert card._asdict() == {"suit": "红桃", "rank": 13}
```

需要类型标注时可选 `typing.NamedTuple`；需要数据对象时可用 `dataclass(frozen=True)`。冻结只禁止字段重新赋值，不会递归冻结字段引用的列表等对象。

```python
from dataclasses import dataclass


@dataclass(frozen=True)
class Point:
    horizontal: int
    vertical: int
```

集合去重且不可变可用 `frozenset`；有实际性能需求时再评估第三方 `immutables.Map`。

## 迭代工具

`itertools` 适合组合、惰性读取和序列拼接；`more-itertools` 补充分块、滑动窗口等配方。

```python
from itertools import chain, islice, pairwise

values = list(chain([1, 2], [3, 4]))
first_three = list(islice(range(100), 3))
pairs = list(pairwise(["a", "b", "c"]))
```

## 资源管理与文件锁

简单资源用 `with`；多个动态资源用 `ExitStack`，异步资源用 `AsyncExitStack`。`atexit` 只用于兜底清理，不依赖它做关键业务提交。

```python
from contextlib import ExitStack

paths = ["a.txt", "b.txt"]
with ExitStack() as stack:
    files = [stack.enter_context(open(filename, encoding="utf-8")) for filename in paths]
    data = [file.read() for file in files]
```

跨进程修改同一文件可用 `filelock`。直接使用 `fcntl` 或 `msvcrt` 时，需要处理平台差异；网络文件系统的锁行为另行确认。

## 压缩、标识与校验

| 任务 | 工具 | 注意点 |
| --- | --- | --- |
| 压缩与解压 | `zipfile`、`tarfile` | 解压不可信文件时限制目标路径、大小和数量 |
| 紧凑位数组 | `bitarray` | 适合 bitmap、布隆过滤器等 |
| 随机 UUID | `uuid.uuid4()` | 不依赖中心服务；不是顺序 ID |
| 时间有序 UUID | Python 3.14 的 `uuid.uuid7()`，或旧版 Python 的 `uuid6` 包 | 按运行版本选择；不代表泄漏后仍安全 |
| CRC 校验 | `binascii.crc32()` | 检测意外变化，不是密码学哈希 |

Linux 命令速查：

```bash
tar -czf archive.tar.gz notes
tar -xzf archive.tar.gz
zip -r notes.zip notes
unzip notes.zip
```

## 相关内容

- [[concurrency.ipynb|并发与任务队列]]：线程、进程、协程与队列的运行示例。
- [[learning/web/networking|网络服务]]：socket、HTTPX 与 SSH 客户端。
