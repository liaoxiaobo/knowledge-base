---
title: "Python 面试题汇总"
tags:
  - 理论
  - Python
  - 面试
created: "2026-07-06"
updated: "2026-07-06"
---

# Python 面试题汇总

> 汇总 Python 面试中常见的数据操作、文件处理、语言特性、面向对象与并发相关题目，适用于面试准备和快速回顾 Python 核心知识点。

## 一、数据操作

### 1.1 如何将字符串 "ilovechina" 进行反转

```python
s = 'ilovechina'
s = s[::-1]
```

### 1.2 如何将 "1,2,3" 分割成 ["1","2","3"]？

```python
s = "1,2,3"
stdout = s.split(',')
```

### 1.3 Python 中的字符串格式化方式你知道哪些？

Python 3.6 之后的版本提供了三种字符串格式化的方式：

```python
# 1. %s 占位符
def foo(name):
    return 'hello %s' % name

# 2. format()
def foo(name):
    return 'hello {}'.format(name)

# 3. f-string
def foo(name):
    return f'hello {name}'
```

### 1.4 列表元素怎么去重

**自己动手写函数实现（保留原顺序）**

```python
old_list = [2, 3, 4, 5, 1, 2, 3]
new_list = []
for i in old_list:
    if i not in new_list:
        new_list.append(i)
print(new_list)  # [2, 3, 4, 5, 1]
```

**用字典 dict 去重（保留原顺序）**

```python
# 使用 list 元素值创建一个 dict，这将自动删除任何重复项
old_list = [2, 3, 4, 5, 1, 2, 3]
new_list = list(dict.fromkeys(old_list))
print(new_list)  # [2, 3, 4, 5, 1]
```

**使用 sort 函数来去重（保留原顺序）**

```python
lst1 = [2, 1, 3, 4, 1]
lst2.sort(key=lst1.index)
print(lst2)  # [2, 1, 3, 4]
```

**使用内置 set 方法（不保证原顺序）**

```python
# 将 list 转化为 set 再转化为 list
list1 = [1, 3, 4, 3, 2, 1]
list2 = list(set(list1))
print(list2)  # {1, 2, 3, 4}
```

### 1.5 两个列表怎么转成字典

**动手写一个函数**

```python
list1 = ['name', 'age', 'sex']
list2 = ['张三', 18, '男']
dict = {}
for i in range(len(list1)):
    dict[list1[i]] = list2[i]
print(dict, type(dict))
# {'name': '张三', 'age': 18, 'sex': '男'} <class 'dict'>
```

**使用内置函数 zip**

```python
list1 = ['name', 'age', 'sex']
list2 = ['张三', 18, '男']
d = dict(zip(list1, list2))
print(d)  # {'name': '张三', 'age': 18, 'sex': '男'}
```

### 1.6 如何合并两个列表

```python
a = [1, 5, 7, 9]
b = [2, 2, 6, 8]

# 方法1
a.extend(b)

# 方法2
a[0:0] = b

# 方法3
a += b
```

### 1.7 举例 sort 和 sorted 的区别

```
sort() 与 sorted() 的不同在于，sort 是在原位重新排列列表，而 sorted() 是产生一个新的列表。
```

### 1.8 如何把元组 ("a","b") 、(1,2)，变为字典 {"a":1,"b":2}？

```python
a = ('a', 'b')
b = (1, 2)
z = zip(a, b)
c = dict(z)
```

### 1.9 字典操作中 del 和 pop 有什么区别？

```
del 操作删除键值对，不返回值；
pop 操作删除键值对的同时，返回键所对应的值。
```

### 1.10 如何合并两个字典

```python
# python3 合并字典有三种方式
# 1.
a = {'a': 1, 'b': 2}
b = {'c': 3, 'd': 4}
c = {}
c.update(a)
c.update(b)

# 2.
c = dict(a, **b)

# 3. 动态解包
c = {**a, **b}  # 官方推荐这种方式
```

## 二、文件处理

### 2.1 read、readline、readlines 的作用

- `read()`：一次读取文本全部内容为字符串（可以指定 size 去读取），占用空间大。
- `readline()`：读取文本为一个生成器，支持遍历和迭代，占用空间小。
- `readlines()`：一次读取文本全部内容为列表，占用空间大。

### 2.2 序列化与反序列化

- **序列化**：将数据结构或对象转换成二进制串的过程。例如：将字典转换为 JSON 字符串或二进制数据。
- **反序列化**：将在序列化过程中所生成的二进制串转换成原始的数据结构或对象。例如：将 JSON 字符串或二进制数据解析为字典。

