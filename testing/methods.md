---
date created: 2026-10-25T14:34
date updated: 2026-10-29T22:53
---

# 测试方法与用例设计

> 用途：用一个小型“产线配置字服务”串起常见测试概念和 pytest 写法。

## 结论或简介

测试方法不是互斥标签：一个测试既可以是黑盒测试，也可以是集成测试和回归测试。实际工作中先明确测试目标，再选择测试层级、测试数据和验证方式。

示例服务负责校验配置字、调用设备写入并读取校验。它足够小，但能覆盖单元、白盒、黑盒、灰盒、集成、回归、边界值、等价类、异常注入和性能测试。

## 示例对象

### 被测代码

```python
from dataclasses import dataclass
from typing import Protocol


class Device(Protocol):
    def write_config(self, value: bytes) -> None: ...
    def read_config(self) -> bytes: ...


@dataclass
class ConfigService:
    device: Device

    def apply(self, value: bytes) -> None:
        if len(value) != 2:
            raise ValueError("config word must contain exactly 2 bytes")
        if value == b"\x00\x00":
            raise ValueError("config word cannot be zero")

        self.device.write_config(value)
        actual = self.device.read_config()
        if actual != value:
            raise RuntimeError("config word verification failed")
```

### 测试替身

```python
class FakeDevice:
    def __init__(self, value: bytes = b"\x00\x01") -> None:
        self.value = value
        self.write_count = 0

    def write_config(self, value: bytes) -> None:
        self.write_count += 1
        self.value = value

    def read_config(self) -> bytes:
        return self.value
```

## Stub、Mock 与 Fake

测试替身（Test Double）是总称。Stub 提供预设响应，Fake 实现简化行为，Mock 用于验证交互。`unittest.mock.Mock` 是工具，既可以当 Stub，也可以记录和验证调用。

实机测试关注设备是否真正生效；这些替身主要用于验证框架逻辑，不能代替真实 CAN/UDS、网络或设备测试。

### 如何选择

| 需求 | 优先选择 | 配置字服务示例 |
| --- | --- | --- |
| 单个用例只需要固定返回值或异常 | 用 `Mock` 充当 Stub | 固定回读配置，或让写入抛出超时 |
| 多个测试反复使用同一种响应，具名实现更清楚 | 自定义 Stub 类 | `MismatchDeviceStub` 始终返回错误配置 |
| 需要写入与回读关联，或维护简单状态 | Fake 类 | 上面的 `FakeDevice` 保存配置，读取时返回当前值 |
| 调用参数、次数或顺序本身就是约定 | Mock 加调用断言 | 校验写入参数、先写后读、失败后不再读取 |
| 依赖是简单纯函数或值对象 | 直接用真实对象 | 配置字长度校验不需要替换 `bytes` 或 `len` |

先从少量 Mock 配置开始；同一套配置反复出现且具名类更易读时，再提取 Stub。需要状态行为时写 Fake，不要用大量 `side_effect` 拼出另一套设备实现。

### Mock 常用接口

逐个运行示例见 [[learning/testing/mock.ipynb|Mock 实践]]，包含返回值、三种 `side_effect`、调用断言、接口约束、异步和 `patch`。

### 设置响应，再验证调用

以下示例复用前面的 `Device` 和 `ConfigService`。

```python
from unittest.mock import call, create_autospec


def test_apply_calls_device_in_order() -> None:
    device = create_autospec(Device, instance=True, spec_set=True)
    device.read_config.return_value = b"\x01\x02"

    ConfigService(device).apply(b"\x01\x02")

    device.write_config.assert_called_once_with(b"\x01\x02")
    device.read_config.assert_called_once_with()
    assert device.mock_calls == [
        call.write_config(b"\x01\x02"),
        call.read_config(),
    ]
```

`return_value` 只是预设响应。真正的传参发生在 `apply()` 内部的 `write_config(value)`；Mock 记录调用，最后的断言查记录。这里精确比较 `mock_calls`，还会拒绝额外调用。

仅当“恰好一次”或“特定顺序”是要求时才断言它们。若只关心最终状态，优先像 Fake 示例那样检查 `device.value`，避免把测试绑定到内部实现。

