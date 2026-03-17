# C++ Cast Types and Common Usage

C++ provides four distinct cast operators, each serving specific purposes with different safety guarantees.

## `static_cast`

The most common cast, used for well-defined conversions the compiler can verify at compile time.

```cpp
// Numeric conversions
double d = 3.14;
int i = static_cast<int>(d);  // truncates to 3

// Enum to/from integer
enum class Color { Red, Green, Blue };
int val = static_cast<int>(Color::Green);  // 1

// Upcasting (derived to base) - always safe
class Base {};
class Derived : public Base {};
Derived d;
Base* b = static_cast<Base*>(&d);

// Downcasting (base to derived) - no runtime check, you must be certain
Base* b = new Derived();
Derived* d = static_cast<Derived*>(b);  // works if b actually points to Derived

// void* conversions
void* vp = static_cast<void*>(&i);
int* ip = static_cast<int*>(vp);
```

## `dynamic_cast`

Runtime-checked cast for polymorphic types (classes with virtual functions). Returns `nullptr` for pointers or throws `std::bad_cast` for references on failure.

```cpp
class Base { virtual ~Base() = default; };
class Derived : public Base { void special() {} };

Base* b = getSomeBase();

// Safe downcasting with runtime check
if (Derived* d = dynamic_cast<Derived*>(b)) {
    d->special();  // safe to use
}

// Reference version throws on failure
try {
    Derived& d = dynamic_cast<Derived&>(*b);
} catch (std::bad_cast&) {
    // handle failure
}
```

## `const_cast`

Adds or removes `const` (or `volatile`). Use sparingly—modifying a truly const object is undefined behavior.

```cpp
// Removing const (legitimate use: interfacing with legacy APIs)
void legacyApi(char* str);  // doesn't modify, but wasn't declared const

const char* msg = "hello";
legacyApi(const_cast<char*>(msg));  // OK if legacyApi doesn't actually modify

// Adding const (always safe)
int x = 42;
const int* cp = const_cast<const int*>(&x);
```

## `reinterpret_cast`

Bit-level reinterpretation. The most dangerous cast—use only when you truly need to treat memory as a different type.

```cpp
// Pointer to integer (platform-specific)
int* p = &x;
std::uintptr_t addr = reinterpret_cast<std::uintptr_t>(p);

// Type punning for serialization/hardware
struct Packet { uint32_t header; char data[100]; };
char buffer[104];
Packet* pkt = reinterpret_cast<Packet*>(buffer);  // alignment must be correct

// Function pointer conversions (rare, platform-specific)
using FuncPtr = void(*)();
void* handle = dlsym(lib, "func");
FuncPtr f = reinterpret_cast<FuncPtr>(handle);
```

## Quick Reference

| Cast | Compile-time | Runtime check | Typical use |
|------|-------------|---------------|-------------|
| `static_cast` | ✓ | ✗ | Numeric conversions, known-safe up/downcasts |
| `dynamic_cast` | ✓ | ✓ | Safe polymorphic downcasting |
| `const_cast` | ✓ | ✗ | Adding/removing const for API compatibility |
| `reinterpret_cast` | ✓ | ✗ | Low-level bit reinterpretation, serialization |

## Modern C++ Guidance

Prefer `static_cast` for most situations. Use `dynamic_cast` when you genuinely don't know the runtime type. Avoid `const_cast` unless interfacing with const-incorrect APIs. Reserve `reinterpret_cast` for low-level code like memory-mapped I/O or binary serialization where you fully understand the implications.