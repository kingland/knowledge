`std::to_string` is a standard library function that converts numeric types to their string representation.

## Basic Usage

```cpp
#include <string>

int n = 42;
std::string s = std::to_string(n);  // "42"

double pi = 3.14159;
std::string ps = std::to_string(pi);  // "3.141590"
```

## Supported Types

It has overloads for all fundamental numeric types: `int`, `long`, `long long`, `unsigned` variants, `float`, `double`, and `long double`.

## Key Characteristics

**Floating-point formatting** — Uses `%f` style formatting with 6 decimal places by default, which often produces more digits than you want:

```cpp
std::to_string(1.0);      // "1.000000"
std::to_string(1e10);     // "10000000000.000000" (no scientific notation)
```

**Locale-independent** — Always uses the "C" locale, so decimal point is always `.` regardless of system settings.

**No formatting control** — You can't specify precision, width, or base. For those, use `std::format` (C++20) or `std::ostringstream`.

## Alternatives

For more control, especially in C++23 code:

```cpp
// std::format (C++20) — preferred for formatted output
std::string s = std::format("{:.2f}", 3.14159);  // "3.14"
std::string hex = std::format("{:x}", 255);      // "ff"

// std::to_chars (C++17) — fast, no allocation, writes to buffer
char buf[32];
auto [ptr, ec] = std::to_chars(buf, buf + 32, 42);

// ostringstream — flexible but slower
std::ostringstream oss;
oss << std::setprecision(2) << 3.14159;
```

Given your parser work, `std::to_chars` might be worth considering for diagnostic message generation where performance matters — it avoids heap allocation entirely.