```python
import json

data = {'status': 'success', 'result': [1, 2, 3]}
# 序列化为 JSON 字符串
json_str = json.dumps(data)
# 发送给 Java 服务端后，Java 可以通过 JSON 库解析
```

## 三、语言特性

### 3.1 Python 中的基本数据结构

> 基本结构通常指列表、元组、字典、集合，而字符串可能作为序列类型单独列出。

| 数据结构       | 特点                              | 实现原理                           | 应用场景                                                   |
| -------------- | --------------------------------- | ---------------------------------- | ---------------------------------------------------------- |
| **列表 list**  | 有序、可变、允许重复元素          | 动态数组存储，支持自动扩容         | 动态数据读取/更新、栈/队列实现                             |
| **元组 tuple** | 有序、不可变、允许重复元素        | 静态数组存储                       | 存储固定数据（如函数参数传递、配置参数），比列表更节省内存 |
| **字典 dict**  | 有序（Python 3.7+）、可变、键值对 | 哈希表                             | 快速查询、JSON 数据处理                                    |
| **集合 set**   | 无序、可变、唯一性（自动去重）    | 哈希表（仅键）                     | 去重、集合运算（交集、并集）、成员判断                     |
| **字符串**     | 有序、不可变、字符序列            | 动态编码机制存储字符，如 UTF-8、GBK | 文本处理、数据序列化（JSON/XML 格式）                       |

- 除了 set，其余都属于序列类型（索引、切片、操作符）。
- 列表和元组最大区别：元组是静态数组，不可变对象，即不支持修改。

### 3.2 Python 函数参数的传递机制

Python 的函数参数传递机制是**对象引用传递（Pass by Object Reference）**，其核心是传递对象的引用，而非对象本身或对象的值。这一机制的行为取决于对象是否可变。

#### 如何理解对象引用传递

- **所有变量都是对象的引用**：Python 中变量名本质上是对象的引用（指针），即所有数据（如整数、字符串、列表等）都以对象形式存在，变量仅指向这些对象。
- **传递的是引用的拷贝**：函数调用时，实参对象的引用（地址）会被拷贝给形参。因此，形参和实参**指向同一个对象**，但引用本身是独立的。

#### 不可变对象的传递

**不可变对象**（如 `int`、`str`、`tuple` 等）的值不能被修改，因此：

- **函数内部无法修改原对象**：对形参的“修改”都会创建新对象（如重新赋值），外部变量不变，行为类似“值传递”。

```python
def modify_immutable(x):
    print(f"函数内原始x的id: {id(x)}")
    x = 10  # 创建新对象，原引用被替换
    print(f"修改后x的id: {id(x)}")

x = 5
print(f"调用前x的id: {id(x)}")
modify_immutable(x)
print(f"调用后x的值: {x}")  # 输出：5（未改变）

# 输出:
# 调用前x的id: 4353896768
# 函数内原始x的id: 4353896768
# 修改后x的id: 4353896928
# 调用后x的值: 5
```

#### 可变对象的传递

**可变对象**（如 `list`、`dict`、`set` 等）的值可以被修改，因此：

- **函数内部可直接修改原对象**：通过原地操作（如 `append`、`update`）会修改原对象，会影响外部变量，行为类似“引用传递”。
- 特殊情况：如果在函数内部重新赋值（如 `lst = new_list`），形参的引用会指向新对象，外部变量不变。

```python
def modify_mutable(lst):
    print(f"函数内原始lst的id: {id(lst)}")
    lst.append(4)  # 原地修改，对象内容改变
    print(f"原地修改后lst的id: {id(lst)}")
    lst = [100, 200]  # 此处重新赋值，形参lst指向新对象，但原my_list未受影响
    print(f"重新赋值后lst的id: {id(lst)}")

my_list = [1, 2, 3]
print(f"调用前my_list的id: {id(my_list)}")
modify_mutable(my_list)
print(f"调用后my_list的内容: {my_list}")  # 输出：[1, 2, 3, 4]

# 输出:
# 调用前my_list的id: 4408370176
# 函数内原始lst的id: 4408370176
# 原地修改后lst的id: 4408370176
# 重新赋值后lst的id: 4404760832
# 调用后my_list的内容: [1, 2, 3, 4]
```

### 3.3 Python 垃圾回收机制

Python 的垃圾回收（Garbage Collection, GC）机制旨在自动管理内存，回收不再被使用的对象，避免内存泄漏。其核心机制结合了**引用计数**和**分代回收（Generational Collection）**，同时通过 `gc` 模块提供手动控制的灵活性。

