---
tags:
  - 实战
  - Pytest
---


# Pytest

> 摘要：Pytest 是测试开发中最常用的 Python 测试框架。本文从测试开发视角整理其核心能力、项目级组织方式，以及 fixture、参数化、标记、报告集成等高频实践，帮助快速搭建可维护的自动化测试工程。

## 一、为什么测试开发需要 Pytest

Pytest 几乎已成为 Python 测试生态的事实标准。对测试开发而言，它的核心价值在于：

- **用例编写极简**：普通函数即可作为测试用例，`assert` 语句无需记忆复杂 API。
- **自动发现与执行**：按约定命名即可自动收集用例，命令行、配置文件、代码调用均可触发。
- **Fixture 机制**：用声明式的方式管理测试资源（环境配置、数据库连接、测试数据、SSH 会话等），替代传统 `setup/teardown`。
- **参数化与数据驱动**：`@pytest.mark.parametrize` 和 fixture 参数化让同一测试逻辑覆盖多组输入。
- **插件生态丰富**：Allure 报告、失败重跑、分布式执行、覆盖率等都可以通过插件低成本接入。
- **CI/CD 友好**：命令行驱动、退出码明确、报告格式标准，易于集成到 Jenkins/GitHub Actions 等流水线。

## 二、用例编写与发现规则

### 2.1 命名约定

Pytest 按以下规则自动发现测试用例：

| 层级 | 规则 |
|---|---|
| 测试文件 | `test_*.py` 或 `*_test.py` |
| 测试类 | 以 `Test` 开头，且不能包含 `__init__` 方法 |
| 测试函数 | 以 `test_` 开头的函数；类中以 `test_` 开头的方法 |

### 2.2 最小可运行示例

```python
# test_login.py

def test_login_success():
    resp = login("admin", "admin123")
    assert resp.status_code == 200
    assert resp.json()["token"]


class TestLogout:
    def test_logout_success(self):
        assert logout("valid_token") is True
```

### 2.3 断言方式

Pytest 使用原生 `assert` 进行断言，失败时会自动展开表达式：

```python
def test_dict_equal():
    expected = {"code": 0, "msg": "ok"}
    actual = {"code": 1, "msg": "ok"}
    assert actual == expected
```

对于复杂断言（如接口返回体、列表包含关系），可结合 `pytest-icdiff`、`deepdiff` 等插件增强可读性。

## 三、用例执行策略

### 3.1 命令行执行

```bash
# 运行当前目录下所有用例
pytest

# 运行指定模块/类/方法
pytest tests/test_login.py
pytest tests/test_login.py::TestLogin
pytest tests/test_login.py::TestLogin::test_login_success

# 按关键字匹配
pytest -k "login and not logout"

# 按标记匹配
pytest -m smoke

# 常用输出控制
pytest -v          # 详细模式
pytest -s          # 打印标准输出
pytest -q          # 安静模式
pytest -x          # 首次失败即停止
pytest --maxfail=3 # 失败 3 条后停止
pytest --lf        # 只运行上次失败的用例
pytest --ff        # 先运行上次失败的用例，再运行其余用例
```

### 3.2 通过代码调用

```python
import pytest

if __name__ == "__main__":
    pytest.main(["-v", "tests/test_login.py"])
```

### 3.3 失败重跑与分布式执行

| 插件 | 作用 | 示例 |
|---|---|---|
| `pytest-rerunfailures` | 失败用例自动重跑 | `pytest --reruns 2 --reruns-delay 1` |
| `pytest-repeat` | 重复执行同一用例 | `pytest --count=100 -x test_file.py` |
| `pytest-xdist` | 多进程并行执行 | `pytest -n auto` |

并行执行时需注意：

- 用例之间不能共享可变状态。
- `pytest-xdist` 默认无序执行，可用 `--dist=loadscope` 按类/模块分组。
- 资源密集型 UI 自动化需控制并行度，避免环境资源争抢。

## 四、Fixture：测试资源的组织与复用

### 4.1 Fixture 基础

Fixture 是 Pytest 最核心的能力之一，负责**准备资源 → 注入测试 → 清理资源**的完整生命周期。

```python
import pytest

@pytest.fixture(scope="function")
def db_connection():
    conn = create_db_connection()
    yield conn
    conn.close()


def test_query_user(db_connection):
    result = db_connection.query("SELECT * FROM users WHERE id = 1")
    assert result["name"] == "admin"
```

### 4.2 作用域

| scope | 说明 | 适用场景 |
|---|---|---|
| `function` | 每个测试函数执行一次（默认） | 临时文件、测试数据 |
| `class` | 每个测试类执行一次 | 类内共享的登录态 |
| `module` | 每个模块执行一次 | 模块级数据库连接 |
| `package` | 每个包执行一次 | 包级公共资源 |
| `session` | 整个测试会话执行一次 | 全局配置、环境初始化 |

