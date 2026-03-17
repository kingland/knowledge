## C++ Number Literal Suffixes

Number suffixes tell the compiler what **type** to use for an integer or floating-point literal.

### Integer Suffixes

| Suffix | Type | Example |
|--------|------|---------|
| *(none)* | `int` | `42` |
| `U` or `u` | `unsigned int` | `42U` |
| `L` or `l` | `long` | `42L` |
| `UL` / `LU` | `unsigned long` | `42UL` |
| `LL` or `ll` | `long long` | `42LL` |
| `ULL` / `LLU` | `unsigned long long` | `42ULL` |
| `Z` *(C++23)* | `ptrdiff_t` (signed size) | `42Z` |
| `UZ` *(C++23)* | `size_t` | `42UZ` |

### Floating-Point Suffixes

| Suffix | Type | Example |
|--------|------|---------|
| *(none)* | `double` | `3.14` |
| `F` or `f` | `float` | `3.14F` |
| `L` or `l` | `long double` | `3.14L` |
| `BF16` *(C++23)* | `std::bfloat16_t` | `3.14BF16` |

---

### Why `1LL` matters

```cpp
int a = 1;           // 32-bit,  max ~2.1 billion
long long b = 1LL;   // 64-bit,  max ~9.2 quintillion
```

The suffix is critical in expressions to **prevent overflow before assignment**:

```cpp
// WRONG — overflow! RHS computed as int (32-bit)
long long bad = 1000000 * 1000000;       // overflows, then assigned

// CORRECT — computed as long long (64-bit)
long long good = 1000000LL * 1000000LL;  // = 1'000'000'000'000
```

### Bit-shift example (very common bug)

```cpp
// WRONG — shifts a 32-bit int, UB if bit >= 32
uint64_t mask = 1 << 40;      // undefined behavior!

// CORRECT
uint64_t mask = 1ULL << 40;   // fine, 64-bit shift
```

### Case sensitivity note

Lowercase `l` is legal but **avoid it** — it looks like the digit `1`:

```cpp
long val = 1l;   // hard to read: is that 1L or 11?
long val = 1L;   // clear and unambiguous ✓
```

The convention is to use **uppercase suffixes** (`LL`, `ULL`, `F`, etc.) for readability.