Here are some powerful C++ template metaprogramming techniques worth exploring:

## Compile-Time Computation

**Constexpr with templates** — Modern C++ (17/20) reduces the need for pure TMP by letting `constexpr` functions do heavy lifting, but combining them with templates remains powerful:

```cpp
template<std::size_t N>
constexpr auto fibonacci() {
    if constexpr (N <= 1) return N;
    else return fibonacci<N-1>() + fibonacci<N-2>();
}
```

## SFINAE and Concepts

For your parser work, **constrained templates** help create clean overload sets:

```cpp
// C++20 concepts
template<typename T>
concept ASTNode = requires(T t) {
    { t.accept(std::declval<Visitor&>()) } -> std::same_as<void>;
    { t.children() } -> std::ranges::range;
};

template<ASTNode T>
void traverse(T& node) { /* ... */ }
```

## Type Lists and Manipulation

Useful for AST node type hierarchies:

```cpp
template<typename... Ts> struct TypeList {};

template<typename List, typename T> struct Append;
template<typename... Ts, typename T>
struct Append<TypeList<Ts...>, T> {
    using type = TypeList<Ts..., T>;
};

// Filter types matching a predicate
template<template<typename> class Pred, typename List>
struct Filter;
```

## Tag Dispatching with `if constexpr`

Clean branching for token/node type handling:

```cpp
template<TokenKind Kind>
struct TokenTag {};

template<TokenKind Kind>
auto parse(TokenTag<Kind>) {
    if constexpr (Kind == TokenKind::Identifier)
        return parseIdentifier();
    else if constexpr (Kind == TokenKind::Literal)
        return parseLiteral();
}
```

## Curiously Recurring Template Pattern (CRTP)

Static polymorphism for visitor patterns without vtable overhead:

```cpp
template<typename Derived>
class VisitorBase {
public:
    void visit(Expression& e) {
        static_cast<Derived*>(this)->visitExpression(e);
    }
};
```

## Detection Idiom

Check for member existence at compile time — handy for generic AST operations:

```cpp
template<typename T, typename = void>
struct has_source_location : std::false_type {};

template<typename T>
struct has_source_location<T, 
    std::void_t<decltype(std::declval<T>().location())>> : std::true_type {};
```

---

Any particular area you want to dive deeper into? Given your parser/compiler work, type list manipulation for node variants or compile-time string processing might be especially relevant.