#### 核心机制：引用计数

**基本原理**

- **每个对象都有一个引用计数器**：记录当前有多少变量或对象引用它。
- 引用计数的增减：
  - 当对象被创建时，初始引用计数为 1。
  - 每次有新的变量或对象引用该对象时，计数器加 1。
  - 当引用失效（如变量被删除、超出作用域、重新赋值）时，计数器减 1。
  - **当计数器降为 0 时**，Python 立即释放该对象的内存。

**示例**

```python
a = [1, 2, 3]  # 列表对象的引用计数为1（被a引用）
b = a          # 引用计数变为2（被a和b引用）
del a          # 引用计数减为1（仅b引用）
b = None       # 引用计数降为0，列表对象被回收
```

**引用计数的局限性**

- 无法处理循环引用：当两个或多个对象互相引用时，引用计数永远不会为 0，导致内存泄漏。

```python
a = []
b = []
a.append(b)   # a引用b
b.append(a)   # b引用a
del a, b      # 循环引用的计数仍为1，无法回收
```

#### 分代回收（Generational Collection）

**作用**：解决循环引用问题，通过周期性扫描检测并回收被循环引用的对象。

分代回收的整体思想是：将系统中的所有内存块根据其存活时间划分为不同的集合，每个集合就成为一个“代”，垃圾收集频率随着“代”的存活时间的增大而减小，存活时间通常利用经过几次垃圾回收来度量。

### 3.4 函数装饰器有什么作用？请列举说明

函数装饰器可以在不修改原函数的条件下，为原函数添加额外的功能，例如记录日志、运行性能、缓存等。

### 3.5 @classmethod 和 @staticmethod 用法和区别

| 对比项         | `@classmethod`                                       | `@staticmethod`            |
| -------------- | ---------------------------------------------------- | -------------------------- |
| 参数           | 必须有 `cls`（类本身）                               | 无特殊参数，类似普通函数   |
| 访问权限       | 可访问/修改类属性                                    | 无法访问类或实例属性       |
| 依赖性         | 依赖类的上下文                                       | 与类或实例无关             |
| 典型用途       | 工厂方法（替代构造函数来创建对象）、操作类属性、多态场景 | 工具函数、逻辑相关但独立的功能 |
| 调用灵活性     | 推荐通过类名调用                                     | 推荐通过类名调用           |

### 3.6 你了解 Python 中的反射吗？

Python 的反射能力由一系列内置函数和模块支持，允许代码在运行时根据字符串名称操作对象，实现高度动态的编程。比如动态操作对象属性、调用方法、模块导入等。

#### 内置函数

1. `hasattr(obj, name)`：检查对象是否包含指定属性/方法。
2. `getattr(obj, name, default)`：获取对象的属性/方法，若不存在则返回 `default` 或抛出错误。
3. `setattr(obj, name, value)`：动态设置对象的属性值。
4. `delattr(obj, name)`：删除对象中的属性/方法。

#### 应用场景

**动态调用方法**

```python
import requests

class Http(object):

    def get(self, url):
        res = requests.get(url)
        response = res.text
        return response

    def post(self, url):
        res = requests.post(url)
        response = res.text
        return response

# 使用反射后
url = "https://www.jianshu.com/u/14140bf8f6c7"
method = input("请求方法>>>:")
h = Http()

if hasattr(h, method):
    func = getattr(h, method)
    res = func(url)
    print(res)
else:
    print("你的请求方式有误...")
```

**动态属性操作**

```python
class DynamicObject:
    pass

obj = DynamicObject()
setattr(obj, "dynamic_attr", 42)
print(getattr(obj, "dynamic_attr"))  # 输出 42
```

**动态加载模块或类**

```python
module_name = "math"
module = __import__(module_name)
print(getattr(module, "sqrt")(16))  # 输出 4.0
```

#### inspect 模块

`inspect` 模块提供更强大的内省功能。

严格来说，内省是反射的一部分，主要用于查看分析函数/类的元数据。简单一句就是运行时能够获得对象的类型，比如 `type()`、`dir()`、`getattr()`、`hasattr()`、`isinstance()`。

```python
import inspect

def my_function(a, b):
    """示例函数"""
    return a + b

# 获取函数参数信息
print(inspect.signature(my_function))  # 输出 (a, b)
# 获取文档字符串
print(inspect.getdoc(my_function))     # 输出 "示例函数"
# 获取源代码
print(inspect.getsource(my_function))  # 输出函数定义的源代码
```

### 3.7 copy 和 deepcopy 的区别是什么？

