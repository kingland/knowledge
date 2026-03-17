# std::error_code in C++

`std::error_code` is a lightweight, portable mechanism for representing and transporting error conditions without using exceptions. It's part of `<system_error>` (C++11).

## Core Structure

```cpp
class error_code {
    int val_;                        // numeric error value
    const error_category* cat_;      // pointer to category singleton
};
```

It's essentially a pair: an integer code plus a category that gives meaning to that integer.

## Basic Usage

```cpp
#include <system_error>
#include <iostream>
#include <fstream>

void example() {
    std::error_code ec;
    
    // Many standard library functions accept an error_code& parameter
    std::filesystem::remove("nonexistent.txt", ec);
    
    if (ec) {  // contextually converts to bool
        std::cout << "Error: " << ec.message() << '\n';
        std::cout << "Code: " << ec.value() << '\n';
        std::cout << "Category: " << ec.category().name() << '\n';
    }
}
```

## Creating Error Codes

```cpp
// From system/platform errors (errno on POSIX, GetLastError() on Windows)
std::error_code ec1 = std::make_error_code(std::errc::no_such_file_or_directory);

// From raw values
std::error_code ec2(ENOENT, std::generic_category());   // portable errno values
std::error_code ec3(ERROR_FILE_NOT_FOUND, std::system_category());  // platform-specific

// Default construction (no error)
std::error_code ec4;  // value() == 0
```

## Categories Explained

Categories interpret what error codes mean:

```cpp
std::generic_category()   // portable POSIX-style errors (std::errc enum)
std::system_category()    // platform-native errors (errno or Win32)
std::iostream_category()  // I/O stream errors
std::future_category()    // std::future errors
```

Two error codes are equal only if both value *and* category match:

```cpp
auto ec1 = std::error_code(2, std::generic_category());
auto ec2 = std::error_code(2, std::system_category());
// ec1 != ec2, even though both have value 2
```

## std::error_code vs std::error_condition

| `error_code` | `error_condition` |
|--------------|-------------------|
| Platform-specific, exact error | Portable, abstract error |
| "What happened" | "What kind of problem" |
| Use for transport/storage | Use for comparison/handling |

```cpp
std::error_code ec(EACCES, std::system_category());

// Compare against portable condition, not platform-specific code
if (ec == std::errc::permission_denied) {
    // handles EACCES on POSIX, ERROR_ACCESS_DENIED on Windows
}
```

## Custom Error Categories

```cpp
enum class MyError {
    success = 0,
    invalid_input,
    connection_failed,
    timeout
};

class MyErrorCategory : public std::error_category {
public:
    const char* name() const noexcept override { return "MyLibrary"; }
    
    std::string message(int ev) const override {
        switch (static_cast<MyError>(ev)) {
            case MyError::success:          return "success";
            case MyError::invalid_input:    return "invalid input";
            case MyError::connection_failed: return "connection failed";
            case MyError::timeout:          return "operation timed out";
            default:                        return "unknown error";
        }
    }
    
    // Optional: map to generic conditions for portable comparisons
    std::error_condition default_error_condition(int ev) const noexcept override {
        if (ev == static_cast<int>(MyError::invalid_input))
            return std::errc::invalid_argument;
        return std::error_condition(ev, *this);
    }
};

// Singleton accessor
const MyErrorCategory& my_error_category() {
    static MyErrorCategory instance;
    return instance;
}

// ADL-findable make_error_code overload
std::error_code make_error_code(MyError e) {
    return {static_cast<int>(e), my_error_category()};
}

// Enable implicit conversion
template<>
struct std::is_error_code_enum<MyError> : std::true_type {};
```

Now you can use it naturally:

```cpp
std::error_code ec = MyError::timeout;
if (ec == MyError::timeout) { /* ... */ }
```

## Practical Patterns

**Function returning error code:**
```cpp
std::error_code read_config(const std::string& path, Config& out) {
    std::ifstream file(path);
    if (!file) 
        return std::make_error_code(std::errc::no_such_file_or_directory);
    
    if (!parse(file, out))
        return MyError::invalid_input;
    
    return {};  // success
}
```

**Expected-style wrapper (pre-C++23):**
```cpp
template<typename T>
struct Result {
    std::optional<T> value;
    std::error_code error;
    
    explicit operator bool() const { return value.has_value(); }
};
```

## Why Use It?

1. **No exceptions** — suitable for embedded, real-time, or performance-critical code
2. **Portable** — abstracts platform differences behind `error_condition`
3. **Extensible** — custom categories integrate seamlessly
4. **Lightweight** — just two pointers, trivially copyable
5. **Composable** — different libraries' errors can coexist and be compared

It's the foundation for error handling in `<filesystem>`, `<chrono>`, networking TS, and many modern C++ libraries.


# std::runtime_error in C++

`std::runtime_error` is an exception class for errors detectable only at runtime. It's part of `<stdexcept>` and inherits from `std::exception`.

## Exception Hierarchy

```
std::exception
├── std::logic_error          ← programmer mistakes (preventable)
│   ├── std::invalid_argument
│   ├── std::domain_error
│   ├── std::length_error
│   └── std::out_of_range
│
└── std::runtime_error        ← external/unpredictable failures
    ├── std::range_error
    ├── std::overflow_error
    ├── std::underflow_error
    └── std::system_error     ← wraps std::error_code
```

