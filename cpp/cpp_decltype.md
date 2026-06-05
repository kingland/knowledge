## Lambda with `decltype` + `std::decay_t` Pattern

This is a **generic lambda** that inspects its argument's type at compile time.

---

### Breaking it down piece by piece

```cpp
(const auto& val) -> std::string {
    using T = std::decay_t<decltype(val)>;
    // ...
}
```

**`const auto& val`**
A generic lambda parameter — `auto` makes the lambda templated, so `val` can accept any type. The `const&` avoids copying.

---

**`decltype(val)`**
Yields the *declared type* of `val` as the compiler sees it. Since `val` is `const auto&`, `decltype(val)` gives you something like `const int&`, `const std::string&`, etc. — with the reference and const qualifiers still attached.

---

**`std::decay_t<...>`**
Strips away all the "noise" from the type:

| Transformation | Example |
|---|---|
| Removes `const` / `volatile` | `const int` → `int` |
| Removes references | `const int&` → `int` |
| Array → pointer | `int[3]` → `int*` |
| Function → pointer | `void()` → `void(*)()` |

So `std::decay_t<const int&>` → `int`. You get the **bare, clean type**.

---

**`using T = ...`**
Creates a local type alias. Now `T` is the clean underlying type, usable in the rest of the lambda body for `if constexpr`, `std::is_same_v`, etc.

---

### Typical usage pattern

```cpp
auto describe = [](const auto& val) -> std::string {
    using T = std::decay_t<decltype(val)>;

    if constexpr (std::is_same_v<T, int>)
        return "int: " + std::to_string(val);
    else if constexpr (std::is_same_v<T, std::string>)
        return "string: " + val;
    else if constexpr (std::is_floating_point_v<T>)
        return "float: " + std::to_string(val);
    else
        return "unknown type";
};

describe(42);          // "int: 42"
describe(3.14);        // "float: 3.140000"
describe(std::string{"hi"});  // "string: hi"
```

---

### Why not just use `decltype(val)` directly?

Because `std::is_same_v<decltype(val), int>` would be **false** for an `int` argument — the actual type would be `const int&`, not `int`. `decay_t` normalizes it so comparisons work cleanly.

This pattern is the idiomatic way to do **compile-time type dispatch** inside generic lambdas in C++17 and later.