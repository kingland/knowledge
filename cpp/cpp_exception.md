`std::exception` is the base class for all standard library exceptions in C++, defined in `<exception>`.

## Core Interface

```cpp
class exception {
public:
    exception() noexcept;
    exception(const exception&) noexcept;
    exception& operator=(const exception&) noexcept;
    virtual ~exception();
    virtual const char* what() const noexcept;
};
```

The key method is `what()`, which returns a C-string describing the error.

## Standard Exception Hierarchy

```
std::exception
├── std::logic_error          // programmer errors, detectable before runtime
│   ├── std::invalid_argument
│   ├── std::domain_error
│   ├── std::length_error
│   └── std::out_of_range
├── std::runtime_error        // errors detectable only at runtime
│   ├── std::range_error
│   ├── std::overflow_error
│   ├── std::underflow_error
│   └── std::system_error
├── std::bad_alloc            // failed new
├── std::bad_cast             // failed dynamic_cast
├── std::bad_typeid           // typeid on null pointer
└── std::bad_function_call    // empty std::function invocation
```

## Creating Custom Exceptions

```cpp
// Simple approach: inherit from std::runtime_error or std::logic_error
class ParseError : public std::runtime_error {
public:
    ParseError(std::string_view msg, int line, int col)
        : std::runtime_error(std::format("{}:{}:{}", line, col, msg))
        , line_(line), col_(col) {}
    
    int line() const noexcept { return line_; }
    int col() const noexcept { return col_; }
    
private:
    int line_, col_;
};
```

For your parser work, deriving from `std::runtime_error` is typically cleaner than directly from `std::exception` since it handles the message storage for you—you'd otherwise need to manage the `what()` string lifetime yourself.

## Catching Pattern

```cpp
try {
    // ...
} catch (const ParseError& e) {
    // most specific first
} catch (const std::runtime_error& e) {
    // broader category
} catch (const std::exception& e) {
    // any standard exception
} catch (...) {
    // non-standard throws (discouraged)
}
```

Always catch by `const` reference to avoid slicing and unnecessary copies.