### 4.3 数据清理方式

推荐使用 `yield` 拆分 setup 与 teardown：

```python
@pytest.fixture
def temp_file():
    f = open("temp.txt", "w")
    yield f
    f.close()
    os.remove("temp.txt")
```

对于需要确保清理一定执行的场景，使用 `request.addfinalizer`：

```python
@pytest.fixture
def resource(request):
    obj = create_resource()

    def cleanup():
        obj.destroy()

    request.addfinalizer(cleanup)
    return obj
```

上下文管理器也能与 fixture 自然结合：

```python
@pytest.fixture
def db_conn():
    with create_db_connection() as conn:
        yield conn
```

### 4.4 Fixture 返回工厂函数

当测试用例需要动态生成资源时，返回工厂函数比直接返回对象更灵活：

```python
@pytest.fixture
def make_user():
    users = []

    def _make_user(name=None):
        name = name or f"user-{random_data()}"
        user = create_user(name)
        users.append(user)
        return user

    yield _make_user

    for user in users:
        delete_user(user.id)


def test_user_profile(make_user):
    user = make_user("custom_name")
    assert get_profile(user.id)["name"] == "custom_name"
```

### 4.5 Fixture 参数化

Fixture 本身也可以参数化，供依赖它的用例多次执行：

```python
@pytest.fixture(params=["chrome", "firefox", "webkit"])
def browser(request):
    return request.param


def test_homepage(browser):
    assert browser in ["chrome", "firefox", "webkit"]
```

### 4.6 conftest.py 的组织

`conftest.py` 用于在同一目录及子目录中共享 fixture 和 hook：

```text
project/
├── conftest.py          # 全局 fixture：环境配置、全局登录态
├── tests/
│   ├── conftest.py      # 模块级公共 fixture
│   ├── compute/
│   │   ├── conftest.py  # 计算域 fixture：ecs、image、flavor
│   │   └── test_ecs.py
│   └── network/
│       ├── conftest.py  # 网络域 fixture：vpc、eip、sg
│       └── test_vpc.py
```

组织建议：

- 根目录 `conftest.py` 放**跨域通用 fixture**（如 `config`、`browser_context`、`global_cleanup`）。
- 子目录 `conftest.py` 放**该业务域专用 fixture**。
- 单一业务域内再复用的 fixture 可放到 `_xxx_fixtures.py`，由 `conftest.py` 导入。
- 不具备 fixture 生命周期语义的纯函数应放到 `helpers/` 或 `_xxx_helpers.py`。

## 五、参数化与数据驱动

### 5.1 测试函数参数化

```python
@pytest.mark.parametrize(
    "username, password, expected",
    [
        ("admin", "admin123", 200),
        ("admin", "wrong_pass", 401),
        ("", "admin123", 400),
    ],
)
def test_login(username, password, expected):
    resp = login(username, password)
    assert resp.status_code == expected
```

### 5.2 外部数据驱动

将测试数据抽到 YAML/JSON/CSV 中，实现数据与逻辑的分离：

```yaml
# data/login_cases.yaml
- username: admin
  password: admin123
  expected: 200
- username: admin
  password: wrong_pass
  expected: 401
```

```python
import yaml

@pytest.fixture
def login_cases():
    with open("data/login_cases.yaml") as f:
        return yaml.safe_load(f)


class TestLogin:
    @pytest.mark.parametrize("case", login_cases())
    def test_login(self, case):
        resp = login(case["username"], case["password"])
        assert resp.status_code == case["expected"]
```

### 5.3 indirect 参数化

当参数需要经过 fixture 初始化后再注入测试函数时使用：

```python
@pytest.fixture
def cce_cluster(request, tenant, ssh):
    params = request.param
    name = f"cce-{random_data()}"
    tenant.cce_create(name, **params)
    cluster = tenant.assert_paas(name)
    yield cluster
    tenant.assert_paas_deleted(name)


@pytest.mark.parametrize(
    "cce_cluster",
    [
        {"flavor": "cce.d6.large", "password": "admin1234"},
        {"flavor": "cce.d6.xlarge", "password": "root@pass"},
    ],
    indirect=True,
)
def test_cce_nodes(cce_cluster):
    assert cce_cluster["mfip"] is not None
```

## 六、标记（Mark）与用例过滤

### 6.1 自定义标记

```python
@pytest.mark.smoke
def test_login():
    ...


@pytest.mark.regression
class TestOrder:
    def test_create_order(self): ...
    def test_cancel_order(self): ...
```

执行：

```bash
pytest -m smoke
pytest -m "not slow"
pytest -m "smoke and not unstable"
```

### 6.2 内置标记

| 标记 | 作用 |
|---|---|
| `@pytest.mark.skip(reason="...")` | 无条件跳过 |
| `@pytest.mark.skipif(condition, reason="...")` | 满足条件时跳过 |
| `@pytest.mark.xfail(reason="...")` | 预期失败，失败记为 `XFAIL`，成功记为 `XPASS` |
| `@pytest.mark.usefixtures("fixture_name")` | 为类/函数指定 fixture，无需在参数列表中显式声明 |

