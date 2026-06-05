For a template class like `FlagSet`, the implementation **must stay in the header** — you can't split it into a `.cpp` the normal way because the compiler needs the full definition at instantiation time.

You have three options:

---

**Option 1: Everything in the `.hpp` (simplest)**
```cpp
// flagset.hpp
template<typename Out>
class FlagSet {
    template<typename W>
    IWriter<W> output() {
        // implementation here
    }
};
```

---

**Option 2: Declare in `.hpp`, define in `.inl`, include at bottom**
```cpp
// flagset.hpp
template<typename Out>
class FlagSet {
    template<typename W>
    IWriter<W> output();
};

#include "flagset.inl"  // ← included, not compiled separately
```
```cpp
// flagset.inl
template<typename Out>
template<typename W>
IWriter<W> FlagSet<Out>::output() {
    // implementation here
}
```

---

**Option 3: Explicit instantiation in `.cpp` (only works if you know all types upfront)**
```cpp
// flagset.cpp
#include "flagset.hpp"

template<typename Out>
template<typename W>
IWriter<W> FlagSet<Out>::output() {
    // implementation
}

// Explicitly instantiate only the combinations you need:
template class FlagSet<MyOut>;
template IWriter<MyWriter> FlagSet<MyOut>::output<MyWriter>();
```

---

**Recommendation:** Use **Option 2** (`.inl` pattern) — it keeps the header clean and declaration/definition separate, which is the idiomatic C++ approach for large template libraries. Option 3 is only practical when `Out` and `W` are a closed, known set.