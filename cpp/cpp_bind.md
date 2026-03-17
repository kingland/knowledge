`std::bind` creates a callable wrapper that binds arguments to a function, allowing you to fix some parameters while leaving others as placeholders.

```cpp
#include <functional>
#include <iostream>

void greet(const std::string& greeting, const std::string& name) {
    std::cout << greeting << ", " << name << "!\n";
}

int main() {
    // Bind first argument, leave second as placeholder
    auto say_hello = std::bind(greet, "Hello", std::placeholders::_1);
    say_hello("Alice");  // Hello, Alice!
    
    // Bind second argument
    auto greet_bob = std::bind(greet, std::placeholders::_1, "Bob");
    greet_bob("Goodbye");  // Goodbye, Bob!
    
    // Reorder arguments
    auto reversed = std::bind(greet, std::placeholders::_2, std::placeholders::_1);
    reversed("Charlie", "Hi");  // Hi, Charlie!
}
```

**With member functions:**

```cpp
struct Calculator {
    int multiply(int a, int b) { return a * b; }
};

Calculator calc;
auto times_five = std::bind(&Calculator::multiply, &calc, std::placeholders::_1, 5);
std::cout << times_five(3);  // 15
```

**Modern alternative — prefer lambdas:**

Since C++14, lambdas are generally cleaner and more efficient:

```cpp
// Instead of std::bind
auto say_hello = std::bind(greet, "Hello", std::placeholders::_1);

// Prefer lambda
auto say_hello = [](const std::string& name) { greet("Hello", name); };
```

Lambdas offer better readability, compile-time type checking, and often generate more optimized code. `std::bind` still appears in legacy codebases, but for new code, lambdas are the idiomatic choice.