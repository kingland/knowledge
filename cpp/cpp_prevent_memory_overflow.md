## Preventing Memory Overflow in C++

Here are the key strategies:

### 1. Use Smart Pointers (RAII)
```cpp
// Bad - manual memory, easy to leak
int* arr = new int[1000];
// ... (exception here = leak)
delete[] arr;

// Good - automatic cleanup
auto arr = std::make_unique<int[]>(1000);
auto obj = std::make_shared<MyClass>();
```

### 2. Prefer Stack Allocation & STL Containers
```cpp
// Avoid raw heap when stack works
int arr[100];               // stack - auto freed
std::vector<int> v(1000);  // manages its own memory
std::string s = "hello";   // no manual char*
```

### 3. Avoid Buffer Overflows
```cpp
// Bad
char buf[10];
strcpy(buf, "this is too long!");  // overflow!

// Good
std::string s = "this is too long!";  // safe, grows automatically

// Or with bounds checking
std::array<int, 5> arr{};
arr.at(10) = 1;  // throws std::out_of_range instead of UB
```

### 4. Reserve Capacity for Vectors
```cpp
std::vector<int> v;
v.reserve(10000);  // pre-allocate to avoid repeated reallocations

for (int i = 0; i < 10000; i++)
    v.push_back(i);  // no reallocation overhead
```

### 5. Move Semantics — Avoid Unnecessary Copies
```cpp
std::vector<std::string> words;

// Bad - copies the string
std::string s = "hello";
words.push_back(s);

// Good - moves ownership, no extra allocation
words.push_back(std::move(s));
words.emplace_back("world");  // constructs in-place
```

### 6. Custom Allocators / Memory Pools
```cpp
// For high-frequency small allocations, use a pool
std::pmr::monotonic_buffer_resource pool(1024 * 1024); // 1MB pool
std::pmr::vector<int> v(&pool);  // allocates from pool, not heap
```

### 7. Detect Leaks with Tools

| Tool | Platform | Usage |
|------|----------|-------|
| **Valgrind** | Linux | `valgrind --leak-check=full ./app` |
| **AddressSanitizer** | GCC/Clang | `-fsanitize=address` |
| **Dr. Memory** | Windows | GUI + CLI |
| **_CrtDumpMemoryLeaks** | MSVC | Heap leak report at exit |

With MSVC (since you target Windows):
```cpp
#define _CRTDBG_MAP_ALLOC
#include <cstdlib>
#include <crtdbg.h>

int main() {
    _CrtSetDbgFlag(_CRTDBG_ALLOC_MEM_DF | _CRTDBG_LEAK_CHECK_DF);
    // ... your code
}  // prints leak report on exit
```

### 8. Stack Overflow Prevention
```cpp
// Bad - huge stack array can overflow the stack
void foo() {
    int bigArray[10'000'000];  // ~40MB on stack = crash
}

// Good - put large data on heap
void foo() {
    auto bigArray = std::make_unique<int[]>(10'000'000);
}
```

---

**Core principle:** follow **RAII** — every resource acquisition should be tied to an object's lifetime. If you're calling `new`/`delete` manually in modern C++17/20, that's usually a design smell.