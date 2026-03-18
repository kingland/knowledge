Here's a thorough breakdown:

---

## Value categories first

Every expression in C++ is either an **lvalue** or an **rvalue**:

- **lvalue** — has a name, has an address, can appear on the left of `=`. Example: a local variable `x`.
- **rvalue** — a temporary, no persistent address. Example: `42`, `std::string("hello")`, or the result of `x + y`.

---

## `std::move`

```cpp
int x = 10;
auto&& ref = std::move(x);  // ref is int&&
```

`std::move` is **just a cast** — it converts an lvalue into an rvalue reference (`T&&`). It does **no moving at runtime**. The actual work happens in the move constructor or move assignment operator that gets selected because of this cast.

```cpp
std::vector<int> a = {1, 2, 3};
std::vector<int> b = std::move(a);  // b steals a's heap buffer
// a is now valid but empty (size == 0)
```

After the move, `a` is in a valid-but-unspecified state — you can reassign it, but don't read from it.

---

## `std::forward`

`std::forward` is used in **perfect forwarding** inside templates. The problem it solves: when you receive a parameter in a template, its value category is "lost" — everything named becomes an lvalue.

```cpp
template<typename T>
void wrapper(T&& arg) {
    // arg is ALWAYS an lvalue here — it has a name!
    doWork(arg);             // ❌ always copies
    doWork(std::move(arg));  // ❌ always moves (breaks lvalue callers)
    doWork(std::forward<T>(arg)); // ✅ preserves the original category
}
```

`std::forward<T>(arg)` conditionally casts:
- If `T` deduced as `int&` (lvalue was passed) → returns lvalue ref → copy happens
- If `T` deduced as `int` (rvalue was passed) → returns rvalue ref → move happens---

## Key mental model

| | `std::move` | `std::forward` |
|---|---|---|
| Used in | regular code | template functions |
| Does at runtime | nothing (pure cast) | nothing (pure cast) |
| Always casts to | `T&&` (rvalue ref) | `T&&` preserving category |
| Typical use | signal "I'm done with this" | pass args without losing move/copy intent |

---

## Quick rule of thumb

```cpp
// Use std::move when you own the object and are done with it
return std::move(localVar);   // but NRVO is usually better — don't do this blindly

// Use std::forward in a deduced T&& template parameter
template<typename T>
void make(T&& arg) {
    obj.init(std::forward<T>(arg));  // correct
}
```

One important gotcha: **never `std::forward` a named variable that you use more than once** in the same function — after a move, the source is in an unspecified state.

Want to go deeper into reference collapsing rules, move semantics in class design (Rule of Five), or `std::move` performance pitfalls?