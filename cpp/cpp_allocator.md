## `std::allocator<T>` in C++

`std::allocator` is the **default memory allocator** used by all standard containers (`std::vector`, `std::list`, `std::map`, etc.). It decouples *memory management* from *object construction*, giving containers a uniform interface for raw memory.

---

### Core Responsibilities

| Operation | What it does |
|---|---|
| `allocate(n)` | Gets raw memory for `n` objects (no construction) |
| `deallocate(p, n)` | Releases raw memory (no destruction) |
| `construct(p, args...)` | Placement-new at `p` *(removed in C++20)* |
| `destroy(p)` | Calls destructor at `p` *(removed in C++20)* |

In C++17+, `construct`/`destroy` moved to `std::allocator_traits`.

---

### The Allocator–Container Contract

```
allocate()      → raw bytes (uninitialized)
    ↓
construct()     → object lives here
    ↓
destroy()       → destructor called
    ↓
deallocate()    → memory returned to OS/pool
```

Containers never call `new T` directly — they go through the allocator at every step.

---

### Basic Usage

```cpp
#include <memory>
#include <iostream>

int main() {
    std::allocator<int> alloc;

    // 1. Allocate raw memory for 5 ints (no construction)
    int* p = alloc.allocate(5);

    // 2. Construct objects in place
    std::allocator_traits<std::allocator<int>>::construct(alloc, p,     42);
    std::allocator_traits<std::allocator<int>>::construct(alloc, p + 1, 99);

    std::cout << p[0] << ", " << p[1] << "\n"; // 42, 99

    // 3. Destroy objects
    std::allocator_traits<std::allocator<int>>::destroy(alloc, p);
    std::allocator_traits<std::allocator<int>>::destroy(alloc, p + 1);

    // 4. Deallocate raw memory
    alloc.deallocate(p, 5);
}
```

---

### Writing a Custom Allocator

This is the main reason to understand `std::allocator` — you replace it to control *where* memory comes from (arena, pool, NUMA-local, etc.).

```cpp
template <typename T>
struct PoolAllocator {
    using value_type = T;

    PoolAllocator() = default;

    // Required: rebind to different type (e.g. node type inside std::list)
    template <typename U>
    PoolAllocator(const PoolAllocator<U>&) noexcept {}

    T* allocate(std::size_t n) {
        if (n > std::numeric_limits<std::size_t>::max() / sizeof(T))
            throw std::bad_alloc{};

        // Replace with your pool/arena logic:
        return static_cast<T*>(::operator new(n * sizeof(T)));
    }

    void deallocate(T* p, std::size_t n) noexcept {
        ::operator delete(p);
    }

    // Equality — stateless allocators are always equal
    bool operator==(const PoolAllocator&) const noexcept { return true; }
    bool operator!=(const PoolAllocator&) const noexcept { return false; }
};

// Use it with any container:
std::vector<int, PoolAllocator<int>> v;
v.push_back(1);
v.push_back(2);
```

---

### `std::allocator_traits` — The Real Interface

Containers don't call allocator methods directly; they go through `allocator_traits`, which fills in sensible defaults for anything your allocator doesn't define:

```cpp
std::allocator_traits<A>::allocate(a, n)
std::allocator_traits<A>::construct(a, p, args...)  // defaults to placement new
std::allocator_traits<A>::destroy(a, p)             // defaults to p->~T()
std::allocator_traits<A>::max_size(a)
std::allocator_traits<A>::select_on_container_copy_construction(a)
```

Your custom allocator only *needs* `value_type`, `allocate`, and `deallocate` — everything else is optional and gets a default.

---

### Minimum Required Interface (C++17/20)

```cpp
template <typename T>
struct MinimalAllocator {
    using value_type = T;

    T* allocate(std::size_t n);      // required
    void deallocate(T*, std::size_t); // required

    // allocator_traits fills in the rest
};
```

---

### Key Design Points for `goruntime-cpp`

Since you work with custom memory management in C++20:

- **Stateful allocators** — carry a pointer to your arena; containers propagate them via `propagate_on_container_copy_assignment` / `move_assignment` / `swap` type aliases.
- **`std::pmr`** — C++17's polymorphic allocator (`std::pmr::vector`, `std::pmr::memory_resource`) is often easier than custom allocator templates since you swap the *resource* at runtime, not the allocator type.
- **`allocate_at_least(n)`** — C++23 addition that lets the allocator return *more* than `n` objects if convenient (helps `vector` avoid over-requesting).

```cpp
// pmr example — swap backing memory without changing the container type
std::array<std::byte, 4096> buf;
std::pmr::monotonic_buffer_resource pool{buf.data(), buf.size()};
std::pmr::vector<int> v{&pool};  // same type as any other pmr::vector
```

The separation of allocation from construction is what makes `std::vector` able to `reserve()` capacity without constructing objects — a pattern worth mirroring in any custom container you build.