There are several well-established patterns for safely wrapping C APIs in C++. Here's a practical guide:

## 1. RAII Wrappers (Most Important Pattern)

Wrap C handles/resources in classes that manage lifetime automatically:

```cpp
// C API
// FILE* fopen(...); int fclose(FILE*); size_t fread(...);

// C++ RAII wrapper
class File {
public:
    explicit File(const char* path, const char* mode) {
        handle_ = std::fopen(path, mode);
        if (!handle_)
            throw std::system_error(errno, std::generic_category(), path);
    }

    ~File() { if (handle_) std::fclose(handle_); }

    // Non-copyable
    File(const File&) = delete;
    File& operator=(const File&) = delete;

    // Movable
    File(File&& other) noexcept : handle_(std::exchange(other.handle_, nullptr)) {}
    File& operator=(File&& other) noexcept {
        if (this != &other) {
            if (handle_) std::fclose(handle_);
            handle_ = std::exchange(other.handle_, nullptr);
        }
        return *this;
    }

    FILE* get() const noexcept { return handle_; }

private:
    FILE* handle_ = nullptr;
};
```

---

## 2. `unique_ptr` with Custom Deleter (Lightweight)

For simple cases, skip the full class:

```cpp
// For any C resource with a destroy function
auto make_c_resource(const char* name) {
    using Deleter = decltype([](SomeHandle* h){ some_c_destroy(h); });
    return std::unique_ptr<SomeHandle, Deleter>(some_c_create(name));
}

// Usage — zero overhead, auto-cleanup
auto res = make_c_resource("foo");
res->do_something(); // use like pointer
// destroyed automatically
```

---

## 3. Error Handling: C errno / Error Codes → C++ Exceptions

```cpp
// Wrap C error codes into exceptions
void check(int ret, const char* context) {
    if (ret != 0)
        throw std::system_error(ret, std::generic_category(), context);
}

// Or wrap errno-style errors
void check_errno(int ret, const char* context) {
    if (ret < 0)
        throw std::system_error(errno, std::generic_category(), context);
}

// Usage
check_errno(::connect(fd, addr, len), "connect");
```

For non-throwing paths, use `std::expected` (C++23) or `std::optional`:

```cpp
auto safe_open(const char* path) -> std::expected<File, std::error_code> {
    FILE* f = std::fopen(path, "r");
    if (!f) return std::unexpected(std::error_code(errno, std::generic_category()));
    return File(f); // assume File takes ownership
}
```

---

## 4. Type-Safe Handle Wrappers (Avoid Raw `void*` / `int` Handles)

C APIs often use `int` or `void*` as opaque handles. Make them type-safe:

```cpp
// Strongly-typed handle — prevents accidental mixing
template<typename Tag>
struct Handle {
    int value = -1;
    explicit operator bool() const noexcept { return value >= 0; }
    bool operator==(const Handle&) const = default;
};

struct SocketTag {};
struct FileTag  {};

using SocketHandle = Handle<SocketTag>;
using FileHandle   = Handle<FileTag>;

// Now you can't accidentally pass a socket where a file is expected
```

---

## 5. Span / String View Instead of Raw Pointers

Replace `char*` + length pairs with safe views:

```cpp
// C API: void process(const char* data, size_t len);

// C++ wrapper — no copy, no ownership transfer
void process(std::span<const char> data) {
    ::c_process(data.data(), data.size()); // call C underneath
}

void process(std::string_view sv) {
    ::c_process(sv.data(), sv.size());
}
```

---

## 6. Callbacks: C Function Pointers → `std::function` / Lambdas

C callbacks use `void* user_data` to smuggle context. Bridge them cleanly:

```cpp
// C API: void set_callback(void (*cb)(int event, void* ctx), void* ctx);

template<typename F>
void set_callback_cpp(F&& func) {
    // Store on heap — C API holds raw pointer
    auto* stored = new std::function<void(int)>(std::forward<F>(func));

    ::set_callback([](int event, void* ctx) noexcept {
        auto* f = static_cast<std::function<void(int)>*>(ctx);
        (*f)(event);
    }, stored);
    // Note: need to manage stored lifetime carefully!
}
```

---

## 7. Namespace + `extern "C"` Hygiene

In headers that include C headers:

```cpp
// mylib_wrapper.hpp
extern "C" {
    #include <some_c_lib.h>   // prevent C++ name mangling
}

namespace mylib {
    // All your C++ wrappers go here
    class Connection { ... };
}
```

---

## Summary Table

| Problem | C Pattern | C++ Wrap |
|---|---|---|
| Resource leaks | Manual `free()`/`close()` | RAII class or `unique_ptr` + deleter |
| Error handling | Return codes / `errno` | Exceptions or `std::expected` |
| Type safety | `void*`, `int` handles | Typed `Handle<Tag>` template |
| Buffer overflows | `char*` + length | `std::span` / `std::string_view` |
| Callbacks | `void* user_data` | Lambda + trampoline |
| Name collisions | Global C names | `extern "C"` + namespace |

The most impactful pattern by far is **RAII** — it eliminates entire classes of resource leak bugs that are endemic in C APIs.