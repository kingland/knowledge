## Porting LINQ to C++

LINQ (Language Integrated Query) concepts map well to modern C++, primarily through **ranges** (C++20) and the STL algorithms. Here's a comprehensive guide:

---

## Core LINQ → C++ Mapping

### 1. `Where` → `std::views::filter`

```cpp
// C# LINQ
var evens = numbers.Where(x => x % 2 == 0);

// C++20 Ranges
auto evens = numbers | std::views::filter([](int x) { return x % 2 == 0; });
```

### 2. `Select` → `std::views::transform`

```cpp
// C#
var squares = numbers.Select(x => x * x);

// C++20
auto squares = numbers | std::views::transform([](int x) { return x * x; });
```

### 3. `SelectMany` → `std::views::join` / `join_with`

```cpp
// C#
var flat = lists.SelectMany(x => x);

// C++20
auto flat = lists | std::views::join;
```

### 4. `Take` / `Skip`

```cpp
auto first5  = numbers | std::views::take(5);
auto after5  = numbers | std::views::drop(5);
auto middle  = numbers | std::views::drop(2) | std::views::take(3);
```

### 5. `OrderBy` → `std::ranges::sort`

```cpp
// C# (lazy)
var sorted = people.OrderBy(p => p.age);

// C++ (eager — ranges sort is in-place)
std::ranges::sort(people, {}, &Person::age);  // project by member

// Or with a copy
auto sorted = people;
std::ranges::sort(sorted, [](const auto& a, const auto& b){ return a.age < b.age; });
```

### 6. `GroupBy` → `std::unordered_map` + loop

```cpp
// C#
var groups = people.GroupBy(p => p.dept);

// C++
std::unordered_map<std::string, std::vector<Person>> groups;
for (const auto& p : people)
    groups[p.dept].push_back(p);
```

### 7. `Aggregate` / `Reduce` → `std::accumulate` / `std::reduce`

```cpp
// C#
var sum = numbers.Aggregate(0, (acc, x) => acc + x);

// C++
int sum = std::accumulate(numbers.begin(), numbers.end(), 0);
// or
int sum = std::reduce(std::execution::par, numbers.begin(), numbers.end(), 0);
```

### 8. `Any` / `All` / `Count`

```cpp
bool anyEven  = std::ranges::any_of(v, [](int x){ return x % 2 == 0; });
bool allPos   = std::ranges::all_of(v, [](int x){ return x > 0; });
auto cnt      = std::ranges::count_if(v, [](int x){ return x > 10; });
```

### 9. `First` / `FirstOrDefault`

```cpp
// First matching element
auto it = std::ranges::find_if(v, pred);
if (it != v.end()) { /* use *it */ }

// With C++23 optional-like wrapper
auto first = v | std::views::filter(pred) | std::views::take(1);
```

### 10. `Distinct` → `std::ranges::unique` / set

```cpp
auto sorted_copy = v;
std::ranges::sort(sorted_copy);
auto [first, last] = std::ranges::unique(sorted_copy);
sorted_copy.erase(first, last);

// Or use std::unordered_set to preserve order:
std::unordered_set<int> seen;
auto distinct = v | std::views::filter([&](int x){ return seen.insert(x).second; });
```

### 11. `Zip`

```cpp
// C++23
auto zipped = std::views::zip(v1, v2);

// C++20 workaround
auto zipped = std::views::iota(0uz, std::min(v1.size(), v2.size()))
            | std::views::transform([&](size_t i){ return std::pair{v1[i], v2[i]}; });
```

### 12. `Join` (SQL-style inner join)

```cpp
// C# LINQ join
var result = from o in orders
             join c in customers on o.customerId equals c.id
             select new { o, c };

// C++ — build a lookup map first (O(n) instead of O(n²))
std::unordered_map<int, Customer> customerMap;
for (const auto& c : customers) customerMap[c.id] = c;

std::vector<std::pair<Order, Customer>> result;
for (const auto& o : orders) {
    if (auto it = customerMap.find(o.customerId); it != customerMap.end())
        result.emplace_back(o, it->second);
}
```

---

## Chaining (Pipelines)

C++20 ranges support full lazy pipelines just like LINQ:

```cpp
auto result = employees
    | std::views::filter([](const auto& e){ return e.dept == "Engineering"; })
    | std::views::transform([](const auto& e){ return e.salary; })
    | std::views::take(10);

double avg = std::accumulate(result.begin(), result.end(), 0.0) 
           / std::ranges::distance(result);
```

---

## Building a LINQ-style Fluent Wrapper (C++20)

If you want the `.Where().Select()` dot-chaining syntax:

```cpp
template<std::ranges::range R>
struct LinqView {
    R range_;

    template<typename Pred>
    auto where(Pred&& pred) && {
        return LinqView{ range_ | std::views::filter(std::forward<Pred>(pred)) };
    }

    template<typename Proj>
    auto select(Proj&& proj) && {
        return LinqView{ range_ | std::views::transform(std::forward<Proj>(proj)) };
    }

    auto take(std::size_t n) && {
        return LinqView{ range_ | std::views::take(n) };
    }

    template<typename T = std::ranges::range_value_t<R>>
    auto to_vector() && {
        return std::vector<T>(range_.begin(), range_.end());
    }

    auto begin() { return std::ranges::begin(range_); }
    auto end()   { return std::ranges::end(range_); }
};

template<std::ranges::range R>
auto from(R&& r) { return LinqView<R>{ std::forward<R>(r) }; }

// Usage:
auto result = from(employees)
    .where([](const auto& e){ return e.age > 30; })
    .select([](const auto& e){ return e.name; })
    .to_vector();
```

---

## Quick Reference Table

| LINQ | C++20 Equivalent |
|---|---|
| `Where` | `views::filter` |
| `Select` | `views::transform` |
| `SelectMany` | `views::join` |
| `Take` / `Skip` | `views::take` / `views::drop` |
| `OrderBy` | `ranges::sort` + projection |
| `GroupBy` | `unordered_map` grouping |
| `Aggregate` | `std::accumulate` / `std::reduce` |
| `Any` / `All` | `ranges::any_of` / `ranges::all_of` |
| `Count` | `ranges::count_if` |
| `Distinct` | `ranges::unique` or set |
| `Zip` | `views::zip` (C++23) |
| `First` | `ranges::find_if` |
| `ToList` | `std::vector(r.begin(), r.end())` |

---

## Key Differences to Keep in Mind

- **Laziness**: C++ range views are lazy like LINQ — they don't evaluate until iterated.
- **Materialization**: Use `.to_vector()` or range-for to force evaluation (like `.ToList()` in C#).
- **Mutability**: C++ ranges can mutate in-place (`sort`, `reverse`) — LINQ always produces new sequences.
- **No query syntax**: C++ has no `from/where/select` keywords — only method chaining or pipe syntax.
- **Third-party libraries**: [range-v3](https://github.com/ericniebler/range-v3) (the precursor to C++20 ranges) offers even more LINQ-like operators.