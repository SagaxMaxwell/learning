---
date created: 2024-07-01 13:13
date updated: 2026-11-06T15:36
---

# Python 函数与对象

## Python 的参数传递机制是什么？

- Python 的参数传递通常称为"对象引用传递"或"共享传参"。
- 函数形参拿到的是对象引用的副本，不能让外部变量重新绑定到另一个对象，但可以修改可变对象本身。
- 不可变对象（如 `int`、`str`、`tuple`）在函数内重新赋值只会让局部变量绑定到新对象。
- 可变对象（如 `list`、`dict`、`set`）在函数内执行原地修改会影响调用方持有的对象。

## 如何在 Python 中重载构造函数或方法？

- Python 不支持像 Java/C++ 那样按参数类型或参数个数进行方法重载，后定义的同名方法会覆盖先定义的方法。
- 通常通过默认参数、可变参数、关键字参数、`functools.singledispatch` 或手动分派来实现类似重载的效果。
- `__init__()` 负责初始化对象；真正创建对象的是 `__new__()`。

## `__init__` 方法是什么？

- Python 中的方法或构造函数
- 当创建类的新对象/实例时，会自动调用此方法来初始化对象成员
- 所有类都有 `__init__` 方法

## 装饰器

装饰器接收函数并返回一个可调用对象，用来增加计时、日志、鉴权等行为。带参数时，最外层先接收配置，再返回真正的装饰器。

```python
from collections.abc import Callable
from functools import wraps
from time import perf_counter
from typing import ParamSpec, TypeVar

Parameters = ParamSpec("Parameters")
Result = TypeVar("Result")


def timer(label: str) -> Callable[[Callable[Parameters, Result]], Callable[Parameters, Result]]:
    def decorate(function: Callable[Parameters, Result]) -> Callable[Parameters, Result]:
        @wraps(function)
        def wrapper(*args: Parameters.args, **kwargs: Parameters.kwargs) -> Result:
            started = perf_counter()
            try:
                return function(*args, **kwargs)
            finally:
                print(f"{label}: {perf_counter() - started:.3f}s")

        return wrapper

    return decorate


@timer("读取配置")
def read_config() -> bytes:
    return b"\x01\x02"
```

`wraps` 保留函数名称、文档和 `__wrapped__`，便于调试及签名检查；`ParamSpec` 和 `TypeVar` 保留参数与返回值的类型关系。它不会自动处理异步函数，异步包装器还需要 `async def` 和 `await`。

## Python 中的迭代器是什么？

- 迭代器是实现了迭代协议的对象：`__iter__()` 返回迭代器自身，`__next__()` 返回下一个元素。
- 没有更多元素时，`__next__()` 会抛出 `StopIteration`。
- 迭代器本身不一定保存所有元素，可以惰性地产生数据。

## Python 中的生成器是什么？

- 生成器是一种更简单的迭代器写法，生成器对象同样实现了 `__iter__()` 和 `__next__()`。
- 如果一个函数包含 `yield`，调用该函数时不会立即执行函数体，而是返回一个生成器对象。
- 每次迭代执行到 `yield` 时暂停并返回值，下次迭代从暂停位置继续执行。
- 适合处理大文件、流式数据和无限序列，优点是惰性计算、节省内存。

## Python 中的 `lambda` 函数是什么？

- 匿名函数，使用 `lambda` 定义
- 普通函数，使用 `def` 定义

## Python 中的 `self` 是什么？

- `self` 是一个类的实例或对象
- 实例方法的第一个成员
- `__init__()` 方法中的自变量是指新创建的对象，而在其他实例方法中，是指调用其方法的对象

## 在 Python 中复制对象的方法有哪些？

- `copy.copy()`
- `copy.deepcopy()`
- `item.copy()`

## 浅拷贝和深拷贝

- 深拷贝问题
	- 对象如果直接或间接的引用了自身，会导致无休止的递归拷贝
	- 深拷贝可能对原本设计为多个对象共享的数据也进行拷贝
- 浅拷贝通常只复制对象本身
- 而深拷贝不仅会复制对象，还会递归的复制对象所关联的对象

## 如何实现单例模式？

- 使用装饰器
- 使用元类

```python
def singleton(cls):
    instances = {}

    def wrapper(*args, **kwargs):
        if cls not in instances:
            instances[cls] = cls(*args, **kwargs)
        return instances[cls]

    return wrapper


@singleton
class User:
    pass


jojo = User()
dio = User()
print(jojo is dio)  # True
```

```python
class SingletonMeta(type):
    _instances = {}

    def __call__(cls, *args, **kwargs):
        if cls not in cls._instances:
            cls._instances[cls] = super().__call__(*args, **kwargs)
        return cls._instances[cls]


class MyClass(metaclass=SingletonMeta):
    pass


m1 = MyClass()
m2 = MyClass()
print(m1 is m2)  # True
```

## `__init__()` 和 `__new__()` 方法有什么区别？

- Python 中调用构造器创建对象属于两阶段构造过程，首先执行 `__new__()` 方法获得保存对象所需的内存空间，再通过 `__init__()` 执行对内存空间数据的填充（对象属性的初始化）
- `__new__()` 方法的返回值是创建好的 Python 对象（的引用），而 `__init__()` 方法的第一个参数就是这个对象（的引用），所以在 `__init__()` 中可以完成对对象的初始化操作
- `__new__()` 是类方法，它的第一个参数是类，`__init__()` 是实例方法，它的第一个参数是对象

## Python 中的魔术方法？

