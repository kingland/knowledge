## `std::exchange` in C++

`std::exchange` replaces a value with a new one and **returns the old value** — in a single atomic-ish operation.

---

## Signature

```cpp
// <utility> header, since C++14
template<class T, class U = T>
T exchange(T& obj, U&& new_val);
```

It does exactly this — but as **one expression**:

```cpp
// What exchange does internally:
T old = std::move(obj);
obj = std::forward<U>(new_val);
return old;
```

---

## Why It Exists — The Problem It Solves

Without `std::exchange`, swapping out a value and capturing the old one requires **separate steps**:

```cpp
// Clunky manual way
FILE* old = handle_;
handle_ = nullptr;
return old;

// Clean way with exchange
return std::exchange(handle_, nullptr);
```

Same result, one line, no chance of using the variable after you've moved from it.

---

## Most Common Use Case: Move Constructor / Move Assignment

This is where you'll see it most in real code:

```cpp
class File {
    FILE* handle_ = nullptr;
public:
    // Move constructor — take other's handle, leave other as null
    File(File&& other) noexcept
        : handle_(std::exchange(other.handle_, nullptr))
    {}

    // Move assignment
    File& operator=(File&& other) noexcept {
        if (this != &other) {
            if (handle_) std::fclose(handle_);
            handle_ = std::exchange(other.handle_, nullptr);
            //        ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
            //        Gets other.handle_, sets other.handle_ = nullptr
        }
        return *this;
    }

    ~File() {
        if (handle_) std::fclose(handle_);  // safe — nullptr check
    }
};
```

Without `exchange`, the move constructor would be:

```cpp
// Without exchange — more verbose, easy to forget the nullptr reset
File(File&& other) noexcept : handle_(other.handle_) {
    other.handle_ = nullptr;  // Must remember this!
}
```

---

## Step-by-Step Trace

```cpp
File a = open("foo.txt");   // a.handle_ = 0xABCD
File b = std::move(a);      // move constructor fires

// Inside move ctor:
// handle_ = std::exchange(other.handle_, nullptr)
//         = std::exchange(0xABCD, nullptr)
// Step 1: saves old = 0xABCD
// Step 2: other.handle_ = nullptr   ← a is now "empty"
// Step 3: returns 0xABCD            ← b.handle_ gets it

// Result: b.handle_ = 0xABCD, a.handle_ = nullptr ✓
```

---

## Other Practical Uses

### Flag toggling / state machines

```cpp
bool dirty_ = false;

// Check and clear in one step
bool was_dirty = std::exchange(dirty_, false);
if (was_dirty) flush();
```

### Returning old value while updating

```cpp
int counter_ = 0;

// Post-increment style — return old, store new
int next_id() {
    return std::exchange(counter_, counter_ + 1);
    // same as counter_++, but works for any type
}
```

### Swap-like patterns

```cpp
// Transfer ownership of a node pointer
Node* detach() {
    return std::exchange(root_, nullptr);
}
```

### In destructors (defensive reset)

```cpp
~Connection() {
    if (auto h = std::exchange(handle_, nullptr))
        ::disconnect(h);
}
```

---

## `std::exchange` vs Related Tools

| Tool | Purpose |
|---|---|
| `std::exchange(a, b)` | Replace `a` with `b`, **return old `a`** |
| `std::swap(a, b)` | Swap two variables, returns `void` |
| `std::move(a)` | Cast to rvalue, doesn't change `a` itself |
| `std::forward<T>(a)` | Perfect forwarding in templates |

Key difference from `swap`:

```cpp
std::swap(a, b);          // a ↔ b, both changed
std::exchange(a, b);      // a = b, old a returned — b unchanged
```

---

## With Non-Pointer Types

Works on any assignable type, not just pointers:

```cpp
std::string name_ = "old";
std::string prev = std::exchange(name_, "new");
// prev == "old", name_ == "new"

std::vector<int> buf_;
auto old_buf = std::exchange(buf_, {});  // clear and take old contents
```

---

## Summary

```
std::exchange(obj, new_val)
      │              │
      │              └─ what obj becomes
      └─ what gets returned (old value)
```

Think of it as **"replace and return old"** — most valuable in move operations where you need to steal a resource and leave the source in a valid empty state in one clean expression.