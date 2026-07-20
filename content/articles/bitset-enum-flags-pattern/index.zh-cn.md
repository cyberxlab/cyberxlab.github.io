---
title: "为 C++ 枚举类设计类型安全位标志"
date: 2026-07-20T04:06:50Z
draft: false
tags: ["C++", "enum-class", "bit-flags", "type-safety"]
categories: ["技术实践"]
author: "X 实验室"
description: "一套经过实战检验的模式：在保留 enum class 类型安全的同时，恢复位运算组合的工程体验。"
summary: "从 scoped enum 基础到通用 BitFlags 模板，本文展示如何兼顾类型安全与可审计的位标志操作，避免跨枚举误用与运算符优先级陷阱。"
toc: true
---

## 背景与动机

位标志无处不在：文件权限、渲染通道、GPU 能力掩码、报文头部位、网络能力集合。C 语言的经典写法是声明一个 unscoped `enum`，成员赋值为 2 的幂，然后依靠隐式 `int` 转换去组合：

```cpp
enum Permission { Read = 1, Write = 2, Execute = 4 };
int p = Read | Write;     // 编译通过，但名字泄漏到外层作用域
int leaks_unsafe_value = 7;
```

这种写法方便，但会掩盖两类反复出现的 bug：

1. **作用域上提**——每个 enumerator 都会成为外层作用域中的名字，导致 `File::Read` 与 `Stream::Read` 冲突，错误信息难读。
2. **类型安全丢失**——位运算返回 `int`，不再携带 enum 类型。不同 enum 之间的位标志会被悄悄相容，`File::Read | Network::Readable` 可以编译。

C++11 引入 `enum class`，要求访问时写完整名字（`File::Permission::Read`），并禁止向 `int` 的隐式转换，从根上解决了问题 (1)。代价是 `enum class` 不再原生支持 `operator|`、`&`、`^`、`~`，必须自己实现。多出来的样板让许多项目退回到 unscoped enum，继续吞噬问题 (2)。

本文的目标是给出一套可复用、类型安全、可审计的模式，让位标志：

- 保留 `enum class` 的命名空间卫生。
- 恢复组合习惯用法（`a | b`、`flags & a`、`flags &= ~a`）。
- 在类型层面区分"单个 enumerator"和"一组 flags"。
- 不再为每个 enum 重复一堆 operator。

文中所有片段在 C++20/23 下均可编译（BitFlags 模板最低要求 C++17）。

## 前置条件

- C++17 及以上工具链（gcc 9+、clang 10+、MSVC 19.20+）。C++23 示例另外用到 `<utility>` 的 `std::to_underlying`。
- `<type_traits>` 提供 `std::underlying_type_t` 和 `std::is_enum_v`。
- `<initializer_list>` 用于花括号构造。
- 熟悉 `enum class` 语法与基础模板元编程。
- 可选：单元测试框架（GoogleTest、Catch2）用于验证步骤。

## 步骤一：准备工作

从最简洁的 enum 定义开始。务必把底层类型固定为定宽无符号整型，使位布局跨平台一致、可以直接序列化到线上协议：

```cpp
#include <cstdint>
#include <type_traits>

namespace fs {

enum class Permission : std::uint8_t {
    None      = 0,
    Read      = 1 << 0,
    Write     = 1 << 1,
    Execute   = 1 << 2,
    // 组合可以命名成 alias。
    ReadWrite = Read | Write,
    All       = Read | Write | Execute,
};

}  // namespace fs
```

这里做了几个约定：

- 必然存在 `None = 0`。否则调用方只能凭空造一个 0 来表示"没有 flag"。
- 取值用 `1 << n`，而不是直接写数字字面量。位移本身就标注了 bit 宽度，并防止重排 enumerator 时漏掉/错位。
- 组合 alias（如 `ReadWrite`）按需添加，类型仍然是 `Permission`，但仍只是单个值。建议少加，把组合放到 BitFlags 包装里更合适。

不在 `Permission` 上直接定义 operator。这是接下来模式的关键前提。

## 步骤二：核心实现

我们采用 Anthony Williams 提出、后由 Voithos 与 Andreas Fertig 改良的 BitFlags 模板。包装类型 `BitFlags<T>` 持有一组 flag 位，`T` 仍然是单值的 enum。类型系统现在能区分"一个权限"与"一组权限"——这正是直接给 enum 重载 operator 时丢失的区别。

