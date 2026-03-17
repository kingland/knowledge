`std::enable_if` is a metaprogramming utility in C++ that conditionally enables or disables function overloads or class template specializations based on compile-time boolean conditions. It's part of the SFINAE (Substitution Failure Is Not An Error) toolkit.

## Basic Mechanism

```cpp
template<bool B, class T = void>
struct enable_if {};

template<class T>
struct enable_if<true, T> { using type = T; };
```

When the condition is `true`, `enable_if<true, T>::type` exists and equals `T`. When `false`, there's no `type` member—causing substitution failure, which removes that overload from consideration.

## Common Usage Patterns

**As a return type:**
```cpp
template<typename T>
std::enable_if_t<std::is_integral_v<T>, T>
square(T x) {
    return x * x;
}

template<typename T>
std::enable_if_t<std::is_floating_point_v<T>, T>
square(T x) {
    return x * x;  // could use different implementation
}
```

**As a template parameter (often cleaner):**
```cpp
template<typename T, std::enable_if_t<std::is_integral_v<T>, int> = 0>
void process(T value) {
    // integral version
}

template<typename T, std::enable_if_t<std::is_floating_point_v<T>, int> = 0>
void process(T value) {
    // floating-point version
}
```

**In class templates:**
```cpp
template<typename T, typename = void>
struct Container {
    // general implementation
};

template<typename T>
struct Container<T, std::enable_if_t<std::is_trivially_copyable_v<T>>> {
    // optimized for trivially copyable types
};
```

## C++17/20 Alternatives

Since you're working with C++20+, you'll often prefer `if constexpr` or concepts:

```cpp
// if constexpr (C++17)
template<typename T>
auto square(T x) {
    if constexpr (std::is_integral_v<T>) {
        return x * x;
    } else {
        return x * x;  // different logic if needed
    }
}

// Concepts (C++20) — much cleaner
template<std::integral T>
T square(T x) { return x * x; }

template<std::floating_point T>
T square(T x) { return x * x; }
```

Concepts are generally preferred in modern C++ for readability and better error messages, but `enable_if` remains useful for backward compatibility and certain edge cases where concepts don't fit cleanly.