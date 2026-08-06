---
title: Python 并发与性能分析
tags:
  - python
categories:
  - 干正事
---


## 1 并发与并行

- **并发（concurrency）**：多个任务在同一时间段内推进；
- **并行（parallelism）**：多个任务在同一时刻真正执行。

选择并发模型前，先判断瓶颈：

| 任务类型        | 典型例子                 | 常见选择                                                                      |
| --------------- | ------------------------ | ----------------------------------------------------------------------------- |
| I/O 密集        | 请求服务、读取大量小文件 | 线程或 `asyncio`（基于协程（Coroutine）和事件循环（Event Loop）。本质单线程） |
| Python CPU 密集 | 纯 Python 解析、复杂循环 | 多进程或原生扩展                                                              |
| 原生算子密集    | NumPy/PyTorch/CUDA 运算  | 由底层线程池/GPU 并行，Python 负责调度                                        |
| 大数据跨进程    | 数据加载、共享缓存       | 多进程 + 共享内存，谨慎序列化                                                 |

### 3.2 GIL 到底限制了什么

常见 CPython 构建使用全局解释器锁（GIL）保护解释器内部状态。同一进程中，通常只有一个线程能同时执行 Python 字节码。因此，两个执行纯 Python CPU 循环的线程一般不能获得两倍速度。

但这不等于“Python 线程没有用”：

- 阻塞 I/O 时解释器会释放 GIL；
- NumPy、PyTorch 等原生扩展可在耗时计算时释放 GIL；
- 后台日志、预取和轻量控制任务常适合线程。

GIL 是 CPython 实现细节，不应泛化成所有 Python 实现或所有构建的永恒规则。判断性能时要测量当前解释器和依赖，而不是只背结论。

### 3.3 线程：共享内存，也共享风险

```python
from concurrent.futures import ThreadPoolExecutor
from urllib.request import urlopen

def fetch_size(url: str) -> int:
    with urlopen(url, timeout=10) as response:
        return len(response.read())

urls = ["https://example.com"] * 4
with ThreadPoolExecutor(max_workers=4) as pool:
    sizes = list(pool.map(fetch_size, urls))
```

线程共享进程内存，传数据便宜，但对同一可变对象的“读取—修改—写回”可能产生竞态。需要锁时让临界区尽可能小：

```python
from threading import Lock

lock = Lock()
counter = 0

def increment() -> None:
    global counter
    with lock:
        counter += 1
```

不要依赖“某个内置操作看起来是原子的”来设计正确性；解释器版本、对象实现或复合操作都可能破坏这种假设。
- 例：`counter += 1` 至少包含三个独立指令：加载原值、执行加法、存储新值

### 3.4 多进程：绕开 GIL，但数据搬运有成本

```python
from concurrent.futures import ProcessPoolExecutor

def count_primes(limit: int) -> int:
    def is_prime(value: int) -> bool:
        if value < 2:
            return False
        return all(value % divisor for divisor in range(2, int(value**0.5) + 1))
    return sum(is_prime(value) for value in range(limit))

if __name__ == "__main__":
    with ProcessPoolExecutor() as pool:
        results = list(pool.map(count_primes, [50_000] * 4))
```

`if __name__ == "__main__":` 对使用 `spawn` 启动子进程的平台尤其重要，可避免子进程重新导入模块时递归创建新进程。

多进程的主要成本包括：

- 创建进程和独立解释器；
- 参数与返回值的序列化/反序列化（主进程向子进程分发参数5000，int 序列化为二进制，子进程再反序列化，return value 同理）；
- 复制或映射内存；
- 进程间通信与同步。

如果每个任务只做几十微秒工作，调度成本可能比计算更高。应增大任务粒度，并尽量避免来回传递大型数组或张量。

> **GPU 关联**：CUDA 上下文和多进程启动方式组合不当会引发初始化错误或额外显存占用。使用框架的数据加载与分布式能力时，应遵循框架推荐的启动器和 start method，不要在父进程初始化 CUDA 后随意 `fork`。

### 3.5 `asyncio`：协作式 I/O 并发

`asyncio` 用事件循环调度协程。协程只会在 `await` 处主动让出控制权：

```python
import asyncio

async def worker(name: str, delay: float) -> str: #调用 worker() 先返回一个协程对象再执行代码
    await asyncio.sleep(delay)  # 模拟非阻塞 I/O
    return name

async def main() -> None:
    results = await asyncio.gather(
        worker("a", 0.2),
        worker("b", 0.1),
    )
    print(results) # "b" 会先完成。gather 会等待所有协程执行完毕，并按照参数传入的顺序将结果打包成一个列表返回，即 ['a', 'b']。

asyncio.run(main())
```

在事件循环中直接执行阻塞 I/O 或长时间 CPU 计算会卡住所有协程。此时应改用异步库，或把阻塞工作放到线程/进程执行器。

### 3.6 正确做基准测试

性能测试最常见的错误是只测一次、把初始化算进去，或者忘记 GPU 异步执行。

纯 CPU 小片段可用 `timeit`, 底层自动处理了循环次数的动态调整，并会在测试期间暂时关闭垃圾回收器（Garbage Collector），防止 GC 介入干扰计时：

```bash
python -m timeit -s 'xs = list(range(1000))' 'sum(xs)'
```

手工测试至少要做到（预热可以促使 CPU 将相关数据和指令加载到 L1/L2 缓存中，触发操作系统的缺页中断以分配物理内存，或者触发底层 JIT 编译器的即时编译。跳过预热会导致首次执行时间呈现出极端的假性长尾）：

```python
from statistics import median
from time import perf_counter

def benchmark(fn, warmup: int = 5, repeat: int = 20) -> float:
    for _ in range(warmup):
        fn()

    samples = []
    for _ in range(repeat):
        start = perf_counter()
        fn()
        samples.append(perf_counter() - start)
    return median(samples)
```

对于 GPU，Kernel 启动通常是异步的。CPU 计时器只量到“提交任务”的时间，需要在边界同步，或使用框架提供的 GPU Event。同步会改变流水线行为，因此端到端吞吐和单算子延迟应分别测量。

### 3.7 从“慢”定位到具体代码

建议按由粗到细的顺序：

1. **端到端计时**：确认回归真实存在；
2. **阶段计时**：区分数据、计算、通信和保存；
3. **函数级 profiling**：用 `cProfile` 找累计耗时高的函数；
4. **行级/采样 profiling**：进一步定位热点行或原生调用栈；
5. **内存 profiling**：用 `tracemalloc` 等工具找 Python 分配增长；
6. **GPU profiling**：进入 PyTorch Profiler、Nsight Systems/Compute 等专用工具。

```bash
python -m cProfile -o profile.out train.py
python -m pstats profile.out
```

在 `pstats` 中可按累计时间排序：

```text
sort cumulative
stats 30
```

检查 Python 内存分配：

```python
import tracemalloc

tracemalloc.start()
# 运行待检查代码
snapshot = tracemalloc.take_snapshot()
for stat in snapshot.statistics("lineno")[:10]:
    print(stat)
```

### 3.8 Python 性能优化的优先级

推荐顺序如下：

1. 先改算法复杂度和数据结构；
2. 减少不必要的 I/O、复制和序列化；
3. 把细粒度 Python 循环改成批处理/向量化原生算子；
4. 减少 Python 与 C++/GPU 边界的频繁往返；
5. 确认热点后，再写 C++、CUDA 或 Triton 扩展。

例如，一百万次小算子调用即使每次都很快，也会累积大量调度开销。把数据组成批次后调用一次大算子，通常比微调 Python 语法有效得多。
