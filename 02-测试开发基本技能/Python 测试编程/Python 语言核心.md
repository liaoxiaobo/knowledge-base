---
tags:
  - 理论
  - Python
---
# Python 语言核心

> 来源：`source/_posts/python-interview.md`、`source/_posts/python-decorator.md`  
> 整理日期：2026-06-16  
> 适用场景：测试开发中需要快速判断该用哪种数据结构、理解参数传递行为、编写测试辅助工具

---

## 一、核心数据结构及其测试开发场景

| 数据结构 | 核心特性 | 测试开发典型场景 |
|---|---|---|
| `list` | 有序、可变、支持索引/切片 | 存放用例列表、收集批量执行结果 |
| `tuple` | 有序、不可变、可哈希 | 作为字典键、固定配置项、函数多返回值 |
| `dict` | 键值映射、查找 O(1) | 存储测试配置、接口请求体/响应体断言 |
| `set` | 去重、成员检测 O(1) | 去重失败用例 ID、集合运算 diff 两个结果集 |
| `frozenset` | 不可变集合、可哈希 | 需要把集合作为字典键或放入其他集合时 |
| `deque` | 双端队列、两端操作 O(1) | 滑动窗口、最近 N 条日志/用例缓存 |

**测试开发中的选择原则**：
- 需要随机访问和尾部追加 → `list`
- 需要快速查找/去重 → `set` / `dict`
- 需要不可变以保证不被测试副作用修改 → `tuple` / `frozenset`
- 需要 FIFO/LIFO 或滑动窗口 → `deque`

---

## 二、可变与不可变：参数传递的陷阱

Python 参数传递是**对象引用传递**。理解这一点对测试数据隔离至关重要。

```python
def append_item(items, item):
    items.append(item)  # 原地修改，外部可见

config = {"timeout": 5}
def update_config(c):
    c = {"timeout": 10}  # 重新赋值，外部不可见
```

**测试开发启示**：
- 用默认参数 `def foo(items=None)` 避免可变默认参数陷阱。
- 断言前对被测函数的返回值和入参分别校验，防止函数意外修改了测试数据。
- 深拷贝测试数据，避免多个用例共享同一对象导致互相污染。

---

## 三、浅拷贝与深拷贝

| 操作 | 效果 | 测试场景 |
|---|---|---|
| `b = a` | 引用同一对象 | 不建议用于测试数据准备 |
| `copy.copy(a)` | 浅拷贝，嵌套对象仍共享引用 | 简单结构的数据模板 |
| `copy.deepcopy(a)` | 递归拷贝所有嵌套对象 | 复杂测试数据独立副本 |

```python
import copy

base_case = {
    "request": {"url": "/api/user", "method": "GET"},
    "expected": {"code": 200}
}

case_a = copy.deepcopy(base_case)
case_a["request"]["url"] = "/api/order"  # 不影响 base_case
```

---

## 四、装饰器：测试工具的核心实现手段

装饰器本质是**接收 callable 并返回 callable 的高阶函数**，常用于日志、计时、重试、参数校验等测试工具。

### 4.1 函数装饰器模板

```python
from functools import wraps

def my_decorator(func):
    @wraps(func)  # 保留原函数元数据
    def wrapper(*args, **kwargs):
        # 前置逻辑
        result = func(*args, **kwargs)
        # 后置逻辑
        return result
    return wrapper
```

### 4.2 带参数的装饰器工厂

```python
def retry(max_attempts=3):
    def decorator(func):
        @wraps(func)
        def wrapper(*args, **kwargs):
            for attempt in range(1, max_attempts + 1):
                try:
                    return func(*args, **kwargs)
                except Exception as e:
                    if attempt == max_attempts:
                        raise
        return wrapper
    return decorator
```

### 4.3 类装饰器

当需要保存状态或提供多个配置项时，可用类实现装饰器。

```python
class CallCounter:
    def __init__(self, func):
        self.func = func
        self.count = 0

    def __call__(self, *args, **kwargs):
        self.count += 1
        return self.func(*args, **kwargs)
```

---

## 五、反射与动态加载

测试框架常需要根据配置动态发现用例或调用方法。

```python
# 动态检查并调用方法
obj = SomeApiClient()
method_name = "get_user"
if hasattr(obj, method_name):
    result = getattr(obj, method_name)(user_id=1)

# 动态导入模块
module = __import__("tests.scenarios.login")
```

**典型应用**：
- 根据 Excel/JSON 用例模板动态映射到测试函数。
- 插件化测试工具，按目录自动发现并加载扩展。

---

## 六、并发模型速览

| 模型 | 适用场景 | 测试开发应用 |
|---|---|---|
| 多线程 `threading` | IO 密集型 | 并发发送接口请求、并行拉取日志 |
| 多进程 `multiprocessing` | CPU 密集型 | 大量数据解析、图片/日志处理 |
| 协程 `asyncio` | 高并发 IO | 异步接口压测客户端、批量异步任务 |

**注意 GIL**：CPython 的多线程在 CPU 密集型任务中无法真正并行；IO 密集型任务因 IO 释放 GIL，仍可受益。

---

## 七、垃圾回收与测试稳定性

Python 以引用计数为主，分代回收为辅。测试开发中需关注：

- **循环引用**：测试对象之间互相引用可能导致内存无法及时释放，长时间运行会出现内存增长。
- **资源释放**：文件、数据库连接、网络会话等应在 `try/finally` 或上下文管理器中关闭，避免测试环境资源泄漏。

---

## 八、实践建议

1. **优先使用标准库数据结构**：不要重复造轮子，`collections` 模块已提供 `deque`、`defaultdict`、`Counter` 等利器。
2. **测试数据要隔离**：复杂对象用 `copy.deepcopy`，避免用例间互相污染。
3. **用装饰器封装横切逻辑**：日志、计时、重试、限流等通用能力抽象为装饰器，保持业务测试代码简洁。
4. **理解 GIL 限制**：CPU 密集型任务不要用多线程硬扛，考虑多进程或外部服务。
5. **避免可变默认参数**：`def foo(items=[]): ...` 是常见坑，应使用 `items=None` 并在函数内初始化。
