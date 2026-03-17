`std::transform` is an algorithm that applies a function to each element in a range and stores the results in a destination range.

## Basic Signature

```cpp
// Unary version: applies op to each element
template<class InputIt, class OutputIt, class UnaryOp>
OutputIt transform(InputIt first, InputIt last, OutputIt d_first, UnaryOp op);

// Binary version: applies op to pairs of elements from two ranges
template<class InputIt1, class InputIt2, class OutputIt, class BinaryOp>
OutputIt transform(InputIt1 first1, InputIt1 last1, InputIt2 first2, OutputIt d_first, BinaryOp op);
```

## Examples

**Unary transform** — modify each element:

```cpp
std::vector<int> v{1, 2, 3, 4, 5};
std::vector<int> result(v.size());

std::transform(v.begin(), v.end(), result.begin(), [](int x) {
    return x * x;
});
// result: {1, 4, 9, 16, 25}
```

**In-place transform** — output to same range:

```cpp
std::vector<std::string> words{"hello", "world"};
std::transform(words.begin(), words.end(), words.begin(), [](std::string& s) {
    s[0] = std::toupper(s[0]);
    return s;
});
// words: {"Hello", "World"}
```

**Binary transform** — combine two ranges:

```cpp
std::vector<int> a{1, 2, 3};
std::vector<int> b{10, 20, 30};
std::vector<int> sum(3);

std::transform(a.begin(), a.end(), b.begin(), sum.begin(), std::plus<>{});
// sum: {11, 22, 33}
```

## Key Points

The destination must have enough space — `std::transform` doesn't resize containers. For appending to an empty container, use `std::back_inserter`:

```cpp
std::vector<int> src{1, 2, 3};
std::vector<int> dst;

std::transform(src.begin(), src.end(), std::back_inserter(dst), [](int x) {
    return x * 2;
});
```

Since C++20, there's also `std::ranges::transform` which accepts ranges directly and provides projections:

```cpp
std::vector<int> v{1, 2, 3};
std::vector<int> out(v.size());

std::ranges::transform(v, out.begin(), [](int x) { return x * 2; });
```