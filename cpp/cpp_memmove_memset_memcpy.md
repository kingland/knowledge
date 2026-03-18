Here's a thorough breakdown.

---

## `memcpy`

```cpp
void* memcpy(void* dst, const void* src, size_t n);
```

Copies `n` bytes from `src` to `dst`. It makes **one critical assumption**: the two regions **do not overlap**. If they do, the behaviour is **undefined** — the compiler and CPU are free to read/write in any order, and on modern CPUs with SIMD vectorization, data gets corrupted silently.

It's the fast path. Most implementations use SIMD (SSE/AVX) load/store intrinsics because there's no need to worry about direction.

---

## `memmove`

```cpp
void* memmove(void* dst, const void* src, size_t n);
```

Handles overlapping regions correctly. Internally it checks:

- If `dst < src`: copy **forwards** (low address → high), safe
- If `dst > src`: copy **backwards** (high address → low), prevents clobbering unread source bytes

This makes it slightly slower than `memcpy` in theory, but in practice modern implementations (glibc, MSVC CRT) detect the non-overlapping case and delegate to the same fast path anyway.

---

## The overlap problem illustrated

When shifting bytes right in-place (e.g. `dst = src + 2`), if you copy forward:

```
src:  [A][B][C][D]
            ↕↕  ← overlap here
dst:        [_][_][_][_]

Forward copy step 1: dst[0] = src[0] = A  → buf: [A][B][A][D]
Forward copy step 2: dst[1] = src[1] = B  → buf: [A][B][B][D]
Forward copy step 3: dst[2] = src[2] = A  ← reads corrupted slot!
```

`memmove` would instead copy **from the end**, preserving original values before they're overwritten.

---

## Decision guide

| Scenario | Use |
|---|---|
| `dst` and `src` are guaranteed separate allocations | `memcpy` |
| Shifting data within the same buffer | `memmove` |
| Unsure / defensive code | `memmove` |
| `std::copy` on trivially-copyable types | Compiler likely emits `memcpy` anyway |

---

## Modern C++ equivalents

Prefer these over raw `mem*` calls:

```cpp
// Copy trivial types
std::copy(src, src + n, dst);             // may call memcpy internally
std::copy_backward(src, src+n, dst+n);   // like memmove backwards

// Safer: typed, no raw void*
std::ranges::copy(src_span, dst_span.begin());

// Explicit byte blasts (still UB if overlapping)
std::memcpy(dst, src, n * sizeof(T));

// Overlap-safe byte blast
std::memmove(dst, src, n * sizeof(T));
```

For `std::copy` on trivially-copyable types, the standard library is allowed — and most do — optimize it to a `memcpy` call. For overlapping ranges, `std::copy_backward` is the idiomatic C++ equivalent of `memmove`.

---

## Key rule of thumb

> Use `memcpy` when you **know** regions don't overlap (e.g., two separate `malloc` buffers).  
> Use `memmove` whenever regions **might** share memory — inserting/deleting elements in a flat array, ring buffers, in-place string manipulation.

## `memset`

```cpp
void* memset(void* ptr, int value, size_t count);
```

Writes the **low 8 bits** of `value` into every byte of the memory region `[ptr, ptr+count)`. Returns `ptr`.

---

### What it actually does

The `value` parameter is declared as `int`, but `memset` strips it down to a single byte (`(unsigned char)value`) and replicates that byte across every position. It does **not** write an `int`-wide value — always exactly 1 byte, repeated `count` times.

```cpp
char buf[8];
memset(buf, 0xAB, 8);
// buf = { AB AB AB AB AB AB AB AB }
```

---

### Common uses

**Zero-initialising a buffer** — the most frequent use:
```cpp
char buf[256];
memset(buf, 0, sizeof(buf));   // all bytes → 0x00
```
For `struct` or `class` types, prefer `= {}` or `std::fill` (see below), but `memset` is fine for POD types.

**Poison/sentinel fill** — used in debug allocators (MSVC uses `0xCD` for fresh heap, `0xDD` for freed heap):
```cpp
memset(ptr, 0xCD, size);   // mark as uninitialised in debug builds
```

**Wiping sensitive data** — zeroing a password or key buffer before freeing:
```cpp
memset(secret, 0, sizeof(secret));
// ⚠ compiler may optimise this away! Use SecureZeroMemory / explicit_bzero instead
```

---

### The big trap — non-replicating byte values

Because the value is **byte-replicated**, only two values work correctly for multi-byte integer types:

| `value` | Byte | Replicated across `int` | Result |
|---|---|---|---|
| `0` | `0x00` | `0x00000000` | `0` ✓ |
| `-1` / `0xFF` | `0xFF` | `0xFFFFFFFF` | `-1` (two's complement) ✓ |
| `1` | `0x01` | `0x01010101` | `16843009` ✗ |
| `5` | `0x05` | `0x05050505` | garbage ✗ |

This is the classic `memset` bug — expecting `memset(arr, 1, n*4)` to fill an `int` array with `1`s.

---

### Modern C++ alternatives

```cpp
// Zero-init at declaration — prefer this
int arr[100] = {};
MyStruct s{};

// Fill with arbitrary value — type-safe, no byte-replication trap
std::fill(arr, arr + 100, 42);
std::fill_n(arr, 100, 42);

// Ranges version (C++20)
std::ranges::fill(arr, 42);

// Secure wipe (prevents compiler dead-store elimination)
SecureZeroMemory(ptr, size);       // Windows MSVC
explicit_bzero(ptr, size);         // glibc / POSIX
```

---

### Rule of thumb

> Use `memset` for zeroing raw byte buffers (`0`) or all-bits-set fills (`0xFF`/`-1`). For anything else — especially typed arrays — use `std::fill` or value-initialisation to avoid the byte-replication trap.\


```cpp
#include <cstring>   // memset
#include <cstdint>
#include <cassert>
#include <array>

// ── 1. Zero-init a raw buffer ──────────────────────────────
void example_zero_init() {
    char buf[64];
    std::memset(buf, 0, sizeof(buf));
    assert(buf[0] == 0 && buf[63] == 0);
}

// ── 2. Poison fill (debug sentinel 0xCD) ───────────────────
void example_poison() {
    uint8_t heap_sim[32];
    std::memset(heap_sim, 0xCD, sizeof(heap_sim));
    // MSVC debug CRT uses 0xCD for uninitialised heap memory
}

// ── 3. Fill int array — only 0 and -1 are safe ─────────────
void example_int_fill() {
    int arr[4];

    std::memset(arr, 0,  sizeof(arr)); // ✓ all ints = 0
    std::memset(arr, -1, sizeof(arr)); // ✓ all ints = -1 (0xFFFFFFFF)

    // ✗ WRONG – byte 0x01 replicated: each int = 0x01010101
    // std::memset(arr, 1, sizeof(arr));
    // Use std::fill instead:
    std::fill(std::begin(arr), std::end(arr), 1); // ✓
}

// ── 4. Zero a std::array ───────────────────────────────────
void example_stdarray() {
    std::array<uint8_t, 16> key;
    std::memset(key.data(), 0, key.size());
}
```


```cpp
#include <cstring>
#include <cstdint>
#include <vector>

// ── 1. Copy between two separate heap buffers ──────────────
void example_heap_copy() {
    std::vector<uint8_t> src = {0xDE,0xAD,0xBE,0xEF};
    std::vector<uint8_t> dst(src.size());

    std::memcpy(dst.data(), src.data(), src.size()); // ✓ no overlap
}

// ── 2. Copy a POD struct (faster than assignment chain) ────
struct Vec3 { float x, y, z; };

void example_struct_copy(const Vec3& from, Vec3& to) {
    std::memcpy(&to, &from, sizeof(Vec3)); // safe for trivially-copyable
}

// ── 3. Type-punning (strict-aliasing safe in MSVC) ─────────
float bits_to_float(uint32_t bits) {
    float f;
    std::memcpy(&f, &bits, sizeof(f)); // defined behaviour (C++20: std::bit_cast)
    return f;
}

// ── 4. Stack array copy ────────────────────────────────────
void example_stack_copy() {
    char src[128]; // filled elsewhere
    char dst[128];
    std::memcpy(dst, src, sizeof(src));
    // MSVC /O2: emits REP MOVSB or AVX vmovdqu — no function call
}

// ── 5. MSVC-specific: _memccpy (copy until sentinel byte) ──
void example_memccpy() {
    char src[] = "hello\0world";
    char dst[16];
    // copies until '\0' found or limit reached; returns ptr past sentinel
    _memccpy(dst, src, '\0', sizeof(dst));
}
```

```cpp
#include <cstring>
#include <cstdint>
#include <vector>

// ── 1. In-place shift right (insert gap at front) ──────────
void shift_right(uint8_t* buf, size_t len, size_t gap) {
    // moves [0..len-gap) → [gap..len), regions overlap
    std::memmove(buf + gap, buf, len - gap);
    std::memset(buf, 0, gap);          // zero the freed prefix
}

// ── 2. Delete element from flat array ─────────────────────
void erase_at(int* arr, size_t count, size_t idx) {
    // close the gap left by element at idx
    std::memmove(arr + idx,
                 arr + idx + 1,
                 (count - idx - 1) * sizeof(int));
}

// ── 3. Ring buffer drain (src always inside same alloc) ────
void drain_ring(uint8_t* ring, size_t cap,
                size_t head, size_t tail, size_t n) {
    // compact n bytes starting at head to front of buffer
    std::memmove(ring, ring + head, n);
}

// ── 4. vector-style insert (manual) ───────────────────────
void insert_at(std::vector<int>& v, size_t pos, int val) {
    v.push_back(0);                    // extend storage first
    std::memmove(v.data() + pos + 1,
                 v.data() + pos,
                 (v.size() - pos - 1) * sizeof(int));
    v[pos] = val;
}

// ── 5. Shift left (remove prefix) ─────────────────────────
void consume(uint8_t* buf, size_t* len, size_t consumed) {
    *len -= consumed;
    std::memmove(buf, buf + consumed, *len); // dst < src → forward copy
}
```

```cpp
#include <cstring>
#include <windows.h>  // SecureZeroMemory
#include <wincrypt.h> // for key material example

// ── The problem: compiler may eliminate dead memset ────────
void bad_wipe(char* secret, size_t n) {
    std::memset(secret, 0, n);
    // If secret is not read after this, MSVC /O2 may remove
    // the memset as a dead store. Password stays in memory!
}

// ── MSVC solution 1: SecureZeroMemory (WinAPI) ────────────
void good_wipe_win(char* secret, size_t n) {
    SecureZeroMemory(secret, n);  // never optimised away
}

// ── MSVC solution 2: RtlSecureZeroMemory macro ────────────
void good_wipe_rtl(void* p, size_t n) {
    RtlSecureZeroMemory(p, n);    // same guarantee, kernel-mode safe
}

// ── MSVC solution 3: volatile memset workaround ───────────
void good_wipe_volatile(void* p, size_t n) {
    volatile char* vp = static_cast<volatile char*>(p);
    while (n--) *vp++ = 0;        // volatile prevents elision
}

// ── C11 / C23 portable alternative ────────────────────────
// #include <string.h>
// memset_s(secret, n, 0, n);   // C11 Annex K — MSVC supports it
void good_wipe_c11(char* secret, size_t n) {
    memset_s(secret, n, 0, n);   // guaranteed not elided, MSVC CRT
}

// ── Real-world pattern: key lifecycle ─────────────────────
struct CryptoKey { uint8_t bytes[32]; };

void use_and_wipe(const uint8_t* input, size_t len) {
    CryptoKey key{};
    // ... derive key from input ...
    // ... encrypt data ...
    SecureZeroMemory(&key, sizeof(key));  // wipe before leaving scope
}
```


```cpp
#include <cstring>
#include <type_traits>
#include <new>

// ── POD struct — memset/memcpy are safe ───────────────────
struct Packet {
    uint32_t id;
    uint16_t length;
    uint8_t  flags;
    uint8_t  reserved;
    uint8_t  payload[64];
};
static_assert(std::is_trivially_copyable_v<Packet>);

void init_packet(Packet& p) {
    std::memset(&p, 0, sizeof(p));   // ✓ POD — safe
}

void clone_packet(const Packet& src, Packet& dst) {
    std::memcpy(&dst, &src, sizeof(Packet)); // ✓
}

// ── Non-POD — DO NOT use memset/memcpy directly ───────────
struct BadExample {
    std::string name;    // has vtable-equivalent internals
    int value;
};
// std::memset(bad, 0, sizeof(BadExample)); // ✗ destroys std::string internals!

// ── Safe pattern: guard with type_trait ───────────────────
template<typename T>
void safe_zero(T& obj) {
    static_assert(std::is_trivially_copyable_v<T>,
        "safe_zero: T must be trivially copyable");
    std::memset(&obj, 0, sizeof(T));
}

// ── Placement new after memset (MSVC COM pattern) ─────────
struct Widget {
    int x, y, w, h;
    uint32_t flags;
};

Widget* create_widget(void* pool) {
    Widget* w = static_cast<Widget*>(pool);
    std::memset(w, 0, sizeof(Widget));
    return ::new (w) Widget{};  // placement new to properly construct
}

// ── Prefer value-init for stack objects ───────────────────
void modern_way() {
    Packet p{};            // zero-inits all members, no memset needed
    Packet q = {};         // identical
    auto r = Packet{};     // identical
}
```

Here's a summary of what each tab covers and MSVC-specific notes to keep in mind.

---

### MSVC compiler behaviour

Under `/O2`, MSVC intrinsifies all three functions for small constant sizes — they emit inline `MOV`, `REP MOVSB`, or AVX `vmovdqu` rather than calling the CRT function. You can verify this in Compiler Explorer (`godbolt.org`) targeting `x64 msvc v19.x`.

MSVC debug builds (`/RTC1`, `/MDd`) add runtime checks:
- `memcpy` asserts if `dst` and `src` overlap
- `memset` and `memmove` validate that the pointer is not null and count is reasonable
- Heap memory is pre-filled with sentinel bytes: `0xCD` (fresh allocation), `0xDD` (freed), `0xFD` (guard bytes around allocation)

---

### Quick decision table

| Situation | Prefer |
|---|---|
| Zero-init POD struct/array | `T obj{}` or `memset(&obj, 0, sizeof(T))` |
| Copy two separate buffers | `memcpy` |
| Copy within same buffer | `memmove` |
| Fill typed array with non-0/-1 | `std::fill` |
| Wipe a secret/key | `SecureZeroMemory` or `memset_s` |
| Copy non-trivial type | `std::copy` / assignment |

---

### Headers on MSVC

```cpp
#include <cstring>    // std::memset, std::memcpy, std::memmove  ← prefer
#include <memory.h>   // C-style aliases — also works but not idiomatic C++
#include <windows.h>  // SecureZeroMemory, RtlSecureZeroMemory
```