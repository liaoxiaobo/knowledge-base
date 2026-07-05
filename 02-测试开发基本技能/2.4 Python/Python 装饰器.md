---
title: "Python 装饰器"
tags:
  - 理论
  - Python
  - 装饰器
created: "2026-07-06"
updated: "2026-07-06"
---

# Python 装饰器

> 从闭包基础出发，深入解析 Python 装饰器的实现原理，包括函数装饰器、带参数装饰器、类装饰器以及内置装饰器的使用场景。

## 装饰器

在 Python 开发中，装饰器（Decorator）是一种强大而优雅的语法特性，它允许在**不修改原函数代码**的前提下，动态地为函数添加额外功能（如日志记录、性能分析、参数校验等）。本文将从闭包的基础概念出发，深入解析装饰器的实现原理，并通过实际示例展示其应用场景。

## 闭包：装饰器的基石

在理解装饰器之前，必须先了解**闭包（Closure）**的概念。闭包是指在函数内部定义的函数（嵌套函数），该函数可以访问外层函数的变量，即使外层函数已经执行完毕。

```python
def outer():
    x = 10
    def inner():
        print(f"通过闭包访问外层变量x的值: {x}")
    return inner  # 返回闭包

closure = outer()
closure()  # 输出: 通过闭包访问外层变量x的值: 10
```

在这个例子中，`outer` 返回 `inner`，后者可以访问并使用 `outer` 中的变量 `x`。

## 装饰器的实现

装饰器的本质是一个接收函数作为参数的外层函数，其内部定义一个闭包（通常命名为 `wrapper`），用于包裹原函数并增强其功能。

```python
# 装饰器模版
from functools import wraps

def decorator(func):          # 外层函数（装饰器）
    @wraps(func)
    def wrapper(*args, **kwargs):  # 内层函数（闭包）
        # do something
        result = func(*args, **kwargs)  # 访问外层函数的参数 `func`
        return result
    return wrapper
```

在上面的例子中，`decorator` 就是一个装饰器。它接受一个函数 `func` 作为参数，并返回一个包裹原函数的新函数 `wrapper`。

**代码解析**

1. `@wraps(func)`：保留原函数的元数据（如函数名、文档信息）。
2. `*args, **kwargs`：接受任意、不定数量的参数，增强装饰器的通用性。
3. 外层 `return wrapper`：确保装饰器返回包装后的函数，其实就是一个闭包函数。
4. 内层 `return result`：确保原函数的返回值不被丢弃。

## 装饰器应用场景

1. **日志记录**：在大型项目中，使用装饰器可以轻松地为多个函数添加日志功能，而无需在每个函数中重复编写日志代码。
2. **性能分析**：通过装饰器可以方便地测量函数的执行时间，帮助开发者识别性能瓶颈。
3. **访问控制**：在 Web 应用中，装饰器可以用于实现用户认证和授权，确保只有具有特定权限的用户才能访问某些功能。
4. **缓存机制**：对于计算密集型函数，使用缓存装饰器可以显著提高程序的执行效率，避免重复计算。
5. **输入验证**：在数据处理应用中，装饰器可以用于检查函数输入的有效性，提高代码的健壮性。

## 装饰器函数示例

### 日志记录

```python
import logging
from functools import wraps

def log_decorator(func):
    @wraps(func)
    def wrapper(*args, **kwargs):
        # logging.getLogger().setLevel(logging.INFO)
        logging.info(f"调用函数: {func.__name__}, 参数: {args}, {kwargs}")
        result = func(*args, **kwargs)
        logging.info(f"函数返回: {result}")
        return result
    return wrapper

@log_decorator
def add(x, y):
    """两数相加"""
    return x + y

add(3, 5)  # 自动记录日志
```

**代码解析**

1. `@log_decorator` 是装饰器的应用方式，相当于：`add = log_decorator(add)`。
2. 装饰器返回的 `wrapper` 函数能够访问外部函数 `log_decorator` 的作用域，因此可以在原函数执行前后插入自己的逻辑。

### 函数计时

