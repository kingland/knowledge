## String Prefixes in C++

String literals in C++ can have prefixes that control their encoding, character type, and behavior.

---

### Character Type Prefixes

| Prefix | Type | Size | Encoding |
|--------|------|------|----------|
| *(none)* | `const char[]` | 1 byte | Implementation-defined (usually UTF-8) |
| `L"..."` | `const wchar_t[]` | 2B (Windows) / 4B (Linux) | UTF-16 or UTF-32 |
| `u8"..."` | `const char8_t[]` *(C++20)* | 1 byte | UTF-8 |
| `u"..."` | `const char16_t[]` | 2 bytes | UTF-16 |
| `U"..."` | `const char32_t[]` | 4 bytes | UTF-32 |

```cpp
const char*     s1 = "hello";       // narrow string
const wchar_t*  s2 = L"hello";      // wide string (TCHAR on Windows often maps here)
const char8_t*  s3 = u8"hello";     // UTF-8  (C++20)
const char16_t* s4 = u"hello";      // UTF-16
const char32_t* s5 = U"hello";      // UTF-32
```

---

### Raw String Prefix — `R"(...)"` 

Disables escape sequence processing. Great for regex, file paths, and multiline text.

```cpp
// Without raw: need to escape backslashes
const char* path1 = "C:\\Users\\nhookeaw\\docs\\file.txt";

// With raw: backslashes are literal
const char* path2 = R"(C:\Users\nhookeaw\docs\file.txt)";

// Regex without raw — painful
const char* re1 = "\\d{4}-\\d{2}-\\d{2}";

// Regex with raw — clean
const char* re2 = R"(\d{4}-\d{2}-\d{2})";

// Multiline
const char* html = R"(
    <html>
        <body>Hello</body>
    </html>
)";
```

The full raw syntax is `R"delimiter(content)delimiter"` — you can choose any delimiter to avoid conflicts with `)` inside the string:

```cpp
// If content contains ")": use a custom delimiter
const char* s = R"xyz(some ) tricky ) content)xyz";
```

---

### Combining Prefixes

`R` can be combined with encoding prefixes:

```cpp
const wchar_t*  ws = LR"(C:\path\file)";
const char8_t*  u8s = u8R"(\n is not a newline here)";
const char16_t* u16s = uR"(raw UTF-16)";
const char32_t* u32s = UR"(raw UTF-32)";
```

---

### Practical Notes for Windows/MSVC

Since you work with MSVC and `tchar.h`, a few relevant points:

```cpp
#include <tchar.h>

// _T() / TEXT() macro — resolves to L"..." in Unicode builds, "..." in MBCS
LPCTSTR s = _T("hello");   // = L"hello" when UNICODE defined

// Prefer explicit L"..." for new code — TCHAR is legacy
std::wstring ws = L"สวัสดี";   // Thai in wide string

// Windows API paths — raw strings help a lot
CreateFile(LR"(\\?\C:\Very\Long\Path\file.txt)", ...);
```

---

### Key Takeaways

- Use **`L"..."`** for Windows API wide strings (`LPCWSTR`, `std::wstring`)
- Use **`u8"..."`** (C++20) or plain `"..."` for UTF-8 in cross-platform code
- Use **`R"(...)"`** whenever your string contains backslashes or spans multiple lines — it eliminates escape clutter significantly
- **`u"..."`** and **`U"..."`** are rarely needed in practice unless you're doing explicit UTF-16/32 processing (like in your Thai encoding work)