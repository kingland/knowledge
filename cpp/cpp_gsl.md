GSL in C++ refers to two different things — here's a quick breakdown:

## 1. C++ Core Guidelines Support Library (GSL)

This is the more common meaning in modern C++ circles. It's a library created by Microsoft (and maintained as an open-source project) that implements the types and functions recommended by the **C++ Core Guidelines**, co-authored by Bjarne Stroustrup and Herb Sutter.

**GitHub:** `microsoft/GSL`

Key types it provides:

- **`gsl::span<T>`** — a non-owning view over a contiguous sequence (like `std::span` in C++20, which it inspired)
- **`gsl::not_null<T>`** — wraps a pointer and enforces it can never be null at compile/runtime
- **`gsl::owner<T>`** — annotates raw pointers that *own* memory, for static analysis tools
- **`gsl::narrow<T>()`** — a safe narrowing cast that throws if the value doesn't fit
- **`gsl::narrow_cast<T>()`** — like `static_cast` but documents intent
- **`gsl::finally()`** — RAII scope-exit utility
- **`gsl::zstring` / `gsl::czstring`** — type aliases for null-terminated C strings
- **`GSL_SUPPRESS(...)`** — suppress specific Core Guidelines warnings

**Usage example:**
```cpp
#include <gsl/gsl>

void process(gsl::not_null<int*> ptr, gsl::span<int> data) {
    *ptr = gsl::narrow<int>(data.size()); // throws if overflow
}
```

---

## 2. GNU Scientific Library (GSL)

A completely separate C library (usable from C++) for **numerical and scientific computing**. It provides:

- Linear algebra, FFT, numerical integration
- Random number generators, statistics
- Interpolation, root finding, ODEs
- Special mathematical functions

It's a C library, so using it in C++ works but isn't idiomatic modern C++. Projects doing heavy numerical work might prefer Eigen or Armadillo instead.

---

## Which one to use?

| You're doing... | Use... |
|---|---|
| Modern C++ with Core Guidelines | `microsoft/GSL` |
| Scientific/numerical computing | GNU GSL |

In most modern C++ contexts — especially with your work on compiler and systems code — **Microsoft GSL** is the relevant one, and many of its types (like `gsl::not_null` and `gsl::span`) are genuinely useful for expressing ownership and bounds intent clearly.