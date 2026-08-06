---
title: Python 从脚本到工程
date: 2026-08-06 12:06:27
tags:
  - python
categories:
  - 干正事
---



## 1. 名字、对象与引用

Python 变量更准确的说法是“**名字绑定到对象**”。赋值通常不会复制对象，只会建立新的引用：

```python
a = [1, 2]
b = a
b.append(3)

print(a)        # 结果为 [1, 2, 3]，因为 a 和 b 指向的是同一个对象
print(a is b)   # True：is 比较的是内存地址（即它们是否是指向同一个对象）
print(a == b)   # True：== 比较的是对象的值是否相等
```

需要区分两个运算：

- `is`：比较对象的身份（内存地址）。通常用于判断对象是否为单例，例如适合写 `x is None`。
- `==`：调用对象的值比较逻辑，比较“内容是否相等”。

函数参数同样是名字绑定。函数收到的是对象引用，不是“按引用传递”或“按值传递”这两个传统标签中的任意一个：

```python
def mutate(xs: list[int]) -> None:
    # xs 是传入对象的引用。直接调用其方法会修改原始对象。
    xs.append(1)

def rebind(xs: list[int]) -> None:
    # 赋值操作让局部名字 xs 指向了一个全新的列表对象 [1]
    # 外部的 data 依然指向原来的对象，不会被改变。
    xs = [1]

data: list[int] = []
mutate(data)
print(data)               # [1]

rebind(data)
print(data)               # 仍然是 [1]
```

这套模型能解释大量常见 Bug：默认可变参数、浅拷贝、缓存污染以及多线程共享状态。

## 2. 可变对象、不可变对象与拷贝

Python 中的对象可以分为可变和不可变两类。常见不可变对象包括 `int`、`float`、`bool`、`str`、`bytes` 以及元素均不可变时的 `tuple`；常见可变对象包括 `list`、`dict`、`set` 以及大多数用户自定义的实例对象。

#### 默认参数只创建一次

在定义函数时，默认参数会被求值并保存下来，而不是每次调用函数时重新创建。如果默认参数是可变对象，就会导致多次调用共享同一个对象。

```python
# 错误：同一个列表会被多次调用共享
def collect_bad(x: int, result: list[int] = []) -> list[int]:
    result.append(x)
    return result

print(collect_bad(1))  # [1]
print(collect_bad(2))  # [1, 2]，而不是预期的 [2]

# 正确示范：用 None 表示“未提供”，在函数内部创建新列表
def collect(x: int, result: list[int] | None = None) -> list[int]:
    if result is None:
        result = []
    result.append(x)
    return result

print(collect(1))      # [1]
print(collect(2))      # [2]
```

默认参数在函数定义时求值，而不是每次调用时求值。

#### 浅拷贝与深拷贝

在处理复杂数据结构（如列表嵌套列表）时，赋值只会创建新的引用。如果想复制数据本身，需要使用拷贝。

```python
import copy

src = [[1], [2]]
shallow = src.copy()           # 只复制最外层列表
deep = copy.deepcopy(src)      # 递归复制内部对象

shallow[0].append(9)
print(src)                     # [[1, 9], [2]]
print(deep)                    # [[1], [2]]
```

深拷贝并非所有场景下的默认答案。对大型张量、深度学习模型状态或庞大字典做无意复制，可能产生巨大的内存和时间开销。在工程实践中，更好的做法是先明确：哪些数据需要独占修改，哪些数据可以作为只读状态在不同组件间共享。

## 3. 作用域、闭包与 LEGB

Python 按照 **LEGB** 顺序查找名字的作用域：
1. **L**ocal（局部）：当前函数或方法内部。
2. **E**nclosing（外层函数）：嵌套函数中，外层函数的局部作用域。
3. **G**lobal（全局）：当前模块（文件）级别。
4. **B**uiltins（内置）：Python 预定义的内置名称（如 `len`, `print`）。

闭包是指内层函数会保存对其外层作用域中变量的引用：