**浅拷贝**：创建新对象，赋值的时候非递归地复制子对象的引用。

**深拷贝**：创建新对象，赋值的时候递归地复制子对象的引用，即拷贝了包括多层嵌套的元素。

```python
import copy

a = [1, 2, 3, 4, ['a', 'b']]  # 原始对象

b = a  # 赋值，传对象的引用
c = copy.copy(a)  # 对象拷贝，浅拷贝
d = copy.deepcopy(a)  # 对象拷贝，深拷贝

a.append(5)  # 修改对象a
a[4].append('c')  # 修改对象a中的['a', 'b']数组对象

print('a = ', a)
print('b = ', b)
print('c = ', c)
print('d = ', d)

# 输出结果：
# a =  [1, 2, 3, 4, ['a', 'b', 'c'], 5]
# b =  [1, 2, 3, 4, ['a', 'b', 'c'], 5]
# c =  [1, 2, 3, 4, ['a', 'b', 'c']]
# d =  [1, 2, 3, 4, ['a', 'b']]
```

### 3.8 进程与线程的区别

- 进程是 CPU 资源分配的基本单位，线程是独立运行与调度的基本单位（CPU 上真正运行的是线程）。
- 进程拥有自己的资源空间，同一个进程中的多个线程并发执行，这些线程共享进程所拥有的资源，但需处理线程安全问题（如锁机制）。
- 线程的调度与切换比进程快很多。

### 3.9 Python 中的 GIL

CPython 环境下，每个进程都拥有一个 GIL（Global Interpreter Lock），单进程下的多线程，某个线程想要执行，必须先获取 GIL，才可以进入 CPU 执行。因此，一个 Python 进程中的多个线程不能同时使用多个 CPU 核心（Java 中一个进程中的多线程可以利用多核 CPU）。

GIL 释放的情况：

1. 执行的字节码行数到达一定阈值。
2. 通过时间片划分，到达一定时间阈值。
3. 在遇到 IO 操作时，主动释放。

#### GIL 带来的影响

**多线程**：

- Python 的 GIL 会强制解释器在同一时刻只能执行一个线程的字节码。因此，**多线程在 CPU 密集型任务中无法真正并行**，反而可能因线程切换导致效率下降。
- 不过 Python 标准库中的所有阻塞型 I/O 函数都会释放 GIL，等待 IO 时可以切换线程执行其他任务，因此**多线程适合 IO 密集型任务**（如网络请求、爬虫、文件读写）。

**多进程**：

- 每个进程拥有独立的 Python 解释器和 GIL，因此**多进程可以并行利用多核 CPU，适合 CPU 密集型任务**（如科学计算、数据处理）。

### 3.10 Python 协程是什么

**协程是一种比线程更加轻量级的存在，最重要的是，协程不被操作系统内核管理，协程是完全由程序控制的。**

- 运行效率极高，协程的切换完全由程序控制，不像线程切换需要花费操作系统的开销，线程数量越多，协程的优势就越明显。
- 协程不需要多线程的锁机制，因为只有一个线程，不存在变量冲突。
- 对于多核 CPU，利用多进程 + 协程的方式，能充分利用 CPU，获得极高的性能。

Python 里最常见的 `yield` 就是协程的思想！

### 3.11 什么是单例模式

单例模式的主要目的是保证在系统中，某个类只能有一个实例存在。比如保存系统基本配置信息的类，在很多地方都要用到，没有必要频繁创建实例与销毁实例，只需要保存一个全局的实例对象即可，这样可以减少对内存资源的占用。

**方法一：模块级实例**

可以在模块中定义单例类并实例化对象，在用的时候直接导入这个模块的实例对象即可。

```python
class GlobalConfig:
    host = 'xxx.xxx'
    port = 3306
    username = 'username'
    password = '123123'

g = GlobalConfig()
```

**方法二：`__new__()` 构造方法**

`__new__()` 是类的构造方法，在实例创建前被调用，它的作用就是创建实例对象。利用这个方法和类的属性的特点可以实现设计模式的单例模式。

```python
# 多线程下创建单例对象需要加锁
import threading

class Singleton:
    _instance_lock = threading.Lock()

    def __new__(cls, *args, **kwargs):
        # 加锁
        with cls._instance_lock:
            if not hasattr(cls, 'instance'):
                cls.instance = super().__new__(cls)
        return cls.instance

def func():
    s = Singleton()
    print('s:{}'.format(id(s)))

if __name__ == '__main__':
    for _ in range(5):
        td = threading.Thread(target=func)
        td.start()
```

