Here's a comprehensive overview of your options for Decimal 128-bit in C++:

---

## Overview: What Is Decimal128?

`decimal128` is the IEEE 754-2008 standard for 128-bit decimal floating-point, giving you **34 significant decimal digits** and exponents from −6143 to +6144. It avoids the binary rounding errors of `double`, making it critical for financial/monetary calculations.

---

## Option 1: Boost.Decimal (Best Modern Choice ✅)

**Boost.Decimal** (`github.com/cppalliance/decimal`) is a ground-up implementation of IEEE 754 Decimal Floating Point types requiring C++14, co-authored by Matt Borland and Chris Kormanyos. It includes `decimal32`, `decimal64`, and `decimal128`, plus fast variants (`decimal128_fast`) that trade strict IEEE 754 bit-layout compliance for significantly better performance.

```cpp
#include <boost/decimal.hpp>

namespace dec = boost::decimal;

int main() {
    dec::decimal128 a{"1.23456789012345678901234567890123"};
    dec::decimal128 b{"9.87654321098765432109876543210987"};
    dec::decimal128 c = a + b;

    // Prints with full 34-digit precision
    std::cout << c << '\n';

    // Use decimal128_fast for better performance
    dec::decimal128_fast x{3.14};
    dec::decimal128_fast y{2.72};
    auto z = x * y;
}
```

**CMake integration:**
```cmake
find_package(Boost REQUIRED COMPONENTS decimal)
target_link_libraries(myapp PRIVATE Boost::decimal)
```

**Pros:** Header-only, actively maintained, full `<cmath>` support, `<charconv>`, benchmarked, MSVC/GCC/Clang compatible.

---

## Option 2: Intel DFPRT (via `stddecimal` wrapper)

The `stddecimal` library (`github.com/GaryHughes/stddecimal`) implements the proposed C++ ISO/IEC `std::decimal` (N2849/N3871) and is built on top of the Intel Decimal Floating Point Library, which handles the core arithmetic. It requires C++17.

```cpp
#include <decimal/decimal>

std::decimal::decimal128 price{"123.456"};
std::decimal::decimal128 tax = price * std::decimal::make_decimal128(8LL, -2); // 0.08
```

**Pros:** Closest to the proposed standard API (`std::decimal::decimal128`). **Cons:** Depends on Intel's DFPRT; less active than Boost.Decimal.

---

## Option 3: IBM decNumber / CppDecimal

The `CppDecimal` library wraps IBM's `decNumber` package, providing `decimal128` as a 128-bit decimal floating-point with 34 decimal digits of precision in compressed format (DPD encoding), plus a `decQuad` module with direct arithmetic functions.

```cpp
#include "decQuad.h"  // Direct decimal128 operations (C API)

decQuad a, b, result;
decContext ctx;
decContextDefault(&ctx, DEC_INIT_DECQUAD);

decQuadFromString(&a, "3.14159265358979323846264338327950", &ctx);
decQuadFromString(&b, "2.71828182845904523536028747135266", &ctx);
decQuadAdd(&result, &a, &b, &ctx);

char buf[DECQUAD_String];
decQuadToString(&result, buf);
```

**Pros:** Battle-tested (used in GCC/libdecnumber), no external deps. **Cons:** C API, more verbose.

---

## Option 4: GCC Built-in `_Decimal128` (GCC/Clang only)

GCC and Clang support `_Decimal128` as a compiler extension:

```cpp
// GCC/Clang only — NOT available on MSVC
#include <decimal/decimal>  // libstdc++ TR 24733

_Decimal128 a = 1.5DL;
_Decimal128 b = 2.5DL;
_Decimal128 c = a + b;
```

⚠️ **Not available on MSVC** — since you target Windows/MSVC, avoid this option.

---

## Comparison Table

| Library | Standard | C++ Version | MSVC Support | Performance | Maintained |
|---|---|---|---|---|---|
| **Boost.Decimal** | IEEE 754 | C++14+ | ✅ | ⭐⭐⭐ (`_fast` variants) | ✅ Active |
| **stddecimal** | `std::decimal` TR | C++17 | ✅ | ⭐⭐ | ⚠️ Slow |
| **CppDecimal / decNumber** | IEEE 754 DPD | C | ✅ | ⭐⭐ | ⚠️ Stable |
| **GCC `_Decimal128`** | TR 24733 | — | ❌ | ⭐⭐⭐ | N/A |

---

## Recommendation for Your Use Case

Given your MSVC/Windows compiler backend work, **Boost.Decimal is the clear choice**:
- Works with MSVC, CMake-friendly
- Use `decimal128` for strict IEEE 754 compliance, or `decimal128_fast` when performance matters
- Has full `std::numeric_limits` specializations, `<cmath>` math functions, and `<charconv>` for serialization