### 用 Mock 充当 Stub

```python
import pytest
from unittest.mock import Mock


def test_apply_stops_after_write_timeout() -> None:
    device = Mock(spec=Device)
    device.write_config.side_effect = TimeoutError("device timeout")

    with pytest.raises(TimeoutError, match="device timeout"):
        ConfigService(device).apply(b"\x01\x02")

    device.read_config.assert_not_called()
```

异常配置起 Stub 的作用，末尾断言验证 Mock 的调用记录。同一个对象可以同时承担两种角色。

### 自定义 Stub 类

```python
class MismatchDeviceStub:
    def write_config(self, value: bytes) -> None:
        pass

    def read_config(self) -> bytes:
        return b"\xff\xff"


def test_apply_rejects_stub_response() -> None:
    with pytest.raises(RuntimeError, match="verification failed"):
        ConfigService(MismatchDeviceStub()).apply(b"\x01\x02")
```

这个 Stub 不保存写入值，只提供固定回读结果；前面的 Fake 会保存并读回值。分类取决于行为，不取决于类名，也不是用了 `Mock()` 就只能验证调用。

## 测试范围

关注测试覆盖多大的系统边界，不等同于框架自身的代码分层。

### 单元测试

聚焦一个小单元的行为，按需用 Stub、Mock 或 Fake 隔离外部依赖；简单纯函数和值对象可以直接使用。

```python
def test_apply_writes_valid_config() -> None:
    device = FakeDevice()

    ConfigService(device).apply(b"\x01\x02")

    assert device.value == b"\x01\x02"
    assert device.write_count == 1
```

### 集成测试

验证多个真实组件之间能否协同工作。例如服务通过真实 UDS 适配器连接 ECU，检查写入、回读和诊断拒绝响应的解析。只把服务接到 `FakeDevice`，仍是前面的单元测试，不算验证了真实适配器。

下面需要项目提供 `real_device` fixture，负责创建真实适配器、连接设备及清理资源，不能直接脱离环境运行。`integration` 标记需在 pytest 配置中注册。

```python
import pytest


@pytest.mark.integration
def test_apply_on_real_device(real_device: Device) -> None:
    ConfigService(real_device).apply(b"\x10\x01")
    assert real_device.read_config() == b"\x10\x01"
```

### 端到端测试

从定义好的业务入口走到最终结果，验证完整链路，而不只是某几个组件的接口。

电检系统的例子是“提交任务 → 调度器准备配置并启动 pytest → 用例操作设备 → 保存结果和报告”。不仅检查正常任务成功，还检查关键失败场景：

- 某个测试项失败后，其他独立测试仍按规则继续，报告准确记录失败。
- 设备断连后，任务按超时或恢复策略结束，不永久卡住、不误报成功。
- 配置文件无效时，明确拒绝任务、记录原因，不操作设备。

完整链路依赖具体任务入口、设备和结果存储，这里不虚构可运行接口。端到端侧重关键业务结果，参数边界优先在单元层充分验证，不必每层重复穷举。

### 范围如何选择

| 测试    | 电检中的具体例子                                                   | 运行特点                   |
| ----- | ---------------------------------------------------------- | ---------------------- |
| 单元测试  | 用 Mock 或 Fake 隔离设备，验证写配置字的逻辑：非法长度不写入、回读不一致时报错              | 不连接设备，快且稳定             |
| 集成测试  | 通过真实 UDS 适配器连接 ECU，验证配置字能否写入、回读，以及拒绝响应能否正确解析               | 验证组件协作，需要设备或协议仿真环境     |
| 端到端测试 | 从提交电检任务开始，经过调度、pytest 执行、设备操作，直到结果保存；验证正常流程和关键失败场景的最终状态、报告 | 验证完整业务链路，环境依赖多、成本较高    |

推荐比例不是硬性规则，但通常应让大多数测试停留在快速的单元层，少量关键流程使用集成和端到端测试。

## 测试设计

黑盒、白盒既描述设计视角，也常作为测试设计技术的大类；灰盒强调利用部分内部信息。下面按常见归属组织，并非互斥分类。测试范围是另一维度，同一用例可以是黑盒视角的单元测试。