```cpp
#include <bitset>
#include <initializer_list>
#include <iosfwd>
#include <type_traits>
#include <utility>

template <typename T>
class BitFlags {
    static_assert(std::is_enum_v<T>, "BitFlags requires an enum type");
    using UnderlyingT = std::underlying_type_t<T>;

   public:
    constexpr BitFlags() noexcept : flags_(0) {}
    constexpr explicit BitFlags(T v) noexcept : flags_(ToUnderlying(v)) {}

    constexpr BitFlags(std::initializer_list<T> vs) noexcept : BitFlags() {
        for (T v : vs) flags_ |= ToUnderlying(v);
    }

    [[nodiscard]] constexpr bool IsSet(T v) const noexcept {
        return (flags_ & ToUnderlying(v)) == ToUnderlying(v);
    }
    constexpr void Set(T v) noexcept       { flags_ |= ToUnderlying(v); }
    constexpr void Unset(T v) noexcept     { flags_ &= ~ToUnderlying(v); }
    constexpr void Clear() noexcept        { flags_ = 0; }
    constexpr void Toggle(T v) noexcept    { flags_ ^= ToUnderlying(v); }

    constexpr explicit operator bool() const noexcept { return flags_ != 0; }

    friend constexpr BitFlags operator|(BitFlags a, BitFlags b) noexcept {
        return BitFlags{a.flags_ | b.flags_};
    }
    friend constexpr BitFlags operator&(BitFlags a, BitFlags b) noexcept {
        return BitFlags{a.flags_ & b.flags_};
    }
    friend constexpr BitFlags operator^(BitFlags a, BitFlags b) noexcept {
        return BitFlags{a.flags_ ^ b.flags_};
    }
    friend constexpr BitFlags operator~(BitFlags a) noexcept {
        return BitFlags{static_cast<UnderlyingT>(~a.flags_)};
    }

    friend constexpr BitFlags& operator|=(BitFlags& a, BitFlags b) noexcept {
        a.flags_ |= b.flags_; return a;
    }
    friend constexpr BitFlags& operator&=(BitFlags a, BitFlags b) noexcept {
        a.flags_ &= b.flags_; return a;
    }
    friend constexpr BitFlags& operator^=(BitFlags& a, BitFlags b) noexcept {
        a.flags_ ^= b.flags_; return a;
    }

    // 适配那些仍期望 int 的老接口或第三方 API。
    [[nodiscard]] static constexpr BitFlags FromRaw(UnderlyingT raw) noexcept {
        return BitFlags{raw};
    }
    [[nodiscard]] constexpr UnderlyingT ToRaw() const noexcept { return flags_; }

    friend std::ostream& operator<<(std::ostream& os, BitFlags bf) {
        return os << std::bitset<sizeof(UnderlyingT) * 8>(bf.flags_);
    }

    friend constexpr bool operator==(BitFlags a, BitFlags b) noexcept {
        return a.flags_ == b.flags_;
    }
    friend constexpr bool operator!=(BitFlags a, BitFlags b) noexcept {
        return a.flags_ != b.flags_;
    }

   private:
    constexpr explicit BitFlags(UnderlyingT raw) noexcept : flags_(raw) {}
    static constexpr UnderlyingT ToUnderlying(T v) noexcept {
#if __cplusplus >= 202302L
        return std::to_underlying(v);
#else
        return static_cast<UnderlyingT>(v);
#endif
    }
    UnderlyingT flags_;
};
```

再加一个 deduction guide 调用起来更顺手：

```cpp
template <typename T>
BitFlags(T) -> BitFlags<T>;
template <typename T>
BitFlags(std::initializer_list<T>) -> BitFlags<T>;
```

一个文件 API 的真实调用大致这样：

```cpp
#include <iostream>
namespace fs { /* Permission + BitFlags 同上 */ }

int main() {
    using namespace fs;

    BitFlags<Permission> ro{Permission::Read};
    BitFlags<Permission> rw = {Permission::Read, Permission::Write};

    // 组合集合。
    auto all = rw | BitFlags<Permission>{Permission::Execute};

    // 测试单个 flag。
    std::cout << "read set? " << std::boolalpha << all.IsSet(Permission::Read)  // true
              << "\nexec set? " << all.IsSet(Permission::Execute);              // true

    // 切换与清除。
    all.Toggle(Permission::Read);
    all.Unset(Permission::Write);

    if (all) { /* 还有任意 flag 被置位 */ }

    std::cout << "\nraw=" << all.ToRaw() << " bits=" << all << '\n';

    // 错误：下面这行不会编译——这正是模式要解决的事。
    // BitFlags<Permission> bad = OpenMode::Binary;
}
```

值得展开的属性：

