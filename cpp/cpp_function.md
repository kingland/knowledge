`std::function` is a general-purpose polymorphic function wrapper in C++ that can store, copy, and invoke any callable target—functions, lambda expressions, bind expressions, function objects, or pointers to member functions.

## Basic Usage

```cpp
#include <functional>

// Store a regular function
int add(int a, int b) { return a + b; }
std::function<int(int, int)> fn = add;

// Store a lambda
std::function<int(int)> square = [](int x) { return x * x; };

// Store a functor
struct Multiplier {
    int factor;
    int operator()(int x) const { return x * factor; }
};
std::function<int(int)> triple = Multiplier{3};
```

## How It Works

`std::function` uses type erasure internally—it hides the concrete callable type behind a uniform interface. This involves:

1. **Small buffer optimization (SBO)**: Small callables are stored inline to avoid heap allocation
2. **Virtual dispatch or function pointers**: To invoke the stored callable polymorphically

## When to Use It

**Good use cases:**
- Callbacks with runtime-determined behavior
- Storing heterogeneous callables in containers
- API boundaries where you need a stable function signature

```cpp
class Button {
    std::function<void()> onClick;
public:
    void setCallback(std::function<void()> cb) { onClick = std::move(cb); }
    void click() { if (onClick) onClick(); }
};
```

## Performance Considerations

`std::function` has overhead compared to raw function pointers or templates:

- Potential heap allocation for large callables
- Indirect call (no inlining)
- Typically ~32-48 bytes in size

**Prefer templates when possible:**

```cpp
// Zero overhead—callable is known at compile time
template<typename F>
void process(F&& func) { func(); }

// Use std::function only when you need type erasure
void registerCallback(std::function<void()> cb);
```

## Checking for Empty State

```cpp
std::function<void()> fn;
if (fn) {  // or fn != nullptr
    fn();
} else {
    // empty—calling would throw std::bad_function_call
}
```

Given your parser work, you might use `std::function` for visitor callbacks or semantic actions, though for performance-critical paths in your Java parser, direct virtual dispatch or CRTP-based approaches would avoid the indirection cost.


A lambda in C++ is an anonymous function object that you can define inline. Introduced in C++11 and enhanced in subsequent standards, lambdas provide a concise way to write function objects without explicitly defining a class.

## Basic Syntax

```cpp
[capture](parameters) -> return_type { body }
```

```cpp
auto add = [](int a, int b) { return a + b; };
int result = add(3, 4);  // 7
```

The return type is usually deduced automatically, so `-> return_type` is often omitted.

## Capture Clause

The capture clause `[]` determines how the lambda accesses variables from the enclosing scope.

```cpp
int x = 10;
int y = 20;

auto by_value = [x]() { return x; };           // copy of x
auto by_ref = [&x]() { x++; };                 // reference to x
auto all_by_value = [=]() { return x + y; };   // copy all used variables
auto all_by_ref = [&]() { x++; y++; };         // reference all used variables
auto mixed = [x, &y]() { y = x; };             // x by value, y by reference
```

**C++14 init-capture** allows creating new variables in the capture:

```cpp
auto ptr = std::make_unique<int>(42);
auto lambda = [p = std::move(ptr)]() { return *p; };  // move into capture
```

## Mutable Lambdas

By default, captured-by-value variables are const inside the lambda. Use `mutable` to modify them:

```cpp
int count = 0;
auto counter = [count]() mutable { return ++count; };
counter();  // returns 1
counter();  // returns 2
// original count is still 0
```

## Generic Lambdas (C++14)

Use `auto` for parameters to create templated lambdas:

```cpp
auto print = [](const auto& x) { std::cout << x << '\n'; };
print(42);
print("hello");
print(3.14);
```

## Template Lambdas (C++20)

Explicit template syntax for more control:

```cpp
auto cast = []<typename T>(auto value) { return static_cast<T>(value); };
int i = cast.operator()<int>(3.14);  // awkward call syntax

// More useful with constraints
auto process = []<typename T>(std::vector<T>& v) {
    // T is explicitly available
};
```

## Lambdas vs `std::function`

A lambda has a unique, unnamed type known at compile time:

```cpp
auto lambda = [](int x) { return x * 2; };  // zero overhead, can inline

std::function<int(int)> fn = lambda;  // type erasure, potential allocation
```

Use `auto` when possible; use `std::function` only when you need to store lambdas of different types uniformly.

## Immediately Invoked Lambda (IIFE)

Useful for complex initialization:

```cpp
const auto config = [&]() {
    Config c;
    c.name = computeName();
    c.value = expensive_calculation();
    return c;
}();  // note the () - invoked immediately
```

## Common Patterns

**With STL algorithms:**
```cpp
std::vector<int> v = {1, 2, 3, 4, 5};
auto it = std::find_if(v.begin(), v.end(), [](int n) { return n > 3; });
std::sort(v.begin(), v.end(), [](int a, int b) { return a > b; });
```

**Recursive lambdas (C++23 deducing this, or workaround):**
```cpp
// C++23
auto fib = [](this auto&& self, int n) -> int {
    return n <= 1 ? n : self(n - 1) + self(n - 2);
};

// Pre-C++23 workaround
std::function<int(int)> fib = [&fib](int n) {
    return n <= 1 ? n : fib(n - 1) + fib(n - 2);
};
```

For your parser work, lambdas are excellent for inline semantic actions or visitor callbacks where you don't need the full generality of `std::function`.
