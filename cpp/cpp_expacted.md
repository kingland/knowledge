## `std::expected` in C++23

`std::expected<T, E>` represents a value that is **either a success (`T`) or an error (`E`)** — without using exceptions. It's the modern C++ alternative to returning error codes or using `std::optional`.

---

## The Problem It Solves

```cpp
// Old C style — error via out-param or return code
int parse_int(const char* s, int* out);   // what does int mean? errno?

// std::optional — knows "failed" but NOT why
std::optional<int> parse_int(std::string_view s);  // lost the error reason

// std::expected — knows success value AND error reason
std::expected<int, std::string> parse_int(std::string_view s);
//                    ^success  ^error
```

---

## Basic Anatomy

```cpp
#include <expected>   // C++23

std::expected<int, std::string> divide(int a, int b) {
    if (b == 0)
        return std::unexpected("division by zero");  // ← error path
    return a / b;                                    // ← success path
}
```

Returning success — just return `T` directly:

```cpp
return 42;                        // implicit success
return std::expected<int,E>(42);  // explicit (rarely needed)
```

Returning error — must wrap in `std::unexpected`:

```cpp
return std::unexpected("something went wrong");
return std::unexpected(ErrorCode::NotFound);
```

---

## Checking and Accessing the Value

```cpp
auto result = divide(10, 2);

// Check
if (result)             { /* has value */ }
if (result.has_value()) { /* same */ }

// Access success value
int v = result.value();       // throws std::bad_expected_access if error
int v = *result;              // UB if error — use only after checking
int v = result.value_or(0);  // safe fallback

// Access error
auto e = result.error();      // UB if success — use only after checking
```

Full pattern:

```cpp
auto result = divide(10, 0);

if (result) {
    std::println("Result: {}", *result);
} else {
    std::println("Error: {}", result.error());
}
```

---

## Error Type Choices

You can use any type as `E`:

```cpp
// String error
std::expected<int, std::string>

// Enum error code (most common in systems code)
enum class ParseError { InvalidInput, Overflow, Empty };
std::expected<int, ParseError>

// std::error_code (interops with system errors)
std::expected<int, std::error_code>

// Custom struct with extra context
struct Error {
    int         code;
    std::string message;
    int         line;
};
std::expected<int, Error>
```

---

## Chaining with `and_then` / `or_else` / `transform`

The real power — **monadic chaining** without nested `if` blocks:

```cpp
// Each step only runs if previous succeeded
auto result = read_file("data.txt")          // expected<string, Error>
    .and_then(parse_json)                    // runs if read succeeded
    .and_then(extract_field("user"))         // runs if parse succeeded
    .transform([](const Json& j) {           // transform the success value
        return j.as_string();
    })
    .or_else([](const Error& e) {            // handle any error
        log(e);
        return std::unexpected(e);
    });
```

| Method | Runs when | Returns |
|---|---|---|
| `.and_then(f)` | has value | `f(*this)` must return `expected` |
| `.transform(f)` | has value | wraps `f(*this)` in `expected` |
| `.or_else(f)` | has error | `f(error())` must return `expected` |
| `.transform_error(f)` | has error | wraps `f(error())` — maps error type |

---

## Real Example: Parsing Pipeline

```cpp
enum class ParseError { Empty, InvalidChar, Overflow };

std::expected<int, ParseError> parse_int(std::string_view s) {
    if (s.empty()) return std::unexpected(ParseError::Empty);

    int result = 0;
    for (char c : s) {
        if (!std::isdigit(c))
            return std::unexpected(ParseError::InvalidChar);
        result = result * 10 + (c - '0');
        if (result > 1'000'000)
            return std::unexpected(ParseError::Overflow);
    }
    return result;
}

std::expected<double, ParseError> parse_ratio(std::string_view a,
                                              std::string_view b) {
    return parse_int(a).and_then([&](int va) {
        return parse_int(b).transform([&](int vb) {
            return static_cast<double>(va) / vb;
        });
    });
}

// Usage
auto r = parse_ratio("10", "4");
if (r) std::println("{}", *r);        // 2.5
else   std::println("failed");
```

---

## In Move Constructors / RAII Code

Pairs perfectly with the RAII wrappers you write:

```cpp
class File {
public:
    static std::expected<File, std::error_code> open(const char* path) {
        FILE* f = std::fopen(path, "r");
        if (!f)
            return std::unexpected(
                std::error_code(errno, std::generic_category()));
        return File(f);  // success
    }

private:
    explicit File(FILE* f) : handle_(f) {}
    FILE* handle_;
};

// Caller
auto file = File::open("data.txt")
    .transform([](File& f) { return f.read_all(); })
    .value_or("");
```

---

## Comparison with Alternatives

| Approach | Carries error reason | No exceptions | Chainable | C++ std |
|---|---|---|---|---|
| Return code (`int`) | ✗ implicit | ✓ | ✗ | always |
| `errno` | ✗ global state | ✓ | ✗ | C |
| Exception | ✓ | ✗ | ✗ | always |
| `std::optional<T>` | ✗ | ✓ | partial | C++17 |
| `std::expected<T,E>` | ✓ | ✓ | ✓ | C++23 |

---

## Key Rules to Remember

```cpp
// ✓ Return success — just return T
return 42;

// ✓ Return error — must use std::unexpected
return std::unexpected(MyError::Failed);

// ✓ Safe access
if (r) use(*r);
r.value_or(default_val);

// ✗ Never access without checking
*r;          // UB if error
r.error();   // UB if success
```

---

## Summary

```
std::expected<T, E>
       │        │
       │        └─ error type  (returned via std::unexpected)
       └─ success type         (returned directly)
```

Think of it as **"a value or a reason it's missing"** — it makes error handling explicit, composable, and zero-overhead compared to exceptions, which fits perfectly with the systems/compiler work you do in C++.