- `BitFlags<Permission>` 与 `BitFlags<OpenMode>` 是不同的类型。跨 enum 组合无法编译——不存在悄无声息的混用。
- `Permission` 本身永远不会接受 `|` 组合，enum 始终代表单个值。
- 容器友好（`initializer_list`）、可序列化（`ToRaw`/`FromRaw`）、可打印（`std::bitset` 视图）。
- 每个函数都是 `constexpr`，可以在常量上下文里使用，比如 `constexpr BitFlags<X> table[]`。

## 步骤三：验证与调优

在仓库里推行该模式前，先写一个最小冒烟测试把契约锁死：

```cpp
#include <cassert>

int main() {
    using P = fs::Permission;
    fs::BitFlags<P> empty;
    assert(!empty);
    assert(empty.ToRaw() == 0);

    fs::BitFlags<P> rw{P::Read, P::Write};
    assert(rw.IsSet(P::Read) && rw.IsSet(P::Write));
    assert(!rw.IsSet(P::Execute));

    auto all = rw | fs::BitFlags<P>{P::Execute};
    assert(all.IsSet(P::Execute));
    assert(all.ToRaw() == 0b0000'0111);

    all.Unset(P::Read);
    assert(!all.IsSet(P::Read));

    // 跨枚举保护：把下一行解开注释必须编译失败。
    // fs::BitFlags<P> bad = net::SocketFlag::NonBlocking;
    return 0;
}
```

下面是几条值得在不同环境里调校的性能与正确性要点：

1. **零开销** —— `BitFlags<T>` 是 trivially copyable 的单整数包装。`static_cast` 与 inline friend operator 在 `-O2` 下与手写 `int` 标志生成相同的汇编。若是热路径，可用 `objdump -d` 验证。
2. **`operator|` 不要返回 `bool`** —— 所有二元 operator 必须返回 `BitFlags`。返回 `bool` 是常见 bug，会让链式表达式（`a | b | c`）失效。
3. **模板里放 `static_assert(std::is_enum_v<T>)`** —— 在声明处就能拦下"传入 int 或 `std::byte`"的误用，而不是后置报错。
4. **C++23 时改用 `std::to_underlying`** —— 比 `static_cast` 更短，意图更明确。
5. **把 `FromRaw`/`ToRaw` 关到序列化边界** —— 这两个函数标记 `static` 与 `explicit`。原始值转换属于 I/O 边界，不要流进业务逻辑里。一次 grep 就能审完所有调用点。

## 最佳实践小结

- **固定底层类型**：写 `enum class X : std::uint8_t { ... }` 让位布局可移植、可直接 JSON 序列化，并对编译器/ABI 稳定。
- **定义 `None = 0`**：任何 flag enum 都需要一个"无 flag"值。它比想象的更有用，是默认构造与无值表达之间的唯一桥梁。
- **区分 enum 单值与 BitFlags 集合**：让 enum 负责"命名"，`BitFlags<T>` 负责"表示"。既能避开 `operator&` 优先级陷阱，也避免 `(flags & Perm::Read) == Perm::Read` 在文档里堆叠成 `(flags & Perm::Read) == 0`。
- **仅把 `FromRaw`/`ToRaw` 用于序列化边界**：标 `static` 与 `explicit`，让审计变成一次 grep，而不是遍布业务逻辑。
- **用 trait 显式 opt-in 而不是 `using namespace`**：为兼容老代码而保留基于 operator 的写法时，使用 `enable_bitmask_operator(Permission);` 这类 ADL 友好的 trait（参考 Andreas Fertig 的 `requires` 表达式），新 enum 类型只需多一行而不是一堵宏墙。

## 参考资料

- Voithos — Type-safe Enum Class Bit Flags：
  https://voithos.io/articles/type-safe-enum-class-bit-flags/
- Andreas Fertig — C++20 Concepts applied（基于 scoped enum 的位掩码）：
  https://andreasfertig.com/blog/2024/01/cpp20-concepts-applied/
- ortogonal — Scoped enums 与位标志/位模式：
  https://ortogonal.github.io/cpp/enum-class-bit-flags-pattern/
- Aaron Bos — C# 中将枚举用作位标志（跨语言对照）：
  https://aaronbos.dev/posts/csharp-flags-enum
- elbeno — A better bitset for enum flags（批评意见与 BitFlags 替代方向）：
  https://www.elbeno.com/blog/?p=1836
- Stack Overflow — How to use C++11 enum class for flags（讨论串）：
  https://stackoverflow.com/questions/32578638/how-to-use-c11-enum-class-for-flags