```python
from collections.abc import Callable

def make_multiplier(scale: float) -> Callable[[float], float]:
    # 外层函数接收 scale 参数
    def multiply(x: float) -> float:
        # 内层函数 multiply 记住了外层函数的 scale
        return x * scale
    return multiply

double = make_multiplier(2.0)
print(double(3.0))  # 6.0
```

在使用循环创建闭包时，要注意**延迟绑定**问题：闭包在被实际调用时，才会去外层作用域读取变量的当前值。

```python
# 错误示范：三个函数最终调用时读取同一个 i，而循环结束时 i 的值是 2
bad = [lambda: i for i in range(3)]
print([fn() for fn in bad])   # [2, 2, 2]

# 正确示范：利用默认参数在函数定义时立即求值并保存当前 i 的值
good = [lambda i=i: i for i in range(3)]
print([fn() for fn in good])  # [0, 1, 2]
```

如果需要在内层函数中重新绑定外层函数的名字，使用 `nonlocal`；重新绑定模块全局名字则使用 `global`。优先通过参数和返回值传递状态，通常更易测试。

## 4. 类、数据模型与协议

Python 的特殊方法让对象接入语言协议。例如，实现 `__len__` 后可调用 `len(obj)`，实现 `__iter__` 后可被 `for` 遍历。

```python
from dataclasses import dataclass
from collections.abc import Iterator

@dataclass
class Batch:
    samples: list[list[float]]

    def __len__(self) -> int:
        return len(self.samples)

    def __iter__(self) -> Iterator[list[float]]:
        return iter(self.samples)

batch = Batch([[1.0, 2.0], [3.0, 4.0]])
print(len(batch))       # 2，自动调用 batch.__len__()

for sample in batch:    # 自动调用 batch.__iter__() 获取迭代器
    print(sample)       # 依次输出 [1.0, 2.0] 和 [3.0, 4.0]
```

常见特殊方法及用途：

| 方法                     | 触发方式            | 典型用途                             |
| ------------------------ | ------------------- | ------------------------------------ |
| `__init__`               | 创建实例后          | 初始化对象的内部状态                 |
| `__repr__`               | `repr(obj)`、调试器 | 提供给开发者看的无歧义调试信息表示   |
| `__call__`               | `obj(...)`          | 让类实例可以像函数一样被直接调用     |
| `__enter__` / `__exit__` | `with obj:`         | 管理资源的获取与释放（上下文管理器） |
| `__iter__` / `__next__`  | `for`、`next`       | 自定义迭代逻辑与返回元素             |
| `__getitem__`            | `obj[key]`          | 支持索引、切片或类似字典的键值访问   |

#### 实例方法、类方法和静态方法

在类内部，你可以定义三种方法：
- **实例方法**：第一个参数必须是 `self`，用于操作或访问具体对象的实例属性。
- **类方法**（`@classmethod`）：第一个参数必须是 `cls`，它接收类本身而不是实例。常用于实现工厂模式或备选构造器。
- **静态方法**（`@staticmethod`）：不接收隐式参数（没有 `self` 或 `cls`），它就像是普通的工具函数，只是为了代码组织方便而放在了类的命名空间中。

```python
@dataclass
class Device:
    kind: str
    index: int

    @classmethod
    def parse(cls, value: str) -> "Device":
        # cls 指代 Device 类本身。
        # 这种模式允许我们通过不同的输入格式（如字符串）来创建实例。
        kind, index = value.split(":")
        return cls(kind=kind, index=int(index))

device = Device.parse("cuda:0")
```

## 5. 迭代器与生成器：按需生产数据

**可迭代对象**是指能返回一个迭代器的对象（如列表、字典、字符串）；**迭代器**是负责保存遍历状态的对象，并通过 `__next__` 逐个产出元素。

生成器函数是编写迭代器的一种极其简洁的方式，只需在函数中使用 `yield` 关键字。

```python
from collections.abc import Iterator

def read_batches(path: str, batch_size: int) -> Iterator[list[str]]:
    batch: list[str] = []
    # 假设我们读取一个非常大的文件
    with open(path, encoding="utf-8") as file:
        for line in file:
            batch.append(line.rstrip("\n"))
            if len(batch) == batch_size:
                yield batch
                batch = []
    if batch:
        yield batch

# 假设 data.txt 非常大，使用生成器不会占用大量内存
# for batch in read_batches("data.txt", 100):
#     process(batch)
```

