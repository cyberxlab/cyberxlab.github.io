---
title: "Type-Safe Bit Flags for C++ Enum Classes"
date: 2026-07-20T04:06:50Z
draft: false
tags: ["C++", "enum-class", "bit-flags", "type-safety"]
categories: ["Technical Practices"]
author: "Cyber·X·Lab"
description: "A field-tested pattern for combining the type safety of enum class with the ergonomics of bitwise flag manipulation."
summary: "From scoped enum foundation to a generic BitFlags template, this guide shows how to keep type safety while making C++ bit-flag enums ergonomic and easy to audit."
toc: true
---

## Background and Motivation

Bit flags are everywhere: file permissions, render passes, GPU feature masks, message-header options, network capability sets. The classical C idiom is to declare an unscoped `enum` with power-of-two members and rely on implicit `int` conversion to combine them:

```cpp
enum Permission { Read = 1, Write = 2, Execute = 4 };
int p = Read | Write;     // works, leaks into the surrounding scope
int leaks_unsafe_value = 7;
```

This syntactic convenience masks two recurring bugs:

1. **Scope hoisting** — every enumerator becomes a name in the enclosing namespace, leading to collisions (`File::Read` vs `Stream::Read`) and unreadable error messages.
2. **Lost type safety** — an `int` returned from the flag combination no longer carries the enum's type. Mixing flags from different enums compiles silently: `File::Read | Network::Readable` is permitted.

C++11 introduced `enum class` (scoped enumerations) which solve problem (1) by requiring `File::Permission::Read` accessors and removing implicit conversion to `int`. The catch: `enum class` no longer supports `operator|`, `&`, `^`, `~` out of the box. You must write the operators yourself. That overhead pushed many codebases back to unscoped enums, perpetuating problem (2).

The engineering goal of this article is to show a reusable, type-safe, audit-friendly pattern for bit-flag enums that:

- Keeps the namespace hygiene of `enum class`.
- Restores ergonomic bitwise combination (`a | b`, `flags & a`, `flags &= ~a`).
- Makes the difference between a single enumerator and a set of flags explicit in the type system.
- Avoids boilerplate per-enum operator definitions.

All snippets below compile under C++20/23 (the BitFlags template needs C++17 at minimum).

## Prerequisites

- A C++17-or-later toolchain (gcc 9+, clang 10+, MSVC 19.20+). The C++23 examples additionally use `<utility>`'s `std::to_underlying`.
- `<type_traits>` for `std::underlying_type_t` and `std::is_enum_v`.
- `<initializer_list>` for brace-enclosed flag construction.
- Familiarity with `enum class` syntax and template metaprogramming basics.
- Optional: a unit-test harness (GoogleTest, Catch2) for the verification step.

## Step 1: Preparation

Start from the simplest sane enum definition. Always pin the underlying type to a fixed-width unsigned integer so the bit layout is predictable across platforms and so the enum can serialize straight to a wire format.

```cpp
#include <cstdint>
#include <type_traits>

namespace fs {

enum class Permission : std::uint8_t {
    None      = 0,
    Read      = 1 << 0,
    Write     = 1 << 1,
    Execute   = 1 << 2,
    // Combinations can be defined as named aliases.
    ReadWrite = Read | Write,
    All       = Read | Write | Execute,
};

}  // namespace fs
```

Decisions baked in here:

- `None = 0` exists. Without it, callers cannot represent "no flags set" without conjuring a zero magic number.
- Values are assigned with `1 << n`, not raw literals. The shift documents bit width and protects against the swizzling you get when reordering enumerators while leaving numeric literals behind.
- Combination aliases like `ReadWrite` are convenience constants, still typed as `Permission`. Use sparingly; dominated combinations usually belong in the BitFlags wrapper, not the enum.

No operators are defined on `Permission` itself. That is intentional and central to the pattern that follows.

## Step 2: Core Implementation

The pattern we will adopt is the BitFlags template originally described by Anthony Williams and later refined by Voithos and Andreas Fertig. A wrapper template `BitFlags<T>` carries a set of flag bits while `T` remains a single-value enum. The type system now distinguishes "one permission" from "a set of permissions" — exactly the distinction we lost when overloading operators directly on the enum.

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
    friend constexpr BitFlags& operator&=(BitFlags& a, BitFlags b) noexcept {
        a.flags_ &= b.flags_; return a;
    }
    friend constexpr BitFlags& operator^=(BitFlags& a, BitFlags b) noexcept {
        a.flags_ ^= b.flags_; return a;
    }

    // A bridge type for older or third-party APIs that still expect a plain int.
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

A small deduction guide makes site calls nicer:

```cpp
template <typename T>
BitFlags(T) -> BitFlags<T>;
template <typename T>
BitFlags(std::initializer_list<T>) -> BitFlags<T>;
```