### 黑盒测试

只根据输入和输出设计用例，不关注内部实现。

```python
import pytest


@pytest.mark.parametrize(
    ("value", "expected_error"),
    [
        (b"", "exactly 2 bytes"),
        (b"\x01", "exactly 2 bytes"),
        (b"\x01\x02\x03", "exactly 2 bytes"),
        (b"\x00\x00", "cannot be zero"),
    ],
)
def test_apply_rejects_invalid_config(value: bytes, expected_error: str) -> None:
    with pytest.raises(ValueError, match=expected_error):
        ConfigService(FakeDevice()).apply(value)
```

#### 等价类

把预期行为相同的输入划为一类，每类选代表值，减少无效重复。

| 等价类       | 示例                | 预期   |
| --------- | ----------------- | ---- |
| 合法长度、非零   | `b"\x01\x02"`     | 成功   |
| 空输入       | `b""`             | 长度错误 |
| 长度不足      | `b"\x01"`         | 长度错误 |
| 长度超出      | `b"\x01\x02\x03"` | 长度错误 |
| 合法长度但业务无效 | `b"\x00\x00"`     | 拒绝   |

#### 边界值

重点测试规则边界及其相邻值。本例长度要求是 2：

```python
@pytest.mark.parametrize("value", [b"", b"\x01", b"\x01\x02\x03"])
def test_length_boundaries(value: bytes) -> None:
    with pytest.raises(ValueError):
        ConfigService(FakeDevice()).apply(value)
```

通常至少覆盖：最小值、最小值前后、最大值、最大值前后，以及空值、零值和重复值。

#### 判定表

多个条件共同决定结果时，把条件组合与预期动作列成表，避免漏掉业务规则。本例只看输入校验，不包含后续设备故障。

| 长度为 2 | 配置非全零 | 预期动作      |
| ----- | ----- | --------- |
| 否     | 不影响结果 | 拒绝，不访问设备  |
| 是     | 否     | 拒绝，不访问设备  |
| 是     | 是     | 进入写入和回读流程 |

前面的非法输入参数化用例验证前两行；正常写入用例验证最后一行，不必为表格再复制一套测试。

#### 状态迁移

当操作结果取决于当前状态或操作顺序时，验证合法迁移、非法迁移和迁移后的行为。

假设某 ECU 的协议约定如下；这是扩展示例，当前 `ConfigService` 没有实现会话和解锁管理，不代表所有 ECU 的通用规则。

| 当前状态     | 操作          | 预期         |
| -------- | ----------- | ---------- |
| 默认会话     | 进入扩展会话      | 变为扩展会话、未解锁 |
| 扩展会话、未解锁 | 写配置字        | 拒绝，配置不变    |
| 扩展会话、未解锁 | 使用正确密钥解锁    | 变为已解锁      |
| 扩展会话、已解锁 | 写合法配置字      | 写入成功       |
| 扩展会话、已解锁 | ECU 重启并恢复连接 | 回到默认会话、未解锁 |

除了单步操作，还要验证序列，例如“解锁 → 重启 → 再写入”应被拒绝。预期状态、响应和恢复时间以具体 ECU 规范为准。

### 白盒测试

根据代码分支和路径设计用例，目标是覆盖重要逻辑，而不只是覆盖正常输入。

#### 语句覆盖

让可执行语句至少执行一次。例如只测合法配置，虽然经过写入和读取，但没有执行校验失败的 `raise`。

语句执行过不代表每个条件结果都验证过，也不代表断言有效。

#### 分支覆盖

让每个判断的不同出口都执行。本例三个 `if` 都需要真假两种结果：长度是否错误、配置是否全零、回读是否不一致。

分支覆盖不等于条件覆盖；若判断是 `A and B`，仅覆盖整体真假，不一定覆盖每个子条件的取值及其影响。

> 分支全覆盖包括语句全覆盖。

#### 路径覆盖

关注从入口到出口经过的完整控制流。本例包括长度错误退出、全零退出、回读不一致退出和正常完成，并额外验证写入、读取的异常出口。

