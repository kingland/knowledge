In C++, `wchar_t` is a built-in type designed to represent "wide characters"—characters that may require more than one byte to encode.

## The basics

`wchar_t` was introduced to handle character sets larger than ASCII or Latin-1, particularly for internationalization. You use it with the `L` prefix for literals:

```cpp
wchar_t ch = L'ก';           // Thai character
const wchar_t* str = L"สวัสดี";  // wide string literal
std::wstring ws = L"Hello";   // wide string class
```

## The problem with wchar_t

The width of `wchar_t` is implementation-defined, which causes portability headaches:

- **Windows**: 16 bits (UTF-16, but actually UCS-2 behavior in many APIs)
- **Linux/macOS**: 32 bits (can hold any Unicode code point directly)

This means code using `wchar_t` behaves differently across platforms. On Windows, characters outside the Basic Multilingual Plane (like emoji 🎵) require surrogate pairs, while on Linux they fit in a single `wchar_t`.

## Associated facilities

```cpp
std::wstring            // wide string class
std::wcout / std::wcin  // wide character streams
std::wifstream          // wide file streams
wcslen(), wcscpy()      // C-style wide string functions
```

## Modern alternatives

For new code, especially cross-platform work, consider:

- **`char8_t`** (C++20) with UTF-8 encoding—portable and compact
- **`char16_t`** — guaranteed 16-bit, for UTF-16
- **`char32_t`** — guaranteed 32-bit, for UTF-32

For your Thai text processing work, UTF-8 (`std::string` with `char8_t` or plain `char`) is often simpler than `wchar_t`, since TIS-620 conversion libraries typically work with byte sequences anyway.