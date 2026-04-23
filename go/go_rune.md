## Rune in Go and Code Point in UTF

### Unicode Code Point

A **code point** is Unicode's term for a single abstract character — just a number assigned to a symbol.

- `'A'` → U+0041 (decimal 65)
- `'ก'` → U+0E01 (decimal 3585)
- `'😀'` → U+1F600 (decimal 128512)

The full Unicode space covers U+0000 through U+10FFFF — over 1.1 million possible code points. UTF-8, UTF-16, UTF-32 are all just different *encoding schemes* for these numbers into bytes.

---

### Rune in Go

A `rune` in Go is simply an alias for `int32`:

```go
type rune = int32
```

It holds **one Unicode code point** as a signed 32-bit integer. That's the entire definition. It exists purely as a semantic signal — when you see `rune`, you know the value represents a *character*, not a raw number.

```go
var r rune = 'ก'   // stores 3585 (0x0E01)
var n int32 = 'ก'  // exactly the same thing
```

---

### The Difference

| Concept | What it is | Where it lives |
|---|---|---|
| **Code point** | Abstract Unicode spec term — a number mapped to a character | Unicode standard |
| **`rune`** | Go's concrete type (`int32`) that *stores* one code point | Go type system |

They refer to the same value, but from different angles:

- "Code point" is the *conceptual identity* of a character in Unicode.
- `rune` is Go's *in-memory representation* of that identity.

Think of it this way: a code point is like a person's ID number in a registry. A `rune` is the `int32` variable in your program that holds that ID number.

---

### Why `rune` Matters — The String Pitfall

Go strings are raw **byte slices** (UTF-8 encoded). Iterating with index gives you bytes, not characters:

```go
s := "กข"  // 2 Thai chars, but 6 bytes in UTF-8 (3 bytes each)

// WRONG — iterates bytes
for i := 0; i < len(s); i++ {
    fmt.Printf("%d ", s[i])  // prints 6 byte values
}

// CORRECT — iterates runes (code points)
for i, r := range s {
    fmt.Printf("index=%d rune=%d char=%c\n", i, r, r)
    // index=0 rune=3585 char=ก
    // index=3 rune=3586 char=ข
}
```

`range` over a string automatically decodes UTF-8 and yields `rune` values. The index jumps by byte width (1–4), not by 1.

---

### Converting String ↔ []rune

```go
s := "สวัสดี"

// string → []rune: each element is one code point
runes := []rune(s)
fmt.Println(len(s))      // 18 (bytes)
fmt.Println(len(runes))  // 6  (characters)

// []rune → string: encodes back to UTF-8
back := string(runes)
```

This is the correct way to do things like `runes[2]` (3rd character) or `len` by character count — especially critical for Thai, CJK, emoji, and any non-ASCII text.

---

### Quick Mental Model

```
Unicode:  'ก'  =  code point U+0E01  =  3585
UTF-8:    3585  →  bytes  0xE0 0xB8 0x81  (3 bytes on wire/disk)
Go rune:  var r rune = 'ก'  →  int32(3585) in memory
```

So `rune` ≈ code point in practice. The distinction is just: *code point* is the Unicode spec concept, `rune` is Go's concrete `int32` type that carries it.