复杂程序中的分支组合和循环可能产生大量甚至无限条路径，通常优先选择关键可行路径，而不是承诺穷举。

> 所有可行路径全覆盖，意味着所有可达分支和语句也都覆盖；只测关键路径则不保证。

#### 配置字服务的测试示例

这是前面设计方法的应用示例，不是新的设计方法：先分析分支、路径和异常出口，再列出场景与预期，最后写成 pytest 用例，并通过异常注入制造设备故障。

本例覆盖四条业务路径，以及写入、读取两个依赖异常出口。测试复用前面的 `Device` 和 `ConfigService`。

先给三个判断命名：

- A：`len(value) != 2`，长度错误？
- B：`value == b"\x00\x00"`，配置全零？
- C：`actual != value`，回读不一致？

沿代码执行顺序展开路径：

1. A 为 True，立即抛错退出；为 False，才判断 B。
2. B 为 True，立即抛错退出；为 False，才写入并读取。
3. 写入或读取抛异常，立即退出；两者都完成，才判断 C。
4. C 为 True，校验失败；为 False，正常完成。

据此得到下面的用例，而不是直接穷举 A、B、C 的所有真假组合。表中“未执行”不等于 False。

| 路径 | A | B | C | 预期与额外检查 |
| --- | --- | --- | --- | --- |
| 长度错误 | True | 未执行 | 未执行 | `ValueError`，不访问设备 |
| 全零配置 | False | True | 未执行 | `ValueError`，不访问设备 |
| 回读不一致 | False | False | True | `RuntimeError`，已先写后读 |
| 回读一致 | False | False | False | 正常返回，已先写后读 |
| 写入失败 | False | False | 未执行 | 原异常向上传递，不继续读取、不重试 |
| 读取失败 | False | False | 未执行 | 原异常向上传递，不重复写入、不重试 |

例如 A=True 后已经退出，不会继续判断 B；而全零配置本身长度就是 2，A、B 也不可能同时为 True。最后两行虽然判断结果相同，但异常发生位置不同，所以分别测试。

```python
import pytest
from unittest.mock import call, create_autospec


@pytest.mark.parametrize(
    ("value", "message"),
    [
        (b"", "exactly 2 bytes"),
        (b"\x01", "exactly 2 bytes"),
        (b"\x01\x02\x03", "exactly 2 bytes"),
        (b"\x00\x00", "cannot be zero"),
    ],
)
def test_invalid_input_never_accesses_device(value: bytes, message: str) -> None:
    device = create_autospec(Device, instance=True, spec_set=True)

    with pytest.raises(ValueError, match=message):
        ConfigService(device).apply(value)

    assert device.mock_calls == []


def test_mismatch_fails_after_write_and_read() -> None:
    device = create_autospec(Device, instance=True, spec_set=True)
    device.read_config.return_value = b"\xff\xff"

    with pytest.raises(RuntimeError, match="verification failed"):
        ConfigService(device).apply(b"\x01\x02")

    assert device.mock_calls == [
        call.write_config(b"\x01\x02"),
        call.read_config(),
    ]


def test_matching_readback_completes() -> None:
    device = create_autospec(Device, instance=True, spec_set=True)
    device.read_config.return_value = b"\x01\x02"

    assert ConfigService(device).apply(b"\x01\x02") is None

    assert device.mock_calls == [
        call.write_config(b"\x01\x02"),
        call.read_config(),
    ]


@pytest.mark.parametrize("operation", ["write_config", "read_config"])
def test_device_error_propagates_without_retry(operation: str) -> None:
    device = create_autospec(Device, instance=True, spec_set=True)
    error = TimeoutError("device timeout")
    getattr(device, operation).side_effect = error

    with pytest.raises(TimeoutError, match="device timeout") as caught:
        ConfigService(device).apply(b"\x01\x02")

    assert caught.value is error
    expected_calls = [call.write_config(b"\x01\x02")]
    if operation == "read_config":
        expected_calls.append(call.read_config())
    assert device.mock_calls == expected_calls
```

精确比较调用记录，是为了验证这里的“先写后读、失败即退出、不自动重试”行为，不代表真实设备已写入成功。

