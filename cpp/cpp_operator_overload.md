# Operator Overloading in C++ — Deep Dive

## What Is Operator Overloading?

Operator overloading lets you define custom behavior for built-in operators (`+`, `==`, `<<`, `[]`, etc.) when applied to user-defined types. It's syntactic sugar — `a + b` becomes a function call under the hood.

---

## Syntax Forms

Every operator can be overloaded as either a **member function** or a **free (non-member) function**:

```cpp
// Member function: left operand is *this
T operator+(const T& rhs) const;

// Free function: both operands are explicit
T operator+(const T& lhs, const T& rhs);
```

---

## The Full Operator Taxonomy

### 1. Arithmetic Operators: `+` `-` `*` `/` `%`

The canonical pattern uses the **compound assignment as the primitive**, then builds the binary op on top:

```cpp
struct Vec2 {
    float x, y;

    // Compound assignment — modifies *this, returns reference
    Vec2& operator+=(const Vec2& rhs) {
        x += rhs.x;
        y += rhs.y;
        return *this;
    }

    Vec2& operator-=(const Vec2& rhs) { x -= rhs.x; y -= rhs.y; return *this; }
    Vec2& operator*=(float s)          { x *= s;     y *= s;     return *this; }
    Vec2& operator/=(float s)          { x /= s;     y /= s;     return *this; }
};

// Binary ops — free functions, delegate to compound assignment
// Takes lhs by VALUE (copy), mutates it, returns by value
inline Vec2 operator+(Vec2 lhs, const Vec2& rhs) { return lhs += rhs; }
inline Vec2 operator-(Vec2 lhs, const Vec2& rhs) { return lhs -= rhs; }
inline Vec2 operator*(Vec2 v,   float s)          { return v  *= s;   }
inline Vec2 operator*(float s,  Vec2 v)           { return v  *= s;   } // commutative
inline Vec2 operator/(Vec2 v,   float s)          { return v  /= s;   }
```

> **Why take `lhs` by value?** The copy *is* your working object. The compiler can apply NRVO/copy elision. It's more efficient than making a named copy inside.

---

### 2. Comparison Operators

**Pre-C++20:** You define 6 operators manually.  
**C++20:** Define `operator<=>` (spaceship) + `operator==` and get everything for free.

```cpp
#include <compare>

struct Version {
    int major, minor, patch;

    // C++20: one operator gives <, <=, >, >=, and != for free
    auto operator<=>(const Version&) const = default; // memberwise comparison

    // operator== must still be explicit (or also defaulted)
    bool operator==(const Version&) const = default;
};

// Usage:
Version v1{1, 2, 3}, v2{1, 3, 0};
bool older = v1 < v2;   // true — works automatically
```

**Custom spaceship** when memberwise isn't correct:

```cpp
struct CaseString {
    std::string s;

    std::strong_ordering operator<=>(const CaseString& rhs) const {
        // case-insensitive comparison
        std::string a = s, b = rhs.s;
        std::ranges::transform(a, a.begin(), ::tolower);
        std::ranges::transform(b, b.begin(), ::tolower);
        return a <=> b;
    }
    bool operator==(const CaseString& rhs) const {
        return (*this <=> rhs) == std::strong_ordering::equal;
    }
};
```

**Ordering categories:**

| Category | Meaning | Example |
|---|---|---|
| `std::strong_ordering` | Equal means identical | integers |
| `std::weak_ordering` | Equal means equivalent | case-insensitive string |
| `std::partial_ordering` | Some pairs unordered | floats (NaN) |

---

### 3. Stream Operators: `<<` and `>>`

