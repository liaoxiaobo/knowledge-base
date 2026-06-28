---
tags:
  - "#规范"
  - "#Python"
  - "#代码质量"
---
# Python 代码质量与规范

> 整理日期：2026-06-16  
> 适用场景：团队内统一测试代码风格、提升测试脚本可维护性、Code Review 参考

---

## 一、为什么测试代码也要讲质量

测试代码是长期资产，不是一次性的。低质量的测试代码会带来：

- **维护成本高**：用例意图不清，修改时不敢下手。
- **误报率高**：脆弱断言、强依赖顺序、硬编码数据导致频繁失败。
- **可读性差**：新人难以快速理解测试在验证什么。

高质量的测试代码应该像产品代码一样被对待。

---

## 二、命名规范

### 2.1 文件命名

| 类型 | 推荐命名 | 说明 |
|---|---|---|
| 测试模块 | `test_*.py` | 与 Pytest 默认发现规则一致 |
| 测试配置 | `config_*.yaml` / `settings.py` | 区分环境配置 |
| 工具模块 | `utils_*.py` / `helpers.py` | 不混淆为测试用例 |
| 数据文件 | `data_*.json` / `cases_*.xlsx` | 明确是用例数据 |

### 2.2 函数/类命名

| 场景 | 推荐 | 不推荐 |
|---|---|---|
| 测试函数 | `test_login_with_valid_credentials` | `test_1`、`test_login` |
| 测试类 | `TestUserService` | `userTest`、`Test1` |
| 辅助函数 | `build_user_payload` | `foo`、`helper` |
| fixture | `user_client`、`db_session` | `f1`、`fix` |

命名应直接说明**被测对象 + 场景 + 预期结果**。

---

## 三、类型注解

测试代码中使用类型注解可以显著提升可读性和 IDE 提示体验。

```python
from typing import Dict, Any, List

def build_user_payload(username: str, role: str = "guest") -> Dict[str, Any]:
    return {"username": username, "role": role}

def filter_failed_cases(cases: List[Dict[str, Any]]) -> List[Dict[str, Any]]:
    return [c for c in cases if c.get("status") == "failed"]
```

**建议**：
- 测试工具函数、辅助类必须加类型注解。
- 测试函数参数如果是通用 fixture，可酌情注解。
- 复杂字典结构可用 `TypedDict` 或 `dataclass` 替代。

---

## 四、静态检查工具

| 工具 | 作用 | 典型配置 |
|---|---|---|
| `black` | 代码格式化 | 统一团队风格，减少风格争论 |
| `isort` | 导入排序 | 自动分组标准库/第三方/本地导入 |
| `flake8` | 风格与语法检查 | 检测未使用变量、过长行等 |
| `mypy` | 类型检查 | 在 CI 中捕获类型错误 |
| `pylint` | 综合代码质量检查 | 适合作为可选补充 |

**推荐在 CI 中集成**：

```bash
black --check .
isort --check-only .
flake8 .
mypy .
```

---

## 五、测试代码结构

### 5.1 Arrange-Act-Assert（AAA）

每个测试函数尽量清晰地分为三段：

```python
def test_create_user_success():
    # Arrange
    payload = build_user_payload("alice")

    # Act
    response = user_api.create(payload)

    # Assert
    assert response.status_code == 201
    assert response.json()["username"] == "alice"
```

### 5.2 一个测试只验证一个概念

❌ 不佳：一个用例里既验证状态码又验证数据库又验证日志。  
✅ 更好：拆分为 `test_create_user_returns_201`、`test_create_user_persists_to_db`。

### 5.3 避免过度断言

每个断言失败都会阻断后续执行。如果多个断言彼此独立，可用 `pytest-assume` 或拆分用例。

---

## 六、常见反模式

| 反模式 | 问题 | 改进 |
|---|---|---|
| 硬编码魔法值 | 数据变更导致用例失败 | 使用常量或数据工厂生成 |
| 依赖用例执行顺序 | 单独运行会失败 | 每个用例自包含，不共享状态 |
| 捕获所有异常 | 隐藏真正问题 | 只捕获预期的异常类型 |
| 打印代替断言 | 无法自动判定结果 | 明确断言预期行为 |
| 测试代码重复 | 修改时遗漏 | 提取 fixture 或 helper |
| 过于复杂的 setup | 用例意图被淹没 | 使用 fixture 分层管理 |

---

## 七、Docstring 与注释

### 7.1 测试函数 Docstring

```python
def test_login_with_locked_account():
    """验证账号锁定后登录返回 403 并提示账号已锁定。

    前置条件：账号已连续失败 5 次登录。
    """
```

### 7.2 复杂辅助函数 Docstring

```python
def retry_with_backoff(func, max_attempts=3, base_delay=1):
    """指数退避重试。

    Args:
        func: 被重试的函数。
        max_attempts: 最大重试次数。
        base_delay: 首次重试等待秒数，后续按指数增长。

    Returns:
        func 的返回值。

    Raises:
        最后一次重试的异常。
    """
```

---

## 八、实践建议

1. **测试代码与产品代码同标准**：同样要过格式检查、类型检查、Code Review。
2. **优先使用 fixture 管理依赖**：避免在每个用例里重复初始化。
3. **常量集中管理**：URL、状态码、错误提示等放到 `constants.py`。
4. **定期重构测试代码**：删除废弃用例、合并重复逻辑、更新过时的断言。
5. **把测试意图写进名字**：好的测试函数名应该能替代部分注释。

---

## 九、关联文档

- [[Python 语言核心]]
- [[Python 测试编程实践]]
- [[Pytest]]