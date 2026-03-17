## `bool` in C++

### Basics

`bool` is a built-in type that holds exactly two values: `true` or `false`.

```cpp
bool a = true;
bool b = false;
```

---

### Size & Representation

```cpp
sizeof(bool)  // Implementation-defined, typically 1 byte
```

Internally stored as an integer: `true` → `1`, `false` → `0`.

---

### Implicit Conversions

**Arithmetic → bool** (any non-zero → `true`, zero → `false`):
```cpp
bool x = 42;      // true
bool y = 0;       // false
bool z = -1;      // true
bool p = nullptr; // false
bool q = 3.14;    // true
```

**bool → Arithmetic** (`true` → `1`, `false` → `0`):
```cpp
int n = true + true;  // 2
int m = false * 100;  // 0
```

This is often used for counting:
```cpp
int count = (a > 0) + (b > 0) + (c > 0);
```

---

### Logical Operators

| Operator | Meaning | Example |
|---|---|---|
| `&&` | AND | `a && b` |
| `\|\|` | OR | `a \|\| b` |
| `!` | NOT | `!a` |

**Short-circuit evaluation** — the right side is not evaluated if the result is already determined:
```cpp
if (ptr != nullptr && ptr->value > 0)  // safe: ptr is checked first
```

---

### Comparison Operators

All return `bool`:
```cpp
==  !=  <  >  <=  >=
```

In C++20, the **spaceship operator** `<=>` was added and generates comparisons automatically.

---

### Common Pitfalls

**1. Assignment inside condition (accidental `=` instead of `==`)**
```cpp
if (x = true)  // always true! assigns, not compares
if (x == true) // correct comparison
if (x)         // idiomatic — best style
```

**2. Comparing bool to `true`/`false` is redundant**
```cpp
if (isValid == true)   // redundant
if (isValid)           // idiomatic ✓

if (isValid == false)  // redundant
if (!isValid)          // idiomatic ✓
```

**3. Integer overflow doesn't apply, but be careful with `++`**
```cpp
bool b = false;
b++;   // b == true (deprecated in C++17, removed in C++20 ❌)
```

**4. Unsigned arithmetic surprise**
```cpp
bool a = true;   // 1
bool b = true;   // 1
auto c = a - b;  // 0, type is int, not bool
```

---

### `std::boolalpha` for I/O

```cpp
std::cout << true;                        // prints: 1
std::cout << std::boolalpha << true;      // prints: true
std::cout << std::noboolalpha << true;    // prints: 1
```

---

### In Structs / Classes

```cpp
struct Config {
    bool enableLogging = false;
    bool verbose       = false;
    bool dryRun        = true;
};
```

> **Bit-field packing tip:** if you have many flags, you can use `bool : 1` bit-fields or `std::bitset<N>` to save memory, though access is slower.

---

### Core Guidelines Reminders

- Prefer `bool` over `int` for flags — communicates intent clearly.
- Never test `if (func() == true)` — just `if (func())`.
- Avoid implicit bool conversions from pointers in complex expressions — be explicit with `!= nullptr`.