| 魔术方法                                                                         | 作用        |
| ---------------------------------------------------------------------------- | --------- |
| `__new__()`、`__init__()`、`__del__()`                                         | 创建和销毁对象相关 |
| `__add__()`、`__sub__()`、`__mul__()`、`__div__()`、`__floordiv__()`、`__mod__()` | 算术运算符相关   |
| `__eq__()`、`__ne__()`、`__lt__()`、`__gt__()`、`__le__()`、`__ge__()`            | 关系运算符相关   |
| `__pos__()`、`__neg__()`、`__invert__()`                                       | 一元运算符相关   |
| `__lshift__()`、`__rshift__()`、`__and__()`、`__or__()`、`__xor__()`             | 位运算相关     |
| `__enter__()`、`__exit__()`                                                   | 上下文管理器协议  |
| `__iter__()`、`__next__()`、`__reversed__()`                                   | 迭代器协议     |
| `__int__()`、`__long__()`、`__float__()`、`__oct__()`、`__hex__()`               | 类型/进制转换相关 |
| `__str__()`、`__repr__()`、`__hash__()`、`__dir__()`                            | 对象表述相关    |
| `__len__()`、`__getitem__()`、`__setitem__()`、`__contains__()`、`__missing__()` | 序列相关      |
| `__copy__()`、`__deepcopy__()`                                                | 对象拷贝相关    |
| `__call__()`、`__setattr__()`、`__getattr__()`、`__delattr__()`                 | 其他魔术方法    |

## 函数参数 `*arg` 和 `**kwargs` 分别代表什么？

- Python 中，函数的参数分为位置参数、可变参数、关键字参数、默认参数。`*args` 代表可变参数，可以接收 `0` 个或任意多个参数，当不确定调用者会传入多少个位置参数时，就可以使用可变参数，它会将传入的参数打包成一个元组
- `**kwargs` 代表关键字参数，可以接收用 `参数名=参数值` 的方式传入的参数，传入的参数的会打包成一个字典。定义函数时如果同时使用 `*args` 和 `**kwargs`，那么函数可以接收任意参数

## 什么是鸭子类型（duck typing）？

- 鸭子类型是动态类型语言判断一个对象是不是某种类型时使用的方法，也叫做鸭子判定法
- 简单的说，鸭子类型是指判断一只鸟是不是鸭子，我们只关心它游泳像不像鸭子、叫起来像不像鸭子、走路像不像鸭子就足够了
- 换言之，如果对象的行为跟我们的预期是一致的（能够接受某些消息），我们就认定它是某种类型的对象
- 动态语言的鸭子类型使得设计模式的应用被大大简化

## Python 中的闭包？

- 闭包是支持一等函数的编程语言（Python、JavaScript 等）中词法绑定的技术
	- 一等函数可将函数作为返回值
	- 函数可作为参数
	- 函数可赋值给变量
	- 函数可作为元素存储在数据结构中
- 当捕捉闭包的时候，它的自由变量（在函数外部定义但在函数内部使用的变量）会在捕捉时被确定，这样即便脱离了捕捉时的上下文，它也能照常运行
- 可以将闭包理解为能够读取其他函数内部变量的函数
- 闭包会使得函数中创建的对象不会被垃圾回收，可能会导致很大的内存开销

## 猴子补丁？

- "猴子补丁"是动态类型语言的一个特性，代码运行时在不修改源代码的前提下改变代码中的方法、属性、函数等以达到热补丁（hot patch）的效果
- 实际开发中应该避免对猴子补丁的使用，以免造成代码行为不一致的问题

```python
import json

_original_dumps = json.dumps

def compact_dumps(obj):
    return _original_dumps(obj, separators=(",", ":"), ensure_ascii=False)


json.dumps = compact_dumps
print(json.dumps({"name": "张三"}))
```

## 查看结果

```python
class A:
    def who(self):
        print('A', end='')

class B(A):
    def who(self):
        super().who()
        print('B', end='')

class C(A):
    def who(self):
        super().who()
        print('C', end='')

class D(B, C):
    def who(self):
        super().who()
        print('D', end='')

item = D()
item.who()

# ACBD
```

- Python 中的 MRO（方法解析顺序）
- 在没有多重继承的情况下，向对象发出一个消息，如果对象没有对应的方法，那么向上（父类）搜索的顺序是非常清晰的。如果向上追溯到 `object` 类（所有类的父类）都没有找到对应的方法，那么将会引发 `AttributeError` 异常。但是有多重继承尤其是出现菱形继承（钻石继承）的时候，向上追溯到底应该找到那个方法就得确定 MRO
- Python 使用 C3 线性化算法确定 MRO。拿不准顺序时，直接查看类的 `mro()` 方法或 `__mro__` 属性。

> MRO 是解析方法顺序的机制

## 类型标注与接口

`Annotated` 在类型之外附加元数据；类型检查器仍看到基础类型，Pydantic 等框架可以读取元数据。

```python
from typing import Annotated, get_args, get_origin

UserId = Annotated[int, "positive database id"]

assert get_origin(UserId) is Annotated
assert get_args(UserId) == (int, "positive database id")
```

`Protocol` 描述对象需要具备的接口，不提供具体实现。`runtime_checkable` 允许在运行时检查属性或方法是否存在，但不会验证完整的参数签名和类型。

```python
from typing import Protocol, runtime_checkable


@runtime_checkable
class Closer(Protocol):
    def close(self) -> None: ...


def close_if_needed(value: object) -> None:
    if isinstance(value, Closer):
        value.close()
```

输入数据的实际校验见 [[validation|校验与序列化]]，不能把类型标注当成自动运行时校验。

## Java 的 static 与 Python 的区别

Java 的静态字段属于类，静态方法不需要实例，静态初始化块在类初始化时执行，不能声明返回值或参数。共享可变字段需要注意并发和实例之间的相互影响。

Python 的类属性、`@staticmethod`、`@classmethod` 也能表达部分相似意图，但不要直接套用 Java 的类加载和生命周期规则。
