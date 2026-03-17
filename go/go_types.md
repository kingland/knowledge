# Go Types

Go is a statically typed language with a rich set of built-in and composite types, categorized into four main groups: basic types, aggregate types, reference types, and interface types.

## Basic Types

These are the fundamental building blocks of data
1. `Boolean` bool (stores true or false).
2. `Numeric` 
-  **Integers:** Signed (int, int8, int16, int32, int64) and unsigned (uint, uint8, uint16, uint32, uint64). The plain int and uint are platform-dependent in size (either 32 or 64 bits).
-  **Floating-Point:** float32, float64 (for real numbers with a fractional component).
-  **Complex:** complex64, complex128 (for numbers with real and imaginary parts).
-  **Aliases:** byte is an alias for uint8, and rune is an alias for int32 (representing a Unicode code point).
3. `String` string (an immutable sequence of bytes, typically UTF-8 encoded).

## Aggregate Types

These types combine multiple values into a single entity.
1. `Arrays` Fixed-length sequences of elements of the same type.
2. `Structs`  Collections of named fields (variables) which can be of different types, grouped together to form a single record. 

## Reference Types

These types hold references to underlying data structures in memory.

1. `Pointers` Store the memory address of another variable.
2. `Slices` Dynamic-length, flexible segments of an underlying array, offering a powerful way to work with sequences.
3. `Maps` Unordered collections of key-value pairs.
4. `Functions` Functions are first-class citizens in Go and can be treated as types.
5. `Channels` Provide the conduits for communication between concurrent goroutines.

## Interface Types

Interfaces define a set of methods. A type implicitly satisfies an interface if it implements all of its methods. The empty interface (`interface{}`) or `any{}`(as of Go 1.18) can hold values of any type.