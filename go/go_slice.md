## Slices in Go

A **slice** is a dynamically-sized, flexible view into an array. It's one of Go's most important data structures.

---

### Anatomy of a Slice

A slice has three components internally:

```
[ pointer | length | capacity ]
    ↓
[ underlying array: 1 | 2 | 3 | 4 | 5 ]
```

- **Pointer** — points to the first element of the underlying array
- **Length** (`len`) — number of elements in the slice
- **Capacity** (`cap`) — number of elements from the pointer to the end of the underlying array

---

### Creating Slices

```go
// 1. From an array literal
a := [5]int{1, 2, 3, 4, 5}
s := a[1:4]          // s = [2, 3, 4], len=3, cap=4

// 2. Using make (most common)
s := make([]int, 3)       // len=3, cap=3, values=[0,0,0]
s := make([]int, 3, 10)   // len=3, cap=10

// 3. Slice literal
s := []int{1, 2, 3}

// 4. nil slice (zero value)
var s []int          // s == nil, len=0, cap=0
```

---

### Key Operations

```go
s := []int{1, 2, 3}

// Append — may allocate a new array if cap exceeded
s = append(s, 4, 5)

// Copy — copies min(len(dst), len(src)) elements
dst := make([]int, len(s))
copy(dst, s)

// Slicing a slice
sub := s[1:3]        // shares the same underlying array!
```

---

### The Sharing Gotcha ⚠️

Since slices share the underlying array, mutations through one slice affect the other:

```go
a := []int{1, 2, 3, 4, 5}
b := a[1:3]   // b = [2, 3], shares a's array

b[0] = 99     // modifies a too!
fmt.Println(a) // [1, 99, 3, 4, 5]
```

To avoid this, use `copy()` or the **full slice expression** `a[low:high:max]` to limit capacity:

```go
b := a[1:3:3]   // cap=2, so append won't touch a's memory
```

---

### Growth Behavior

When `append` exceeds capacity, Go allocates a **new, larger array** and copies data over:

```go
s := make([]int, 0, 2)
s = append(s, 1, 2)    // len=2, cap=2 — fits
s = append(s, 3)        // len=3, cap=4 — new array allocated! (doubled)
```

> Go's growth strategy roughly doubles capacity for small slices and grows by ~25% for larger ones (exact behavior varies by Go version).

---

### Nil vs Empty Slice

```go
var nilSlice []int          // nil == true,  len=0
emptySlice := []int{}       // nil == false, len=0
emptySlice2 := make([]int, 0) // nil == false, len=0
```

Both behave the same for `append`, `len`, `range` — but differ in JSON serialization (`nil` → `null`, empty → `[]`).

---

### 2D Slices

```go
matrix := make([][]int, 3)
for i := range matrix {
    matrix[i] = make([]int, 4)
}
```

---

### Quick Reference

| Operation | Syntax |
|---|---|
| Declare | `var s []T` |
| Make | `make([]T, len, cap)` |
| Append | `s = append(s, val)` |
| Copy | `copy(dst, src)` |
| Length | `len(s)` |
| Capacity | `cap(s)` |
| Sub-slice | `s[low:high]` |
| Delete element i | `s = append(s[:i], s[i+1:]...)` |

The most important thing to remember: **slices are reference types** — passing a slice to a function shares the underlying array, but `append` may or may not affect the original depending on capacity.