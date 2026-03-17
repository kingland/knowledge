# std::multimap in C++

`std::multimap` is an associative container that stores key-value pairs where **multiple elements can have the same key**. It's like `std::map`, but without the unique-key constraint.

## Core characteristics

- Keys are sorted (by default using `operator<`)
- Allows duplicate keys
- Elements with equal keys maintain insertion order (since C++11)
- Implemented as a balanced binary tree (typically red-black tree)
- Lookup, insertion, and removal are O(log n)

## Basic usage

```cpp
#include <map>
#include <string>
#include <iostream>

int main() {
    std::multimap<std::string, int> scores;
    
    // Insert multiple values for the same key
    scores.insert({"Alice", 95});
    scores.insert({"Bob", 87});
    scores.insert({"Alice", 92});  // duplicate key allowed
    scores.insert({"Alice", 88});
    
    // Iteration (sorted by key)
    for (const auto& [name, score] : scores) {
        std::cout << name << ": " << score << '\n';
    }
    // Output:
    // Alice: 95
    // Alice: 92
    // Alice: 88
    // Bob: 87
}
```

## Finding elements with a specific key

```cpp
// Count occurrences
auto count = scores.count("Alice");  // returns 3

// Get range of elements with key
auto [begin, end] = scores.equal_range("Alice");
for (auto it = begin; it != end; ++it) {
    std::cout << it->second << '\n';
}

// Or use lower_bound/upper_bound
auto lb = scores.lower_bound("Alice");  // first Alice
auto ub = scores.upper_bound("Alice");  // past last Alice
```

## When to use multimap vs alternatives

| Container | Use case |
|-----------|----------|
| `std::multimap` | Sorted keys, duplicates allowed, frequent range queries |
| `std::map` | Sorted keys, unique keys only |
| `std::unordered_multimap` | Unsorted, duplicates allowed, faster average lookup |
| `std::map<K, std::vector<V>>` | When you need to modify all values for a key together |

## Practical example: event log by timestamp

```cpp
#include <map>
#include <string>
#include <chrono>

using TimePoint = std::chrono::system_clock::time_point;

std::multimap<TimePoint, std::string> event_log;

void log_event(const std::string& msg) {
    event_log.insert({std::chrono::system_clock::now(), msg});
}

// Get all events in a time range
void print_events_between(TimePoint start, TimePoint end) {
    auto it_start = event_log.lower_bound(start);
    auto it_end = event_log.upper_bound(end);
    
    for (auto it = it_start; it != it_end; ++it) {
        std::cout << it->second << '\n';
    }
}
```

The key advantage over `std::map<K, std::vector<V>>` is that `multimap` keeps all entries in sorted order globally, making range queries efficient without needing to flatten nested containers.