Must be **free functions** (you can't modify `std::ostream`). Declare as `friend` to access private members:

```cpp
struct Complex {
    double re, im;

    friend std::ostream& operator<<(std::ostream& os, const Complex& c) {
        os << c.re;
        if (c.im >= 0) os << '+';
        return os << c.im << 'i';
    }

    friend std::istream& operator>>(std::istream& is, Complex& c) {
        return is >> c.re >> c.im;
    }
};

// Usage:
Complex z{3.0, -4.0};
std::cout << z;          // 3-4i
std::cin  >> z;
```

---

### 4. Subscript Operator: `[]`

**C++23** allows multi-parameter `[]`. Before that, only single parameter:

```cpp
struct Matrix {
    std::vector<double> data;
    size_t rows, cols;

    // C++23: multi-dimensional subscript
    double& operator[](size_t r, size_t c) {
        return data[r * cols + c];
    }
    const double& operator[](size_t r, size_t c) const {
        return data[r * cols + c];
    }

    // C++20 and earlier: proxy object pattern for m[r][c]
    // (returns a row proxy with its own operator[])
};
```

For **checked access** vs **unchecked**, the convention is:
- `operator[]` — unchecked, fast
- `.at()` — throws `std::out_of_range`

---

### 5. Call Operator: `operator()` — Functors

Makes an object callable. This is the foundation of lambdas:

```cpp
struct Adder {
    int base;
    explicit Adder(int b) : base(b) {}

    int operator()(int x) const { return base + x; }
};

Adder add5{5};
add5(3);   // 8

// Equivalent lambda:
auto add5l = [base = 5](int x) { return base + x; };
```

**Generic call operator (C++20 concepts):**

```cpp
struct Clamp {
    template<typename T>
    requires std::totally_ordered<T>
    T operator()(T val, T lo, T hi) const {
        return std::clamp(val, lo, hi);
    }
};
```

---

### 6. Dereference and Arrow: `*` and `->`

Essential for **smart pointer / iterator** types:

```cpp
template<typename T>
class ObserverPtr {
    T* ptr;
public:
    explicit ObserverPtr(T* p) : ptr(p) {}

    T& operator*()  const { return *ptr; }   // dereference
    T* operator->() const { return ptr;  }   // member access
    explicit operator bool() const { return ptr != nullptr; }
};

// operator-> must return a raw pointer (or another type with ->)
// The compiler keeps applying -> until it gets a raw pointer
```

---

### 7. Increment / Decrement: `++` `--`

The **dummy `int`** parameter distinguishes postfix from prefix:

```cpp
struct Iterator {
    int pos;

    // Prefix: ++it — modify and return reference
    Iterator& operator++() {
        ++pos;
        return *this;
    }

    // Postfix: it++ — save old, increment, return OLD by value
    Iterator operator++(int) {
        Iterator old = *this;  // copy
        ++(*this);             // delegate to prefix
        return old;
    }

    Iterator& operator--()    { --pos; return *this; }
    Iterator  operator--(int) { Iterator old=*this; --(*this); return old; }
};
```

> **Prefer prefix `++` in loops.** Postfix creates an extra copy.

---

### 8. Conversion Operators

Convert your type to another type implicitly or explicitly:

```cpp
struct Celsius {
    double value;

    // explicit: prevents accidental implicit conversion
    explicit operator double() const { return value; }

    // implicit: use carefully
    operator std::string() const {
        return std::to_string(value) + "°C";
    }
};

Celsius c{100.0};
double d = static_cast<double>(c);  // explicit cast needed
std::string s = c;                  // implicit OK
```

> **Always prefer `explicit`** unless implicit conversion is genuinely natural (e.g., `std::string_view` from `std::string`).

---

### 9. Assignment Operators

Beyond the copy/move assignment from the Rule of Five, you can add **converting assignment**:

```cpp
struct MyString {
    std::string data;

    MyString& operator=(const std::string& s) { data = s; return *this; }
    MyString& operator=(std::string&& s)       { data = std::move(s); return *this; }
    MyString& operator=(const char* s)         { data = s; return *this; }
    MyString& operator=(char c)                { data = {c}; return *this; }
};
```

---

### 10. `new` / `delete` — Memory Control

Overload at class scope to control allocation:

```cpp
struct PoolObject {
    static MemoryPool pool;

    void* operator new(std::size_t size) {
        return pool.allocate(size);
    }
    void operator delete(void* ptr, std::size_t size) noexcept {
        pool.deallocate(ptr, size);
    }

    // Array forms:
    void* operator new[](std::size_t size);
    void  operator delete[](void* ptr) noexcept;
};
```

---

## Member vs. Free Function: The Rules

| Operator | Must be member | Should be free |
|---|---|---|
| `=`, `[]`, `()`, `->` | ✅ Yes | — |
| `<<`, `>>` (streams) | — | ✅ Yes (can't modify `std::ostream`) |
| Arithmetic `+`, `-`, `*` | — | ✅ Yes (enables `scalar * obj`) |
| `==`, `<=>` | Prefer member | Can be free |
| Type conversions | ✅ Yes | — |

---

## Critical Rules & Pitfalls

### 1. Return Types Matter

```cpp
T  operator+(...)  // returns new value — correct
T& operator+=(...)  // returns *this — correct
// DO NOT return T& from binary op — dangling reference to local
```

### 2. Preserve Semantics

Don't make `operator+` do subtraction. Users expect operators to behave like their built-in counterparts. The principle: **do as the ints do**.

### 3. Symmetry via Free Functions

```cpp
// Member: only works as obj * 2, NOT 2 * obj
Vec operator*(float s) const;

// Free: works both ways
Vec operator*(Vec v, float s);
Vec operator*(float s, Vec v);
```

### 4. `const`-correctness

Any operator that **reads** without modifying should be `const`:

```cpp
bool operator==(const T& rhs) const;  // ← must be const
T    operator+(const T& rhs)  const;  // ← must be const
T&   operator+=(const T& rhs);        // ← NOT const (modifies *this)
```

### 5. What You Cannot Overload

| Cannot overload |
|---|
| `::` (scope resolution) |
| `.` (member access) |
| `.*` (pointer-to-member) |
| `?:` (ternary) |
| `sizeof`, `alignof`, `typeid` |

---

## Complete Example: A `BigInt`-style Value Type

```cpp
#include <compare>
#include <iostream>
#include <vector>

class Fixed {  // fixed-point number: value = raw / 1000
    int64_t raw;
public:
    explicit Fixed(int64_t r = 0) : raw(r) {}
    static Fixed from_double(double d) { return Fixed{static_cast<int64_t>(d * 1000)}; }

    // Arithmetic (compound assignment as primitive)
    Fixed& operator+=(Fixed rhs) { raw += rhs.raw; return *this; }
    Fixed& operator-=(Fixed rhs) { raw -= rhs.raw; return *this; }
    Fixed& operator*=(Fixed rhs) { raw = raw * rhs.raw / 1000; return *this; }

    // Unary
    Fixed  operator-() const { return Fixed{-raw}; }
    Fixed  operator+() const { return *this; }

    // Comparison (C++20)
    auto operator<=>(const Fixed&) const = default;
    bool operator==(const Fixed&)  const = default;

    // Conversion
    explicit operator double() const { return raw / 1000.0; }

    friend std::ostream& operator<<(std::ostream& os, Fixed f) {
        return os << static_cast<double>(f);
    }
};

// Binary ops as free functions
inline Fixed operator+(Fixed a, Fixed b) { return a += b; }
inline Fixed operator-(Fixed a, Fixed b) { return a -= b; }
inline Fixed operator*(Fixed a, Fixed b) { return a *= b; }
```

---

## Quick Reference Card

| Goal | Signature |
|---|---|
| Addition | `T operator+(T lhs, const T& rhs)` (free) |
| Add-assign | `T& operator+=(const T& rhs)` (member) |
| Equality | `bool operator==(const T&) const` (member) |
| Ordering (C++20) | `auto operator<=>(const T&) const` |
| Stream out | `std::ostream& operator<<(std::ostream&, const T&)` (free friend) |
| Index | `T& operator[](size_t)` + const overload |
| Call | `R operator()(Args...)` |
| Dereference | `T& operator*() const` |
| Arrow | `T* operator->() const` |
| Prefix `++` | `T& operator++()` |
| Postfix `++` | `T operator++(int)` |
| Bool check | `explicit operator bool() const` |
| Conversion | `explicit operator TargetType() const` |

The golden rule: **operator overloading should make your type feel like a built-in type** — unsurprising, consistent, and efficient.