自定义标记需要在 `pytest.ini` 中注册，避免告警：

```ini
[pytest]
markers =
    smoke: 冒烟测试
    regression: 回归测试
    slow: 执行较慢的用例
    unstable: 不稳定用例
```

## 七、测试报告集成

### 7.1 pytest-html

轻量报告，适合快速查看：

```bash
pip install pytest-html
pytest --html=report.html --self-contained-html
```

### 7.2 Allure

Allure 是项目级自动化测试更常用的报告方案，支持层级结构、附件、历史趋势：

```bash
# 运行并生成结果数据
pytest -s --alluredir ./allure-results

# 启动报告服务
allure serve ./allure-results

# 生成静态报告
allure generate ./allure-results -o ./allure-report --clean
```

在测试代码中增加 Allure 注解：

```python
import allure

@allure.epic("用户中心")
@allure.feature("登录模块")
@allure.story("账号密码登录")
def test_login_success():
    with allure.step("输入正确的账号密码"):
        resp = login("admin", "admin123")
    with allure.step("校验登录成功"):
        assert resp.status_code == 200
        allure.attach(resp.text, "响应体", allure.attachment_type.JSON)
```

每次运行前建议清空历史结果，避免旧数据干扰：

```bash
pytest --alluredir ./allure-results --clean-alluredir
```

## 八、项目级配置：pytest.ini 与 conftest.py

### 8.1 pytest.ini 模板

```ini
[pytest]
# 默认命令行参数
addopts = -v --tb=short --alluredir ./allure-results --clean-alluredir

# 测试发现规则
testpaths = tests
python_files = test_*.py
python_classes = Test*
python_functions = test_*

# 自定义标记
markers =
    smoke: 冒烟测试
    regression: 回归测试
    slow: 慢用例
    unstable: 不稳定用例

# 过滤警告
filterwarnings =
    ignore::DeprecationWarning
```

### 8.2 项目级 conftest.py 示例

```python
# conftest.py
import pytest


def pytest_addoption(parser):
    """添加自定义命令行选项，用于多环境切换。"""
    parser.addoption(
        "--env",
        action="store",
        default="test",
        help="测试环境：test / staging / prod",
    )


@pytest.fixture(scope="session")
def env(request):
    return request.config.getoption("--env")


@pytest.fixture(scope="session")
def config(env):
    """根据环境加载配置。"""
    return load_config(f"config/{env}.yaml")
```

## 九、测试开发高频 Checklist

- [ ] 用例命名符合 `test_*.py` / `Test*` / `test_*` 约定。
- [ ] 断言信息足够清晰，失败时能快速判断期望与实际。
- [ ] Fixture 按作用域合理分级，避免过度使用 `function` 造成资源浪费。
- [ ] 资源清理使用 `yield` 或 `addfinalizer`，确保测试间不互相污染。
- [ ] 自定义 mark 在 `pytest.ini` 中注册。
- [ ] 测试数据与测试逻辑分离，优先使用 YAML/JSON 数据驱动。
- [ ] 报告目录使用 `--clean-alluredir` 避免历史数据干扰。
- [ ] CI/CD 集成时通过退出码判断整体结果，而不是解析报告文本。
- [ ] 不稳定用例单独打 `@pytest.mark.unstable` 标记，避免阻塞核心流水线。

## 十、可复用模板

### 10.1 接口测试用例模板

```python
import pytest
import allure


@allure.epic("订单中心")
@allure.feature("订单创建")
class TestOrderCreate:

    @pytest.mark.smoke
    @pytest.mark.parametrize(
        "case",
        load_cases("data/order_create.yaml"),
        ids=lambda c: c["name"],
    )
    def test_create_order(self, case, api_client):
        with allure.step("准备测试数据"):
            payload = case["payload"]

        with allure.step("调用创建订单接口"):
            resp = api_client.post("/orders", json=payload)

        with allure.step("校验返回结果"):
            assert resp.status_code == case["expected_status"]
            assert resp.json()["code"] == case["expected_code"]
```

### 10.2 Fixture 数据准备模板

```python
@pytest.fixture
def order(api_client):
    """创建一个订单并自动清理。"""
    order_id = api_client.post("/orders", json={"sku": "A001"}).json()["data"]["id"]
    yield {"id": order_id}
    api_client.delete(f"/orders/{order_id}")
```

## 参考资源

- [pytest 官方文档](https://docs.pytest.org/)
- [pytest-xdist 官方文档](https://pytest-xdist.readthedocs.io/)
- [Allure Pytest 集成](https://docs.qameta.io/allure-report/frameworks/python/pytest/)
- [Google Software Engineering at Google — 测试章节](https://abseil.io/resources/swe-book)