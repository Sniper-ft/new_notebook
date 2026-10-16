---
title: C/C++：内存、生命周期与资源管理
tags:
  - C/C++
categories:
  - 干正事
math: true
---
## 1 值、地址、指针与引用

```cpp
#include <iostream>

int main() {
    int value = 42;
    int* ptr = &value;  // ptr 保存 value 的地址
    int& ref = value;   // ref 是 value 的别名

    *ptr = 7;           // 解引用指针并写入
    ref += 1;

    std::cout << value << '\n';  // 8
}
```

可以先用这张表建立直觉：

| 概念                | 能否为空               | 能否改指向     | 常见用途                             |
| ------------------- | ---------------------- | -------------- | ------------------------------------ |
| 值 `T`              | 否                     | 不适用         | 拥有一个独立对象                     |
| 指针 `T*`           | 可以是 `nullptr`       | 可以           | 可选对象、数组、底层接口、非拥有观察 |
| 引用 `T&`           | 语言语义上必须绑定对象 | 不可以重新绑定 | 必须存在的别名、函数参数             |
| 常量引用 `const T&` | 必须绑定对象           | 不可以         | 避免复制且不允许修改                 |

函数参数表达的接口意图应尽可能清晰：

```cpp
void consume(T value);          // 按值：函数获得自己的对象
void update(T& value);          // 可变借用：函数会修改调用方对象
void inspect(const T& value);   // 只读借用：避免复制
void optional(T* value);        // 指针可能为空，通常不拥有对象
```

对小型标量（`int`、`float`、指针）按值传递通常最清楚；不要机械地把所有参数都改成引用。

## 2 数组、指针与连续内存

原生数组在大多数表达式中会退化为首元素指针，因此单独的 `T*` 不携带长度：

```cpp
void scale(float* data, std::size_t size, float factor) {
    for (std::size_t i = 0; i < size; ++i) {
        data[i] *= factor;
    }
}
```

现代 C++ 中优先用能表达长度的类型：

```cpp
#include <span>

void scale(std::span<float> values, float factor) {
    for (float& value : values) {
        value *= factor;
    }
}
```

`std::span` 不拥有数据，只是“指针 + 长度”的轻量视图。调用期间底层内存必须仍然有效。

二维矩阵通常线性存放。行主序矩阵 `A[M][N]` 的 `(row, col)` 对应偏移：

$$
\text{offset} = row \times N + col
$$

这个公式会在 CUDA 中反复出现。访问顺序是否连续，直接影响 CPU cache 和 GPU 合并访存效率。

## 3 对象存储期与生命周期

常见存储位置的说法是“栈”和“堆”，但更准确地应关注**对象的存储期和所有权**：

```cpp
void example() {
    int local = 1;                  // 自动存储期，离开作用域销毁
    auto buffer = new float[1024];  // 动态分配，必须显式 delete[]
    delete[] buffer;
}
```

手工 `new`/`delete` 容易产生：

- 内存泄漏：忘记释放；
- 重复释放：同一地址释放两次；
- 悬空指针：对象已销毁，指针仍被使用；
- 分配/释放形式不匹配：`new[]` 配了 `delete`；
- 异常路径泄漏：释放前函数提前退出。

因此现代 C++ 的默认原则是：**让资源由对象管理，让对象由作用域管理。**

## 4 RAII：把资源绑定到对象生命周期

RAII（Resource Acquisition Is Initialization）在构造时获取资源，在析构时释放资源：

```cpp
#include <cstdio>
#include <stdexcept>

class File {
public:
    explicit File(const char* path)
        : handle_(std::fopen(path, "rb")) {
        if (handle_ == nullptr) {
            throw std::runtime_error("failed to open file");
        }
    }

    ~File() {
        std::fclose(handle_);
    }

    File(const File&) = delete;
    File& operator=(const File&) = delete;

private:
    std::FILE* handle_;
};
```

无论函数正常返回还是抛出异常，局部对象的析构函数都会执行。文件、锁、主机内存、GPU 内存、CUDA Stream 都可以用同一思想管理。

实际项目中优先使用标准库已有的 RAII 类型，如 `std::vector`、`std::string`、`std::fstream`、`std::lock_guard` 和智能指针，不要重复造轮子。

## 5 智能指针与所有权

```cpp
#include <memory>

auto unique = std::make_unique<Buffer>(1024);  // 唯一所有权
auto shared = std::make_shared<Buffer>(1024);  // 共享所有权
```

| 类型                 | 语义                                                                        | 成本与注意点                                                                                                                        |
| -------------------- | --------------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------- |
| `std::unique_ptr<T>` | 只能有一个 unique_ptr 指向给定内存地址，unique_ptr 离开作用域内存被自动释放 | 成本接近裸指针，默认首选                                                                                                            |
| `std::shared_ptr<T>` | 允许多个智能指针指向同一个对象看，通过引用计数管理生命周期                  | 有计数与控制块（强引用计数记录有多少个 shared_ptr 共享该对象、弱引用计数供 weak_ptr 使用以及自定义删除器等），循环引用需 `weak_ptr` |
| `std::weak_ptr<T>`   | 不增加引用计数的观察者                                                      | 使用前要 `lock()` 检查对象是否仍存在                                                                                                |
| `T*` / `T&`          | 通常表示非拥有访问                                                          | 被指向对象必须活得足够久                                                                                                            |

如果所有权没有共享需求，不要使用 `shared_ptr`。“避免思考所有权”不是共享所有权的合理理由。

## 6 拷贝、移动与 Rule of Zero

`std::vector` 等容器拥有资源，复制会复制内容，移动则转移内部资源：

```cpp
#include <utility>
#include <vector>

std::vector<float> make_buffer() {
    std::vector<float> values(1'000'000, 0.0f);
    return values;  // 编译器通常消除复制，必要时可移动
}

auto a = make_buffer();
auto b = std::move(a);  // a 仍有效，但内容处于未指定状态
```

如果类只由标准库 RAII 成员组成，通常无需手写析构、复制和移动操作，这就是 **Rule of Zero**。只有直接管理特殊资源时才需要自定义这些操作，并清晰定义所有权。

不要随手对返回值写 `std::move`；它可能阻止返回值优化。也不要在移动后假设源对象仍保留原内容。

## 7 `const`：把不可修改写进接口

```cpp
const float* p1;        // 指向 const float：不能通过 p1 改值
float* const p2 = ptr;  // const 指针：不能让 p2 改指向
const float* const p3 = ptr;
```

成员函数末尾的 `const` 表示不修改对象的可观察状态：

```cpp
class TensorView {
public:
    std::size_t size() const { return size_; }
private:
    std::size_t size_{};
};
```

`const` 有助于编译器和读者验证接口，但不能自动解决并发安全与底层别名问题。

## 8 未定义行为为什么危险

越界访问、使用悬空引用、有符号整数溢出、数据竞争等可能触发未定义行为（Undefined Behavior，UB）。编译器可以假设 UB 永不发生，并据此优化；程序结果不一定只是“报错”，也可能在调试版正常、优化版悄悄算错。

```cpp
std::vector<int> values(4);
values[4] = 1;  // 越界，未定义行为
```

调试时可启用警告和 Sanitizer：

```bash
g++ -std=c++20 -Wall -Wextra -Wpedantic \
    -fsanitize=address,undefined -g main.cpp -o app
./app
```

AddressSanitizer 常用于发现越界、use-after-free；UndefinedBehaviorSanitizer 检查多类 UB；ThreadSanitizer 可定位数据竞争，但通常不能和 AddressSanitizer 同时启用。
