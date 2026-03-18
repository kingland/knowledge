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


## Go Basic Types

Go has a small, well-defined set of built-in types grouped into several categories.

---

### Boolean

```go
var b bool = true
var f bool = false
```

Only `true` or `false`. No implicit conversion from integers (unlike C/C++).

---

### Integer Types

| Type | Size | Range |
|---|---|---|
| `int8` | 8-bit | −128 to 127 |
| `int16` | 16-bit | −32,768 to 32,767 |
| `int32` | 32-bit | −2³¹ to 2³¹−1 |
| `int64` | 64-bit | −2⁶³ to 2⁶³−1 |
| `uint8` | 8-bit | 0 to 255 |
| `uint16` | 16-bit | 0 to 65,535 |
| `uint32` | 32-bit | 0 to 2³²−1 |
| `uint64` | 64-bit | 0 to 2⁶⁴−1 |
| `int` | platform | 32-bit on 32-bit OS, 64-bit on 64-bit OS |
| `uint` | platform | same as `int` |
| `uintptr` | platform | large enough to hold a pointer |

```go
var i int     = 42
var u uint32  = 100
var b byte    = 255    // alias for uint8
var r rune    = 'A'    // alias for int32, represents a Unicode code point
```

> **Rule of thumb:** Use `int` unless you have a specific reason to use a sized type. The compiler won't implicitly convert between `int` and `int32` — you must cast explicitly.

---

### Floating Point

| Type | Size | Precision |
|---|---|---|
| `float32` | 32-bit | ~6–7 decimal digits |
| `float64` | 64-bit | ~15–16 decimal digits |

```go
var f32 float32 = 3.14
var f64 float64 = 3.141592653589793

// Default inferred type is float64
pi := 3.14  // float64
```

---

### Complex Numbers

```go
var c1 complex64  = 1 + 2i
var c2 complex128 = 3.5 + 4.2i

r := real(c2)   // 3.5
i := imag(c2)   // 4.2
```

Rarely used in everyday code, but built into the language for scientific/math work.

---

### String

```go
s := "Hello, 世界"         // UTF-8 encoded, immutable
n := len(s)               // byte count, NOT character count

// Iterate by rune (Unicode code point)
for i, ch := range s {
    fmt.Printf("%d: %c\n", i, ch)
}

// Raw string literal (no escape processing)
raw := `line1
line2\nstill line2`

// String concatenation
s2 := "foo" + "bar"

// Strings are immutable — you cannot do s[0] = 'x'
```

> A `string` in Go is just a read-only slice of bytes. Use `[]rune(s)` when you need to index by character (Unicode), and `[]byte(s)` when you need to mutate.

---

### byte and rune

These are just aliases, but semantically important:

```go
var b byte = 'A'    // uint8  — a raw byte value
var r rune = '界'   // int32  — a Unicode code point

s := "Hi"
fmt.Println(s[0])          // 72 (byte value of 'H')
fmt.Println(string(s[0]))  // "H"
```

---

### Zero Values

Every type has a zero value — Go **never** leaves memory uninitialized:

| Type | Zero Value |
|---|---|
| `bool` | `false` |
| integers | `0` |
| floats | `0.0` |
| complex | `0+0i` |
| `string` | `""` (empty string) |
| pointers, slices, maps, channels, functions, interfaces | `nil` |

```go
var i int     // 0
var s string  // ""
var b bool    // false
var p *int    // nil
```

---

### Type Conversion

Go has **no implicit type conversion** — all conversions must be explicit:

```go
var i int    = 42
var f float64 = float64(i)   // must cast
var u uint   = uint(f)

// String conversions
s := string(65)              // "A"  (interprets as rune)
s  = fmt.Sprintf("%d", 65)  // "65" (number to string)
n, _ := strconv.Atoi("42")  // string to int
```

---

### Constants

Constants are typed or untyped and evaluated at compile time:

```go
const Pi = 3.14159          // untyped — adapts to context
const MaxSize int = 100     // typed

// iota for enumerations
type Direction int
const (
    North Direction = iota  // 0
    East                    // 1
    South                   // 2
    West                    // 3
)
```

Untyped constants have higher precision than typed ones and can be used more flexibly across numeric types.

---

### Summary

```
bool
├── true / false

integers
├── int, int8, int16, int32, int64
├── uint, uint8, uint16, uint32, uint64
├── byte (= uint8)
├── rune (= int32)
└── uintptr

floats
├── float32
└── float64

complex
├── complex64
└── complex128

string  →  immutable UTF-8 byte sequence
```

The biggest difference from C++: **no implicit conversions, no undefined behavior from overflow in unsigned types, and every variable always has a defined zero value.**