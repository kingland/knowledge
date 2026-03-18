## Go Channels: Buffered vs Unbuffered

A channel is a pipe for sending data between goroutines. The key difference between buffered and unbuffered is **when the sender blocks**.

---

### Unbuffered Channel

Created with **no capacity** — sender and receiver must **both be ready at the same time** (synchronous handshake).

```go
ch := make(chan int)   // unbuffered
```

```
Goroutine A (sender)          Goroutine B (receiver)
      |                               |
   ch <- 42  ←——— blocks ———————→  x := <-ch
      |          until B is ready    |
   continues                      continues
```

```go
func main() {
    ch := make(chan int)

    go func() {
        fmt.Println("sending...")
        ch <- 42              // blocks until main receives
        fmt.Println("sent!")
    }()

    time.Sleep(time.Second)   // simulate delay
    v := <-ch                 // receiver finally ready
    fmt.Println("received:", v)
}
// Output:
// sending...
// received: 42
// sent!
```

> The sender is **stuck at `ch <- 42`** until the receiver executes `<-ch`. They rendezvous at the channel.

---

### Buffered Channel

Created with a **capacity** — sender only blocks when the buffer is **full**, receiver only blocks when the buffer is **empty**.

```go
ch := make(chan int, 3)   // buffered, capacity = 3
```

```
Buffer: [ _ | _ | _ ]   capacity = 3

Send 1:  [ 1 | _ | _ ]   ← does NOT block
Send 2:  [ 1 | 2 | _ ]   ← does NOT block
Send 3:  [ 1 | 2 | 3 ]   ← does NOT block
Send 4:  BLOCKS — buffer full, waits for a receiver
```

```go
func main() {
    ch := make(chan int, 3)

    ch <- 1   // no goroutine needed — doesn't block
    ch <- 2
    ch <- 3
    // ch <- 4   would deadlock here — buffer full, no receiver

    fmt.Println(<-ch)   // 1
    fmt.Println(<-ch)   // 2
    fmt.Println(<-ch)   // 3
}
```

---

### Side-by-Side Comparison

| | Unbuffered | Buffered |
|---|---|---|
| Created with | `make(chan T)` | `make(chan T, n)` |
| Sender blocks when | receiver not ready | buffer is full |
| Receiver blocks when | sender not ready | buffer is empty |
| Synchronization | strict — both must rendezvous | loose — decoupled |
| Use case | synchronization, guaranteed handoff | throughput, producer/consumer |

---

### Deadlock Scenarios

```go
// ❌ Deadlock — unbuffered, no goroutine to receive
ch := make(chan int)
ch <- 1       // blocks forever, nobody receiving
v := <-ch

// ✅ Fixed — send in goroutine
ch := make(chan int)
go func() { ch <- 1 }()
v := <-ch

// ❌ Deadlock — buffered but overfilled, no receiver
ch := make(chan int, 2)
ch <- 1
ch <- 2
ch <- 3    // buffer full, blocks forever

// ✅ Fixed — receive before sending more
ch <- 1
ch <- 2
<-ch       // drain one slot
ch <- 3    // now fits
```

---

### Closing a Channel

```go
ch := make(chan int, 3)
ch <- 1
ch <- 2
ch <- 3
close(ch)   // signal: no more values will be sent

// Range drains the channel until closed
for v := range ch {
    fmt.Println(v)   // 1, 2, 3
}

// Check if channel is closed
v, ok := <-ch
if !ok {
    fmt.Println("channel closed")
}
```

> Only the **sender** should close a channel. Sending to a closed channel **panics**. Receiving from a closed empty channel returns the zero value.

---

### Producer / Consumer Pattern

Buffered channels naturally decouple producer and consumer speeds:

```go
func producer(ch chan<- int) {
    for i := 0; i < 5; i++ {
        fmt.Println("produced", i)
        ch <- i
    }
    close(ch)
}

func consumer(ch <-chan int) {
    for v := range ch {
        fmt.Println("consumed", v)
        time.Sleep(200 * time.Millisecond)   // slower consumer
    }
}

func main() {
    ch := make(chan int, 3)   // buffer absorbs bursts
    go producer(ch)
    consumer(ch)
}
```

---

### `select` with Channels

```go
ch1 := make(chan string, 1)
ch2 := make(chan string, 1)

ch1 <- "one"
ch2 <- "two"

// select picks whichever case is ready
select {
case v := <-ch1:
    fmt.Println("from ch1:", v)
case v := <-ch2:
    fmt.Println("from ch2:", v)
default:
    fmt.Println("no channel ready")   // non-blocking fallback
}
```

---

### Channel Directionality

Restrict channels to send-only or receive-only in function signatures:

```go
func sendOnly(ch chan<- int) { ch <- 42 }     // can only send
func recvOnly(ch <-chan int) { v := <-ch; _ = v }  // can only receive
func bidirect(ch chan int)   { ch <- 1; <-ch } // both
```

---

### Mental Model

```
Unbuffered:   A ——— must wait for B ———> B       (synchronous)

Buffered:     A ——→ [ □ □ □ ] ——→ B              (asynchronous up to cap)
              A fills the queue, B drains it independently
```

**Rule of thumb:** Start with unbuffered for synchronization guarantees. Use buffered when you need to absorb bursts, limit concurrency, or decouple producer/consumer speeds.