`yield` 返回一个值后暂停函数，下一次迭代从暂停位置继续。与一次性构造完整列表相比，生成器的峰值内存通常只与一个批次相关，适合流式数据管线。

在工程实践中需要记住：
- 生成器通常只能消费一次，遍历结束后它就“空”了；
- 生成器是惰性求值的，如果内部发生了异常，只有在实际遍历到异常对应的数据时才会抛出；
- 把生成器强制转成列表（如 `list(generator)`）会立即执行完整流程并重新把所有元素加载到内存中，这会失去生成器节省内存的优势。

## 6. 装饰器：在调用边界附加行为

装饰器本质上是一个接收可调用对象（如函数或类）并返回另一个可调用对象的高阶函数。它可以在不修改原函数内部代码的前提下，在函数调用前后附加额外的逻辑。

下面是一个计时器装饰器的完整示例。请注意它是如何使用 `functools.wraps` 来保留原函数的元信息的：

```python
from collections.abc import Callable
from functools import wraps
from time import perf_counter
from typing import ParamSpec, TypeVar

P = ParamSpec("P")
R = TypeVar("R")

def timed(fn: Callable[P, R]) -> Callable[P, R]:
    @wraps(fn)  # 关键点：保留被装饰函数的名字、文档字符串和签名
    def wrapper(*args: P.args, **kwargs: P.kwargs) -> R:
        start = perf_counter()
        try:
            # 执行原始函数逻辑
            return fn(*args, **kwargs)
        finally:
            # 函数执行完毕后附加计时逻辑
            elapsed_ms = (perf_counter() - start) * 1_000
            print(f"{fn.__name__}: {elapsed_ms:.3f} ms")
    return wrapper

@timed
def preprocess(values: list[float]) -> list[float]:
    return [value * 2 for value in values]
```

语法糖 `@timed` 等价于手动执行 `preprocess = timed(preprocess)`。在使用装饰器时，务必加上 `functools.wraps`，否则被装饰后的函数名会变成内部的 `wrapper`，这对日志排查、单元测试以及依赖反射机制的框架（如 FastAPI、Django）是致命的。

装饰器非常适合用来抽离横切关注点，比如：日志记录、缓存（如 `functools.lru_cache`）、权限检查、重试机制和性能观测。但不要滥用：如果仅仅是局部某段代码需要计时，一个普通的 `with timer():` 上下文管理器往往更加直白，不必为了显示技术深度而强行封装装饰器。

## 7. 上下文管理器：让资源必定释放

`with` 语句允许我们将资源的“获取（setup）”和“释放（teardown）”逻辑集中在一个词法范围内。无论块内部的代码是否正常完成，甚至是中途抛出异常、执行了 `return` 或 `break`，资源释放逻辑都必定会被执行。

除了在类中实现 `__enter__` 和 `__exit__` 特殊方法，我们也可以利用 `contextlib.contextmanager` 装饰生成器来快速创建一个上下文管理器：

```python
from contextlib import contextmanager
from collections.abc import Iterator
from time import perf_counter

@contextmanager
def timer(name: str) -> Iterator[None]:
    # 进入 with 块时执行的代码（setup）
    start = perf_counter()
    try:
        # yield 将控制权交还给 with 块内部的代码
        yield
    finally:
        # 退出 with 块时执行的代码（teardown）
        # 无论内部是否发生异常，这段代码都会被执行
        elapsed_ms = (perf_counter() - start) * 1_000
        print(f"{name}: {elapsed_ms:.3f} ms")

# 使用创建的上下文管理器
with timer("生成并处理数据"):
    # 此处的代码在 yield 的位置执行
    data = [value * 2 for value in range(100_000)]
    # 处理完成或者抛出异常后，控制权会返回 finally 块
```

