## `std::any` in C++

`std::any` (C++17) is a type-safe container for single values of **any copy-constructible type** — essentially a type-erased value holder, similar to `void*` but with runtime type safety.

---

### Core Idea

```cpp
#include <any>

std::any a = 42;          // holds int
a = std::string("hello"); // now holds string
a = 3.14;                 // now holds double
```

The container tracks both the **value** and its **type** at runtime.

---

### Key Operations

```cpp
#include <any>
#include <iostream>
#include <string>

int main() {
    std::any a = 42;

    // 1. Check if it holds a value
    if (a.has_value())
        std::cout << "has value\n";

    // 2. Query the type
    std::cout << a.type().name() << "\n"; // "i" (int) on GCC

    // 3. Cast to the correct type (throws std::bad_any_cast if wrong)
    int x = std::any_cast<int>(a);

    // 4. Safe cast via pointer (returns nullptr on failure, never throws)
    if (auto* p = std::any_cast<int>(&a))
        std::cout << *p << "\n";

    // 5. Reset
    a.reset();
    std::cout << a.has_value() << "\n"; // false
}
```

---

### `std::any_cast` — Three Forms

| Form | Behavior on wrong type |
|---|---|
| `std::any_cast<T>(a)` | Throws `std::bad_any_cast` |
| `std::any_cast<T>(&a)` | Returns `nullptr` |
| `std::any_cast<T>(std::move(a))` | Moves the value out (throws on failure) |

---

### Construction Patterns

```cpp
// Direct assignment
std::any a = std::string("hi");

// std::make_any<T> — preferred for complex types
auto b = std::make_any<std::vector<int>>({1, 2, 3});

// In-place construction (avoids extra copy)
std::any c(std::in_place_type<std::string>, 5, 'x'); // "xxxxx"
```

---

### Practical Use Cases

**1. Heterogeneous containers**
```cpp
std::vector<std::any> bag = {42, 3.14, std::string("hello"), true};
for (auto& item : bag) {
    if (auto* i = std::any_cast<int>(&item))       std::cout << "int: "    << *i << "\n";
    else if (auto* d = std::any_cast<double>(&item)) std::cout << "double: " << *d << "\n";
    else if (auto* s = std::any_cast<std::string>(&item)) std::cout << "str: " << *s << "\n";
}
```

**2. Generic configuration / property maps**
```cpp
std::unordered_map<std::string, std::any> config;
config["timeout"]  = 30;
config["host"]     = std::string("localhost");
config["debug"]    = true;

int timeout = std::any_cast<int>(config["timeout"]);
```

**3. Type-erased callback arguments**
```cpp
void process(std::any ctx) {
    if (auto* p = std::any_cast<std::string>(&ctx))
        std::cout << "string context: " << *p << "\n";
}
```

---

### Internals & Performance Notes

- Uses **Small Buffer Optimization (SBO)**: small types (typically ≤ 16 bytes) are stored inline — no heap allocation.
- Larger types trigger a **heap allocation**.
- Copying `std::any` copies the underlying value — can be expensive for large objects.
- Prefer `std::any_cast<T&>` to avoid unnecessary copies when the value is large.

---

### Comparison with Alternatives

| | `std::any` | `std::variant` | `void*` |
|---|---|---|---|
| Type set | Open (any type) | Closed (fixed set) | Open |
| Type safety | ✅ Runtime | ✅ Compile-time | ❌ None |
| Overhead | Higher (heap possible) | Lower (stack) | Minimal |
| Best for | Unknown types at compile time | Known set of types | C interop only |

---

### Common Pitfall

```cpp
std::any a = "hello";                    // holds const char*, NOT std::string!
auto s = std::any_cast<std::string>(a);  // throws bad_any_cast

// Fix:
std::any b = std::string("hello");       // explicit type
```

---

**Rule of thumb**: prefer `std::variant` when the set of types is known at compile time — it's faster and safer. Use `std::any` when the type is genuinely unknown until runtime (plugin systems, scripting bridges, dynamic configs).