```python
import time

def timer(func):
    def wrapper(*args, **kw):
        t1 = time.time()
        # 这是函数真正执行的地方
        func(*args, **kw)
        t2 = time.time()
        # 计算下时长
        cost_time = t2 - t1
        print("花费时间：{}秒".format(cost_time))
    return wrapper

@timer
def want_sleep(sleep_time):
    time.sleep(sleep_time)

want_sleep(10)
```

### 函数重试

若需要为装饰器传递参数（如重试次数），需定义一个**装饰器工厂函数**，它可以根据参数“生产”出不同的装饰器。

```python
from functools import wraps

def repeat(num_times):
    def decorator(func):
        @wraps(func)
        def wrapper(*args, **kwargs):
            for _ in range(num_times):
                result = func(*args, **kwargs)
            return result
        return wrapper
    return decorator

@repeat(num_times=3)
def greet(name):
    print(f"Hello {name}!")

greet("Alice")  # 输出3次"Hello Alice!"
```

从以上例子中可以看到，在带参数的装饰器函数中一共有三层函数嵌套，其实剥离出最外面的一层，我们可以发现和简单（不带参数）的装饰器是一样的。

**代码解析**：

- `repeat` 是一个装饰器工厂函数，负责“生产”返回一个实际的装饰器 `decorator`。
- `@repeat(num_times=3)` 等价于 `greet = repeat(3)(greet)`。

## 类装饰器

绝大多数装饰器都是基于函数和闭包实现的，但这并非制造装饰器的唯一方式。事实上，Python 对某个对象是否能通过装饰器（`@decorator`）形式使用只有一个要求：**decorator 必须是一个“可被调用（callable）”的对象**。

基于类装饰器的实现，必须实现 `__call__` 和 `__init__` 两个内置函数。

### 不带参数的类装饰器

```python
class Decorator(object):
    def __init__(self, func):  # 接收被装饰函数
        self.func = func

    def __call__(self, *args, **kwargs):  # 实现装饰逻辑
        print(f"函数 {self.func.__name__} 被调用")
        return self.func(*args, **kwargs)

@Decorator
def say_hello():
    print("Hello!")

say_hello()  # 输出: 函数 say_hello 被调用 → Hello!
```

在不带参数的类装饰器中，等价于 `func = class(func)`，此时函数作为了类的实例参数。

### 带参数的类装饰器

```python
class logger(object):
    def __init__(self, level='INFO'):  # 接受装饰器参数
        self.level = level

    def __call__(self, func):  # 接受被装饰函数
        def wrapper(*args, **kwargs):
            print("[{level}]: the function {func}() is running..."
                  .format(level=self.level, func=func.__name__))
            func(*args, **kwargs)
        return wrapper  # 返回函数

@logger(level='WARNING')
def say(something):
    print("say {}!".format(something))

say("hello")
```

在带参数的类装饰器中，等价于 `func = class(para)(func)`。也就是说，此时函数并不是实例化参数，而变成了 callable 的参数，这也就是为什么需要在 `__call__()` 中传入 `func`。

## 内置装饰器

Python 还提供了一些常用的内置装饰器：

- `@property`：用于将一个方法转换为属性。广泛应用在类的定义中，可以让调用者写出简短的代码，同时保证对参数进行必要的检查。
- `@staticmethod` 和 `@classmethod`：用于定义静态方法和类方法。

## 总结

1. 装饰器是闭包的一种应用，闭包为装饰器提供了“记住外部函数参数（如被装饰函数 `func`）”的能力。
2. 一切 callable 的对象都可以被用来实现装饰器，比如函数、类。
3. 装饰器会改变原函数的原始签名，需要使用 `functools.wraps`。
4. 在内层函数修改外层函数的变量时，需要使用 `nonlocal` 关键字。
5. “装饰器”并没有提供某种无法替代的功能，等同于 `func = decorator(func)`。它只是一颗“语法糖”而已，就像乐高积木，可以自由组合各种功能模块，在某些特定场景里可以使代码更简洁易维护。