这组用例覆盖当前实现三个 `if` 的真假分支和上述异常出口；不代表穷尽所有输入或设备故障。可用 `pytest-cov` 的 `--cov-branch` 检查分支覆盖率，但 100% 分支覆盖也不等于任意程序的全部路径都覆盖。

### 灰盒测试

利用部分内部设计信息，从外部验证行为。例如知道设备写入会先更新缓存、再持久化，就补充“写入成功 → 重启 → 回读”的场景，验证数据是否真正保存。

仅依据公开协议的两字节长度规则设计用例，也可以是黑盒测试；是否属于灰盒，取决于实际利用了哪些内部信息，而不是是否接触设备。

灰盒测试适合测试开发：利用协议、日志、状态机和数据格式等内部知识，设计更有针对性的黑盒用例。

## 回归测试

修改后重新验证原有功能，检查是否引入退化；也包括把已修复缺陷固化成用例，防止复发。

```python
def test_regression_zero_config_is_rejected() -> None:
    """曾经出现全零配置被错误写入设备的问题。"""
    with pytest.raises(ValueError):
        ConfigService(FakeDevice()).apply(b"\x00\x00")
```

回归测试不等于某种测试层级：它可以是单元测试、集成测试，也可以是端到端测试。

## 性能测试

性能测试应先定义指标和数据规模；不要用小样本的单次耗时得出结论。

```python
import time


def test_apply_latency() -> None:
    device = FakeDevice()
    service = ConfigService(device)
    start = time.perf_counter()

    for _ in range(1_000):
        service.apply(b"\x01\x02")

    elapsed = time.perf_counter() - start
    assert elapsed < 0.5
```

这个例子只是简单基准，不等于正式性能测试。正式测试还应记录吞吐量、P95/P99 延迟、资源占用、并发数和错误率，并避免把真实硬件的网络波动混入纯代码基准。

## 跨视角的测试技术

以下技术可以结合黑盒、白盒或灰盒使用，不必强行归入某一类。

### 异常注入

异常注入（故障注入）是人为制造超时、断连、错误响应等故障，验证系统如何处理。它可以用于单元、集成和端到端测试，也可以配合黑盒、白盒、灰盒设计，因此放在这里，不归属于其中某一种。

#### 与异常场景设计的关系

判定表、状态迁移、分支分析都能找出异常场景，但不保证找全；异常注入负责让选定的故障实际发生。两者配合使用，不是互相替代。

| 场景 | 怎样设计出来 | 怎样制造 |
| --- | --- | --- |
| 长度错误、配置全零 | 等价类、边界值、判定表 | 传入非法值，属于异常输入测试，通常不称为故障注入 |
| 未解锁时写入 | 判定表、状态迁移 | 准备未解锁状态后发请求，属于前置状态测试 |
| 写入超时 | 需求、分支分析、故障分析 | Mock 抛出超时，或仿真器延迟响应 |
| 写成功但回读不一致 | 需求、判定表、分支分析 | Stub 返回其他配置，或由设备仿真器制造不一致 |
| ECU 已写成功，但响应丢失 | 分析通信故障与设备状态 | 仿真器先完成写入，再丢弃响应 |

同样是超时，设备可能尚未写入，也可能已写入但响应丢失。仅让 Mock 抛出 `TimeoutError`，不能证明真实设备处于哪种状态；需要分别准备场景，并按业务约定验证状态确认、恢复或重试行为。

#### 注入位置与验证范围

| 范围 | 常见注入手段 | 重点验证 |
| --- | --- | --- |
| 单元 | Mock 的 `side_effect`、Stub | 异常传播、调用顺序、失败后是否停止 |
| 集成 | 协议仿真器、网络代理或可控通信故障 | 真实组件的超时、响应解析和恢复行为 |
| 端到端 | 在选定业务步骤注入代表性故障 | 后续步骤、最终设备状态、任务结果和报告 |

#### 写入超时示例

下面复用前面的 `ConfigService`。用一个始终写入超时的 Stub，确认异常向上传递且没有继续读取。

