## `fmt.Sprintf()` in Go

`Sprintf` formats data into a **string** using a format specifier and returns it (unlike `Printf` which prints to stdout).

```go
s := fmt.Sprintf(format string, args ...any) string
```

---

### General Verbs

| Verb | Description | Example |
|---|---|---|
| `%v` | Default format | `{1 true hello}` |
| `%+v` | Struct with field names | `{x:1 ok:true}` |
| `%#v` | Go syntax representation | `main.Point{X:1, Y:2}` |
| `%T` | Type of the value | `int`, `[]string` |
| `%%` | Literal `%` sign | `%` |

```go
type Point struct{ X, Y int }
p := Point{1, 2}

fmt.Sprintf("%v",  p)   // "{1 2}"
fmt.Sprintf("%+v", p)   // "{X:1 Y:2}"
fmt.Sprintf("%#v", p)   // "main.Point{X:1, Y:2}"
fmt.Sprintf("%T",  p)   // "main.Point"
```

---

### Integer Verbs

| Verb | Description | Example output |
|---|---|---|
| `%d` | Decimal | `42` |
| `%b` | Binary | `101010` |
| `%o` | Octal | `52` |
| `%x` | Hex lowercase | `2a` |
| `%X` | Hex uppercase | `2A` |
| `%c` | Character (rune) | `*` |
| `%U` | Unicode format | `U+002A` |

```go
n := 42
fmt.Sprintf("%d", n)   // "42"
fmt.Sprintf("%b", n)   // "101010"
fmt.Sprintf("%x", n)   // "2a"
fmt.Sprintf("%X", n)   // "2A"
fmt.Sprintf("%o", n)   // "52"
fmt.Sprintf("%c", 65)  // "A"
fmt.Sprintf("%U", 'A') // "U+0041"
```

---

### Float Verbs

| Verb | Description | Example output |
|---|---|---|
| `%f` | Decimal point, no exponent | `3.141593` |
| `%e` | Scientific notation lowercase | `3.141593e+00` |
| `%E` | Scientific notation uppercase | `3.141593E+00` |
| `%g` | Shortest of `%e` or `%f` | `3.141592653589793` |
| `%G` | Shortest of `%E` or `%f` | same, uppercase |

```go
f := 3.141592653589793
fmt.Sprintf("%f",   f)   // "3.141593"
fmt.Sprintf("%.2f", f)   // "3.14"       ← 2 decimal places
fmt.Sprintf("%e",   f)   // "3.141593e+00"
fmt.Sprintf("%g",   f)   // "3.141592653589793"
```

---

### String & Byte Verbs

| Verb | Description |
|---|---|
| `%s` | Plain string or `[]byte` |
| `%q` | Double-quoted, escaped string |
| `%x` | Hex encoding of each byte |

```go
s := "hello"
fmt.Sprintf("%s",  s)   // "hello"
fmt.Sprintf("%q",  s)   // "\"hello\""
fmt.Sprintf("%x",  s)   // "68656c6c6f"
fmt.Sprintf("%10s", s)  // "     hello"  ← right-aligned
fmt.Sprintf("%-10s", s) // "hello     "  ← left-aligned
```

---

### Boolean Verb

```go
fmt.Sprintf("%t", true)   // "true"
fmt.Sprintf("%t", false)  // "false"
```

---

### Pointer Verb

```go
x := 42
fmt.Sprintf("%p", &x)   // "0xc0000b4010"  (memory address)
```

---

### Width & Precision Flags

```
%[flags][width][.precision]verb
```

| Flag | Meaning |
|---|---|
| `width` | Minimum field width |
| `.precision` | Decimal places for float; max chars for string |
| `-` | Left-align (default is right-align) |
| `0` | Pad with zeros instead of spaces |
| `+` | Always show sign for numbers |
| ` ` | Space before positive numbers |
| `#` | Alternate form (`0x` for hex, `0` for octal) |

```go
fmt.Sprintf("%8d",    42)     // "      42"   right-align, width 8
fmt.Sprintf("%-8d",   42)     // "42      "   left-align
fmt.Sprintf("%08d",   42)     // "00000042"   zero-pad
fmt.Sprintf("%+d",    42)     // "+42"        force sign
fmt.Sprintf("%8.2f",  3.14)   // "    3.14"   width 8, 2 decimals
fmt.Sprintf("%-8.2f", 3.14)   // "3.14    "   left-aligned float
fmt.Sprintf("%#x",    255)    // "0xff"       hex with prefix
fmt.Sprintf("%#o",    8)      // "010"        octal with prefix
```

---

### Dynamic Width & Precision with `*`

You can pass width/precision as arguments using `*`:

```go
fmt.Sprintf("%*d",    8, 42)      // "      42"
fmt.Sprintf("%.*f",   3, 3.14159) // "3.142"
```

---

### Practical Examples

```go
// Building a log line
level, msg := "ERROR", "disk full"
log := fmt.Sprintf("[%s] %s", level, msg)
// "[ERROR] disk full"

// Padding a table
for _, row := range []struct{ name string; score int }{
    {"Alice", 98}, {"Bob", 75}, {"Charlie", 100},
} {
    fmt.Sprintf("%-10s %3d", row.name, row.score)
    // "Alice       98"
    // "Bob         75"
    // "Charlie    100"
}

// Hex dump
data := []byte{0xDE, 0xAD, 0xBE, 0xEF}
fmt.Sprintf("%x", data)   // "deadbeef"
fmt.Sprintf("% x", data)  // "de ad be ef"  ← space between bytes

// Formatting a memory address style output
fmt.Sprintf("0x%08X", 0x1F4)  // "0x000001F4"
```

---

### `Sprintf` vs siblings

| Function | Destination | Returns |
|---|---|---|
| `fmt.Sprintf` | string | `string` |
| `fmt.Printf` | stdout | `n int, err error` |
| `fmt.Fprintf` | `io.Writer` | `n int, err error` |
| `fmt.Errorf` | error value | `error` |

```go
s   := fmt.Sprintf("value: %d", 42)          // → string
      fmt.Printf("value: %d\n", 42)           // → stdout
      fmt.Fprintf(os.Stderr, "err: %s", msg)  // → stderr
err := fmt.Errorf("failed at step %d", 3)    // → error
```

The key mental model: **`%v` for quick debugging, `%d`/`%f`/`%s` for explicit control, width/precision flags for aligned output.**