Real usage in a file API now looks like:

```cpp
#include <iostream>
namespace fs { /* Permission + BitFlags above */ }

int main() {
    using namespace fs;

    BitFlags<Permission> ro{Permission::Read};
    BitFlags<Permission> rw = {Permission::Read, Permission::Write};

    // Combine sets.
    auto all = rw | BitFlags<Permission>{Permission::Execute};

    // Test individual flags.
    std::cout << "read set? " << std::boolalpha << all.IsSet(Permission::Read)  // true
              << "\nexec set? " << all.IsSet(Permission::Execute);              // true

    // Toggle and clear.
    all.Toggle(Permission::Read);
    all.Unset(Permission::Write);

    if (all) { /* any flag still set */ }

    std::cout << "\nraw=" << all.ToRaw() << " bits=" << all << '\n';

    // Error: this does not compile, which is the whole point.
    // BitFlags<Permission> bad = OpenMode::Binary;
}
```

Key properties worth highlighting:

- `BitFlags<Permission>` and `BitFlags<OpenMode>` are distinct types. Cross-enum combinations fail to compile — no silent mixing.
- `Permission` alone never silently accepts bitwise combination; the enum carries a single value.
- Container-friendly (`initializer_list`), serializable (`ToRaw`/`FromRaw`), printable (`std::bitset` view).
- `constexpr` everywhere — usable in `constexpr` contexts, including `constexpr BitFlags<X> table[]`.

## Step 3: Verification and Tuning

Build a tiny smoke test to lock in the contract before merging the pattern across a codebase:

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

    // Cross-enum protection: uncommenting the line below must not compile.
    // fs::BitFlags<P> bad = net::SocketFlag::NonBlocking;
    return 0;
}
```

A few performance notes worth tuning for your environment:

1. **Zero overhead** — `BitFlags<T>` is a trivially copyable single-integer wrapper. `static_cast`-based conversions and the inline friend operators generate the same assembly as a hand-written `int` flag set under `-O2`. Verify with `objdump -d` if it matters for hot paths.
2. **Avoid `bool` return on `operator|`** — return `BitFlags` from every binary operator. Returning `bool` is a common bug that disables expression chaining (`a | b | c`).
3. **Use `static_assert(std::is_enum_v<T>)`** in the template body. It catches accidental instantiation with raw ints or with `std::byte` at the declaration site, not in downstream code.
4. **For C++23** prefer `std::to_underlying` over the manual cast. It is shorter and signals intent.
5. **Keep `FromRaw`/`ToRaw` out of common call sites** — make them `static` and `explicit`. Raw conversions are an I/O boundary; do not let them leak into in-process logic. Auditing for them becomes a single grep.

## Best Practices Summary

- **Pin the underlying type**: `enum class X : std::uint8_t { ... }` makes the bit layout portable, JSON-serializable, and ABI-stable across compilers.
- **Define `None = 0`**: every flag enum needs a "no flags set" value. The convenience is larger than it looks; it is the only bridge to default construction.
- **Separate single-value enums from set-of-flags types**: keep the enum as the *naming* layer, wrap with `BitFlags<T>` for the *representation* layer. This avoids the well-documented precedence pitfalls of `operator&` and keeps `(flags & Perm::Read) == Perm::Read` from compounding into `(flags & Perm::Read) == 0`.
- **Reserve `FromRaw`/`ToRaw` for the serialization boundary**: these are correctness escape hatches. Marking them `static` and `explicit` confines them to one audit surface instead of letting them silently hop through business logic.
- **Opt in with a single trait, not a `using namespace`**: if you also provide operator-based enums for legacy code, gate them behind `enable_bitmask_operator(Permission);` (ADL-friendly trait from Andreas Fertig's `requires`-expression approach) so new enum types are explorable with one extra line instead of a macro wall.

## References

- Voithos — Type-safe Enum Class Bit Flags:
  https://voithos.io/articles/type-safe-enum-class-bit-flags/
- Andreas Fertig — C++20 Concepts applied (Safe bitmasks using scoped enums):
  https://andreasfertig.com/blog/2024/01/cpp20-concepts-applied/
- ortogonal — Scoped enums together with bit flags / patterns:
  https://ortogonal.github.io/cpp/enum-class-bit-flags-pattern/
- Aaron Bos — Defining and Using Enums as Bit Flags in C# (cross-language analogue):
  https://aaronbos.dev/posts/csharp-flags-enum
- elbeno — A better bitset for enum flags (critique and BitFlags alternative direction):
  https://www.elbeno.com/blog/?p=1836
- Stack Overflow — How to use C++11 enum class for flags (thread survey):
  https://stackoverflow.com/questions/32578638/how-to-use-c11-enum-class-for-flags