The distinction: `logic_error` means "your code is wrong" while `runtime_error` means "something outside your control failed."

## Basic Usage

```cpp
#include <stdexcept>
#include <string>

void load_config(const std::string& path) {
    std::ifstream file(path);
    if (!file) {
        throw std::runtime_error("Failed to open config: " + path);
    }
    // ...
}

int main() {
    try {
        load_config("settings.json");
    } catch (const std::runtime_error& e) {
        std::cerr << "Runtime error: " << e.what() << '\n';
        return 1;
    }
}
```

## Construction

```cpp
// From C-string
throw std::runtime_error("something failed");

// From std::string
std::string msg = "Error at line " + std::to_string(line);
throw std::runtime_error(msg);

// Copy construction (exceptions must be copyable)
std::runtime_error err("original");
std::runtime_error copy = err;
```

The message is stored internally and accessed via `what()`:

```cpp
const char* what() const noexcept override;
```

## When to Use runtime_error vs Others

| Exception | Use For |
|-----------|---------|
| `runtime_error` | General runtime failures (file I/O, network, parsing) |
| `system_error` | OS/system calls with error codes |
| `range_error` | Computation result out of meaningful range |
| `overflow_error` | Arithmetic overflow |
| `invalid_argument` | Bad function argument (logic error) |
| `out_of_range` | Index/key not found (logic error) |

```cpp
// runtime_error: external failure
if (!connect(server))
    throw std::runtime_error("Connection refused");

// system_error: OS error with code
if (fd < 0)
    throw std::system_error(errno, std::generic_category(), "open failed");

// invalid_argument: caller's mistake
if (denominator == 0)
    throw std::invalid_argument("denominator cannot be zero");
```

## Custom Exception Classes

Derive from `runtime_error` for domain-specific exceptions:

```cpp
class NetworkError : public std::runtime_error {
public:
    NetworkError(const std::string& msg, int code)
        : std::runtime_error(msg), error_code_(code) {}
    
    int code() const noexcept { return error_code_; }
    
private:
    int error_code_;
};

class ParseError : public std::runtime_error {
public:
    ParseError(const std::string& msg, int line, int col)
        : std::runtime_error(format_message(msg, line, col))
        , line_(line), col_(col) {}
    
    int line() const noexcept { return line_; }
    int column() const noexcept { return col_; }
    
private:
    int line_, col_;
    
    static std::string format_message(const std::string& msg, int line, int col) {
        return msg + " at line " + std::to_string(line) + 
               ", column " + std::to_string(col);
    }
};
```

## Catching Exceptions

Catch by const reference to avoid slicing:

```cpp
try {
    parse_file(path);
} 
catch (const ParseError& e) {
    // Most specific first
    std::cerr << "Parse error at " << e.line() << ":" << e.column() << '\n';
}
catch (const std::runtime_error& e) {
    // Catches runtime_error and all derived types
    std::cerr << "Runtime error: " << e.what() << '\n';
}
catch (const std::exception& e) {
    // Catches everything derived from std::exception
    std::cerr << "Exception: " << e.what() << '\n';
}
```

## Exception Safety Considerations

```cpp
// BAD: might leak if runtime_error is thrown
void bad() {
    int* p = new int[1000];
    might_throw();  // if this throws, p leaks
    delete[] p;
}

// GOOD: RAII handles cleanup
void good() {
    auto p = std::make_unique<int[]>(1000);
    might_throw();  // p automatically cleaned up on throw
}
```

## noexcept and runtime_error

```cpp
// Destructor should never throw
~MyClass() noexcept {
    // cleanup, no exceptions
}

// Mark functions that won't throw
int size() const noexcept { return size_; }

// Let exceptions propagate naturally
void process() {  // implicitly noexcept(false)
    if (error) throw std::runtime_error("failed");
}
```

## Comparison: Exceptions vs Error Codes

| Aspect | std::runtime_error | std::error_code |
|--------|-------------------|-----------------|
| Control flow | Non-local jumps | Explicit checking |
| Performance | Costly on throw | Minimal overhead |
| Ignoring errors | Impossible | Easy to forget |
| Stack unwinding | Automatic | Manual |
| Use case | Exceptional failures | Expected failures |

```cpp
// Exception style: cleaner happy path
std::string read_file(const std::string& path) {
    std::ifstream f(path);
    if (!f) throw std::runtime_error("cannot open " + path);
    return std::string(std::istreambuf_iterator<char>(f), {});
}

// Error code style: explicit control
std::error_code read_file(const std::string& path, std::string& out) {
    std::ifstream f(path);
    if (!f) return std::make_error_code(std::errc::no_such_file_or_directory);
    out.assign(std::istreambuf_iterator<char>(f), {});
    return {};
}
```

## Best Practices

1. **Throw by value, catch by const reference** — avoids slicing and unnecessary copies

2. **Keep what() messages informative** — include context like filenames, values, positions

3. **Derive for domain-specific exceptions** — allows targeted catching and additional data

4. **Don't use exceptions for control flow** — they're for *exceptional* situations

5. **Document which exceptions functions throw** — especially in library interfaces

`std::runtime_error` is your go-to for "something external went wrong" scenarios where exceptions are appropriate.