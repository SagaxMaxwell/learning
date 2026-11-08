---
date created: 2026-11-06
date updated: 2026-11-06
---

# Selenium

Selenium 适合传统浏览器自动化、已有企业测试项目和需要 WebDriver 生态的场景。新项目没有历史包袱时，优先评估 [[playwright|Playwright]]。

## 安装与启动

```bash
uv add selenium pytest
```

Selenium 4 通常会自动管理浏览器驱动：

```python
from selenium import webdriver
from selenium.webdriver.chrome.options import Options


options = Options()
options.add_argument("--headless=new")

driver = webdriver.Chrome(options=options)
try:
    driver.get("https://example.com")
    assert driver.title
finally:
    driver.quit()
```

`close()` 只关闭当前标签页，`quit()` 关闭整个 WebDriver 会话。测试结束优先使用 `quit()`。

## 定位元素

优先使用稳定、可读的定位方式：

```python
from selenium.webdriver.common.by import By


driver.find_element(By.ID, "username")
driver.find_element(By.NAME, "username")
driver.find_element(By.CSS_SELECTOR, "input[name='username']")
driver.find_element(By.XPATH, "//button[@type='submit']")
driver.find_element(By.LINK_TEXT, "登录")
```

优先级通常是 `ID`、稳定属性、CSS，再考虑 XPath。不要依赖自动生成的 class 或层级很深的 XPath。

## 等待

### 显式等待

等待特定条件，范围小、结果稳定，通常优先使用：

```python
from selenium.webdriver.support import expected_conditions as EC
from selenium.webdriver.support.ui import WebDriverWait


button = WebDriverWait(driver, 10).until(
    EC.element_to_be_clickable((By.ID, "submit"))
)
button.click()
```

常用条件包括 `presence_of_element_located`、`visibility_of_element_located`、`element_to_be_clickable`、`url_contains` 和 `frame_to_be_available_and_switch_to_it`。

### 隐式等待

为查找元素设置全局等待：

```python
driver.implicitly_wait(5)
```

隐式等待简单，但会影响所有查找。不要和大量显式等待混用，否则总等待时间不容易判断。不要用固定 `sleep()` 代替状态等待。

## 页面切换

### iframe

```python
driver.switch_to.frame(driver.find_element(By.CSS_SELECTOR, "iframe.payment"))
driver.find_element(By.ID, "card-number").send_keys("test")
driver.switch_to.default_content()
```

### 窗口或标签页

```python
original = driver.current_window_handle
driver.find_element(By.LINK_TEXT, "新页面").click()

WebDriverWait(driver, 10).until(lambda value: len(value.window_handles) == 2)
new_window = next(handle for handle in driver.window_handles if handle != original)
driver.switch_to.window(new_window)
driver.close()
driver.switch_to.window(original)
```

### 原生弹窗

```python
alert = WebDriverWait(driver, 10).until(EC.alert_is_present())
assert alert.text == "确认提交？"
alert.accept()
```

## 常见控件

下拉框使用 `Select`，文件上传直接向 `input[type=file]` 写入路径：

```python
from selenium.webdriver.support.ui import Select


Select(driver.find_element(By.ID, "region")).select_by_visible_text("华东")
driver.find_element(By.CSS_SELECTOR, "input[type=file]").send_keys("/tmp/report.txt")
```

需要滚动或处理页面没有直接暴露的属性时才使用 JavaScript：

```python
driver.execute_script("arguments[0].scrollIntoView()", button)
```

## pytest 组织

用 fixture 统一创建和关闭浏览器，失败时保留截图：

```python
import pytest
from selenium import webdriver


@pytest.fixture
def driver():
    browser = webdriver.Chrome()
    yield browser
    browser.quit()


def test_title(driver):
    driver.get("https://example.com")
    assert driver.title == "Example Domain"
```

真实项目可以在 `pytest_runtest_makereport` 中统一保存失败截图、日志和页面源码，不要在每个用例里重复处理。

## Page Object

页面定位和业务操作集中在页面对象中，用例只描述场景：

```python
class LoginPage:
    def __init__(self, driver):
        self.driver = driver

    def login(self, username: str, password: str) -> None:
        self.driver.find_element(By.ID, "username").send_keys(username)
        self.driver.find_element(By.ID, "password").send_keys(password)
        self.driver.find_element(By.ID, "submit").click()
```

页面对象负责操作和等待，测试用例负责输入、断言和业务意图。

## 常见异常

| 异常 | 常见原因 | 优先排查 |
| --- | --- | --- |
| `NoSuchElementException` | 元素不存在或定位错误 | 定位器、页面状态 |
| `TimeoutException` | 等待条件未满足 | 页面是否加载、条件是否正确 |
| `ElementNotInteractableException` | 元素存在但不可操作 | 可见性、遮罩、滚动 |
| `StaleElementReferenceException` | DOM 更新后旧引用失效 | 重新定位元素 |
| `ElementClickInterceptedException` | 其他元素挡住点击 | 等待可点击、关闭遮罩 |
| `NoSuchFrameException` | iframe 不存在或未加载 | 等待并切换正确 frame |

不要用重试掩盖真实定位或同步问题。先确认页面状态，再决定是否重新定位或等待。

## 调试与安全

- 本地调试可使用有头模式、截图和浏览器日志。
- 失败时保存截图、页面源码和关键请求信息。
- 代理、User-Agent 和浏览器参数只用于合法测试环境，不用于绕过验证码、访问控制或网站限制。
- 抓取任务遵守网站条款、`robots.txt` 和访问频率限制。

## 参考

- Selenium 官方文档：<https://www.selenium.dev/documentation/>
- Selenium Python API：<https://selenium-python.readthedocs.io/>