```python
class FailingDevice:
    def write_config(self, value: bytes) -> None:
        raise TimeoutError("device write timeout")

    def read_config(self) -> bytes:
        raise AssertionError("read should not be called")


def test_device_timeout_is_not_silently_ignored() -> None:
    with pytest.raises(TimeoutError, match="timeout"):
        ConfigService(FailingDevice()).apply(b"\x01\x02")
```

常见注入点：超时、连接断开、错误响应、返回空数据、写入后读回不一致、磁盘满、消息重复和进程中断。

按风险选择注入场景，记录故障位置、触发时机、预期行为和恢复方式。关键是识别重要异常、能够制造并验证结果，不是承诺穷尽所有故障；真实环境注入还需安排设备状态与连接的清理恢复。

### 组合测试

关注多个因素同时变化时是否仍正确；路径覆盖见 [[methods#路径覆盖]]。

本例可组合：

```text
输入长度：正确 / 错误
配置值：普通 / 全零
设备行为：成功 / 超时 / 读回不一致
```

不要机械地穷举所有组合。先排除无效或不可达组合，例如非法长度已经被拒绝，设备故障不会再触发。优先覆盖高风险组合，例如“合法输入 + 写入成功 + 回读不一致”。

因素较多时可考虑两两组合（Pairwise），覆盖每对因素取值的组合以减少用例；它不能保证发现三因素以上的交互缺陷，关键风险仍需补充定向用例。

## pytest 常用能力

### Fixture

fixture 负责准备和清理测试依赖，例如客户端、临时目录和数据库连接。

```python
import pytest


@pytest.fixture
def user_payload() -> dict[str, str]:
    return {"name": "Alice", "email": "alice@example.com"}


@pytest.fixture
def resource():
    item = open_resource()
    yield item
    item.close()
```

`yield` 前准备资源，`yield` 后清理资源。需要共享资源时再考虑 `scope="module"` 或 `scope="session"`，有状态的设备通常使用更小的范围。

### 参数化

同一测试逻辑需要验证多组输入时，优先使用参数化：

```python
@pytest.mark.parametrize(
    ("raw", "expected"),
    [(" A ", "a"), ("Bob", "bob")],
)
def test_normalize(raw: str, expected: str) -> None:
    assert raw.strip().lower() == expected
```

### Hook 与插件

fixture 解决“用例需要什么资源”；hook 解决“在 pytest 哪个执行阶段接入逻辑”。常见 hook 包括 `pytest_addoption`、`pytest_configure`、`pytest_collection_modifyitems` 和 `pytest_sessionfinish`。

常用插件：

| 插件 | 作用 |
| --- | --- |
| `pytest-cov` | 覆盖率 |
| `pytest-xdist` | 并行执行 |
| `pytest-timeout` | 防止用例卡死 |
| `pytest-asyncio` | 异步测试 |
| `pytest-benchmark` | 性能基准 |

## 测试框架的代码组织

这是框架的职责划分，不是测试分类。基于 pytest 的电检领域框架可以按以下职责组织，不必为每项单独建一层或目录：

- 用例与断言：描述测试步骤、输入和预期结果。
- 业务服务：封装安全解锁、写配置字、模式切换等流程。
- 设备适配：实现 CAN、UDS、SSH 通信，隔离设备差异。
- 配置与资源：管理车型参数、fixture、连接和清理。
- 执行与结果：复用 pytest 的执行能力，输出日志和报告。

API 是被测接口类型，不是独立测试层级；API 用例也可以是集成或端到端测试。

## 常见边界

- 单元测试不要连接真实设备；真实设备测试应单独标记并控制环境。
- 覆盖率高不代表质量高，仍要检查断言是否验证了真正的业务结果。
- 测试失败时先区分产品缺陷、测试缺陷、环境故障和数据问题。
- 性能测试要固定数据规模、环境和指标，否则结果不可比较。
- 异常测试不仅要断言“抛错”，还要确认没有继续执行危险操作。

## 设计练习

[[learning/testing/cases|测试用例设计参考]]：按消息撤回、配置字、转账、电检四道题展开，每题选用适合的设计方法，并说明集成或端到端的验证范围。
