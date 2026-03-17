# std::regex in C++

`std::regex` is C++'s standard library regular expression support, introduced in C++11. It provides pattern matching capabilities similar to other regex implementations but with some notable characteristics.

## Core Components

```cpp
#include <regex>

std::regex pattern(R"(\d{3}-\d{4})");  // compiled regex
std::smatch matches;                    // match results for strings
std::cmatch cmatches;                   // match results for const char*
```

## Main Operations

**Matching** checks if the entire string matches:
```cpp
std::string phone = "555-1234";
if (std::regex_match(phone, pattern)) {
    // entire string matches
}
```

**Searching** finds a pattern anywhere in the string:
```cpp
std::string text = "Call 555-1234 today";
if (std::regex_search(text, matches, pattern)) {
    std::cout << matches[0];  // "555-1234"
}
```

**Replacing** substitutes matches:
```cpp
std::string result = std::regex_replace(text, pattern, "XXX-XXXX");
```

**Iterating** over all matches:
```cpp
std::sregex_iterator it(text.begin(), text.end(), pattern);
std::sregex_iterator end;
for (; it != end; ++it) {
    std::cout << it->str() << '\n';
}
```

## The Performance Problem

Given your parser development work, you're likely already aware: `std::regex` has notoriously poor performance in most implementations. It compiles the pattern at runtime, and the matching algorithm (typically backtracking NFA) can be orders of magnitude slower than alternatives.

For performance-critical code like parsers, consider:

- **Compile-time regex** with [CTRE](https://github.com/hanickadot/compile-time-regular-expressions) — patterns compile to state machines at compile time
- **Hand-written lexers** — what you're already doing with your Java/C# parsers is typically faster
- **RE2** or **Hyperscan** — linear-time guarantees, no catastrophic backtracking

For one-off scripts or non-critical paths, `std::regex` works fine. For hot paths in a parser? Probably not the right tool.



This line declares a **static constant `std::regex` object** that matches any regex metacharacter.

## Breaking it down

```cpp
static const std::regex escape( "([.^$|()\\[\\]{}*+?\\\\])" );
```

**`static const`** — initialized once, shared across calls, immutable.

**The pattern `([.^$|()\\[\\]{}*+?\\\\])`** is a character class inside a capturing group:

| In the regex | Matches | Why escaped in C++ string? |
|--------------|---------|---------------------------|
| `.` | literal dot | No (harmless inside `[]`) |
| `^` | caret | No |
| `$` | dollar | No |
| `|` | pipe | No |
| `(` `)` | parentheses | No |
| `\\[` `\\]` | square brackets | Yes — `[` and `]` are special inside character classes |
| `{` `}` | braces | No |
| `*` `+` `?` | quantifiers | No |
| `\\\\` | backslash | Yes — needs `\\` for regex, then `\\` again for C++ string literal |

## Purpose

This regex is typically used to **escape user input** before embedding it in another regex:

```cpp
std::string safe = std::regex_replace(user_input, escape, R"(\$1)");
```

This prepends a backslash to every metacharacter, so `foo.bar*` becomes `foo\.bar\*`.

## Tip

If you're on C++11+, a raw string literal is cleaner:

```cpp
static const std::regex escape( R"(([.^$|()\[\]{}*+?\\]))" );
```

No double-escaping needed for backslashes.