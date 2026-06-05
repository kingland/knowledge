The code as written has a fundamental C++ limitation: **virtual function templates are not allowed**. The compiler needs to know the vtable size at compile time, but templates generate an unbounded number of instantiations.

Here are the practical alternatives:

---

## Option 1: `std::any` (runtime type erasure)

```cpp
#include <any>
#include <stdexcept>

class IGetter {
public:
    virtual ~IGetter() = default;
    virtual std::any get() = 0;

    template<typename T>
    T getAs() {
        return std::any_cast<T>(get());
    }
};

class IValue : public IGetter {
public:
    explicit IValue(std::any value) : value_(std::move(value)) {}

    std::any get() override {
        return value_;
    }

private:
    std::any value_;
};

// Usage:
// IValue v(42);
// int x = v.getAs<int>();
```

---

## Option 2: `std::variant` (closed set of types, zero overhead)

```cpp
#include <variant>

using ValueType = std::variant<int, float, double, std::string>;

class IGetter {
public:
    virtual ~IGetter() = default;
    virtual ValueType get() = 0;

    template<typename T>
    T getAs() {
        return std::get<T>(get());
    }
};

class IValue : public IGetter {
public:
    explicit IValue(ValueType value) : value_(std::move(value)) {}

    ValueType get() override {
        return value_;
    }

private:
    ValueType value_;
};
```

---

## Option 3: CRTP (static polymorphism, no vtable at all)

```cpp
template<typename Derived>
class IGetter {
public:
    template<typename T>
    T get() {
        return static_cast<Derived*>(this)->template getImpl<T>();
    }
};

class IValue : public IGetter<IValue> {
public:
    explicit IValue(int value) : value_(value) {}

    template<typename T>
    T getImpl() {
        return static_cast<T>(value_);
    }

private:
    int value_;
};

// Usage:
// IValue v(42);
// int x = v.get<int>();
```

---

## Which to choose?

| Option | Type safety | Runtime polymorphism | Overhead |
|---|---|---|---|
| `std::any` | Cast throws at runtime | ✅ Yes | Heap alloc possible |
| `std::variant` | Compile-time closed set | ✅ Yes | Zero |
| CRTP | Full compile-time | ❌ No | Zero |

For your systems/compiler work, **`std::variant` is usually the sweet spot** — it gives you a known set of value types (like an AST node value), zero overhead, and `std::visit` for exhaustive dispatch. `std::any` is best when the type set is truly open.