文件操作（`with open(...)`）、线程锁（`with lock:`）、数据库连接管理、临时目录清理、以及深度学习中的自动混合精度上下文（`with torch.autocast(...)`）都采用了这种模式。它解决了在复杂控制流中忘记清理资源的难题，确保了**资源生命周期具有清晰的边界**。

## 8. 异常处理：只捕获能够处理的错误

在工程代码中，异常不仅是报错信息，更是错误处理的一种标准流程。正确的异常捕获与抛出方式对于代码的健壮性至关重要：

```python
def parse_world_size(raw: str) -> int:
    try:
        world_size = int(raw)
    except ValueError as exc:
        # 捕获内置的转换错误，包装成包含更多业务上下文的新异常
        # raise ... from exc 用于在异常堆栈中保留原始异常的痕迹（根因链）
        raise ValueError(f"系统变量 WORLD_SIZE 必须是整数，但实际读取到了 {raw!r}") from exc

    if world_size <= 0:
        raise ValueError("WORLD_SIZE 必须大于 0")
    return world_size
```

在处理异常时，建议严格遵循以下三条工程原则：

1. **捕获具体的异常类**。永远不要使用空的 `except:` 或 `except Exception:` 吞掉所有错误，因为这会意外拦截掉 `KeyboardInterrupt`（导致 `Ctrl+C` 无法终止程序）或 `SystemExit` 等正常信号。如果必须写兜底逻辑，也要明确指定捕获范围并详细记录日志。
2. **只捕获你能恢复、或者需要补充业务上下文的错误**。如果你捕获了一个异常却只能把它 `print` 出来然后让程序继续在错误状态下运行，那不如不捕获，让程序尽早崩溃（Fail Fast）。
3. **保留异常链条**。当你需要把底层异常（如 `KeyError`、`requests.ConnectionError`）转换成本项目的自定义异常时，使用 `raise NewError() from exc` 语法，这样在报错日志中能看到完整的上下文追溯。

另外，不要把异常用于高频的正常控制流（例如在循环中不断地尝试 `try-except` 来替代 `if-else`），这会带来不必要的性能开销。

## 9. 类型标注与数据类

Python 依然是一门动态类型语言。代码中增加的类型标注（Type Hints）主要服务于静态分析工具（如 `mypy`）、IDE 的自动补全以及未来的代码阅读者，Python 解释器本身**不会**在运行时强制验证这些类型。

结合数据类（`dataclasses`）可以极大简化结构化数据的声明：

```python
from dataclasses import dataclass
from collections.abc import Sequence

# frozen=True 表示这个类的实例在创建后不可修改，类似于不可变对象
@dataclass(frozen=True)
class ShardSpec:
    rank: int
    world_size: int
    
    # dataclass 会自动帮你生成 __init__, __repr__, __eq__ 等方法

# Sequence 接口表明 values 支持按索引访问和获取长度（如 list 或 tuple）
def shard(values: Sequence[int], spec: ShardSpec) -> Sequence[int]:
    # 根据分片规则提取数据切片
    return values[spec.rank::spec.world_size]

# IDE 会提示你需要传入 rank 和 world_size
spec = ShardSpec(rank=0, world_size=4)
data = list(range(10))
print(shard(data, spec))
```

常用选择：

- **宽进严出**：函数参数只要求最小能力时，使用抽象协议（如标注为 `Iterable[T]` 或 `Sequence[T]` 而非具体的 `list[T]`）。这能让函数支持更广泛的输入类型。
- **返回具体**：函数返回值通常可以标注得更具体（如 `list[T]` 或是 `dict[K, V]`），方便调用方确切知道得到的是什么数据结构。
- **显式标明缺失可能**：使用 `T | None`（或 `Optional[T]`）清晰地告诉调用方这个值有可能为空，要求对方做空值检查。
- **按需使用 Protocol**：对于需要满足某些特定方法的对象，可以定义结构化类型的 `Protocol` 接口，而不需要强迫对象去继承某个基类。
- **适度标注**：类型系统的目标是尽早在模块和函数接口边界处暴露误用，不要尝试把动态语言硬写成 Java。公共函数的参数和返回值应该严格标注，但函数内部一目了然的局部变量往往不需要逐个标注。