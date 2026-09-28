+++
date = '2026-09-27'
draft = false
title = 'Memory Allocation in Go'
description = 'How Go decides between the stack and the heap.'
tags = ['go', 'memory', 'performance']
+++

In this post, we'll explore **how Go manages memory**. We'll look into key concepts and run benchmarks to analyze the differences between stack and heap.

First, let's refresh the basics: What is a variable? Basically, a variable is a named location in memory used to store data. Based on this premise, we could ask a few fundamental questions: 

- What happens to data in memory once it's no longer needed?
- Does data persist indefinitely in memory?
- Is every variable allocated to the same memory region?
- How does memory keep track of data types?
- What is the maximum size limit for a variable in memory?

## How Go manages memory

Like many modern languages, Go divides memory allocation into two main regions: the **Stack** and the **Heap**. So, how does Go determine where a variable should be stored? During compilation, Go performs a check (known as _escape analysis_) to determine a variable's lifetime and decide its allocation strategy.

Let's dive into the strategies behind these decisions. The main characteristics of these memory regions are:
### Stack

- In Go, every `goroutine` has its own stack, starting at `2KB` and growing as needed.
- There is no special function to allocate/deallocate data in this region, it is basically just adding or subtracting bytes to/from the Stack Pointer.
- Data only lives as long as the function is executing.
- CPU cache-friendly due to contiguous memory allocation and temporal locality.

### Heap

- This region is shared by all `goroutines`.
- The **runtime** needs to call a special function (`mallocgc`) to allocate memory.
- To deallocate memory, Go depends on the **Garbage Collector** (GC).
- Any operation in this memory region is slower compared to the Stack. 

#### The Real Cost of Heap Allocations

Why should we minimize unnecessary heap allocations?
Every time a variable escapes to the heap, it incurs a double performance penalty:

- Garbage Collector Pressure: The GC must traverse pointer graphs to mark and sweep unused memory. Frequent allocations elevate GC CPU usage and can trigger latency spikes.
- Cache misses and indirection: Heap allocations are dispersed throughout memory, leading to pointer indirection and reduced CPU cache efficiency compared to the contiguous memory layout of the stack
### Escape analysis

As I mentioned, Go uses escape analysis to determine a variable's lifetime. How does it work? If a variable needs to stay alive even after function ends, it escapes to the heap, otherwise, the compiler puts it on the stack. 

Now, let's test these assertions. Go provides several tools, but in this case,  we will use one of them to check memory allocation:

```sh
go build -gcflags="-m"
```

The `-m` flag instructs the compiler to print optimization decisions, specifically showing which variables escape to the heap and why.

We compile our program using this flag to check how many variables escape to the heap. Below, I show some examples to prove this.

#### Pointers

Returning a pointer to a local variable forces that variable to escape to the heap. Here are two code examples that are almost identical, with only minor differences:

```go
func sum(a, b int) *int {
	c := a + b
	return &c
}
```

```bash
$ go build -gcflags="-m"
./main.go:5:2: moved to heap: c
```

```go
func sum(a, b int) int {
	c := a + b
	return c
}
```

```bash
$ go build -gcflags="-m"
$
```

After compiling each program, we can see that only the first example escapes to the heap, Returning a pointer is a common way for a variable to escape to the heap. Why? As I mentioned, the compiler determines that the variable needs to stay alive after the `sum` function ends.

#### Closures

When an anonymous function captures a local variable from its enclosing scope, that variable escapes.

```go
func counter() func() int {
	x := 0
	return func() int {
		x++
		return x
	}
}
```

```bash
$ go build -gcflags="-m"
./main.go:35:2: moved to heap: x
./main.go:36:9: func literal escapes to heap
```

In this case `x` is still alive even after the `counter` function ends. Why? The stack stores function data while the function is running, when the counter finishes, its memory space is released. The problem is: the returned function still needs to update and read the `x` variable in the future. So, the compiler moves this variable to the heap.

#### Goroutines

Sharing local variables across concurrent boundaries causes them to escape to the heap.

```go
func run() {
	for i := 0; i < 3; i++ {
		go func() {
			fmt.Println(i)
		}()
	}
}
```

```bash
$ go build -gcflags="-m"
./main.go:6:6: func literal escapes to heap
```

The `run` function launches three goroutines and exits almost immediately. Once run terminates, its stack frame is gone. Since goroutines execute concurrently and independently, if a goroutine attempts to read `i` after run has returned, that stack-allocated variable would no longer exist. Go detects that `i` is shared across goroutine boundaries with an unpredictable lifetime, forcing `i` to escape to the heap to ensure it remains accessible.

#### Interfaces

Passing concrete values to functions that accept empty interfaces (`interface{}` or `any`) triggers heap allocations.

```go
func Log(v interface{}) {
	fmt.Println(v)
}
```

```bash
$ go build -gcflags="-m"
./main.go:15:10: leaking param: v
./main.go:16:13: ... argument does not escape
./main.go:21:6: v escapes to heap
```

A few interesting things are happening here. We can see different messages from the Go compiler, so let me break each one down:

1. `leaking param: v`: The compiler is telling us that this parameter is being passed through to another function. In this case, `Log` simply passes `v` directly to `fmt.Println`.
2. `... argument does not escape`: After `v` is passed inside `fmt.Println`, the compiler guarantees its lifecycle ends within that execution stack, so the underlying variadic slice wrapper doesn't need to be allocated on the heap.
3. `v escapes to heap`: Because `fmt.Println` accepts empty interfaces (`any`), Go cannot deterministically prove the lifetime of the underlying concrete value across function calls. As a result, it safely allocates the value on the heap.

#### Slices

A very common performance pattern in Go revolves around who owns memory allocation: returning a new slice versus receiving a buffer as a parameter.

##### Returning a new slice (Upward Pointer) 

```go 
func createBuffer(size int) []byte { 
	buf := make([]byte, size) 
	return buf 
}
```

```bash
$ go build -gcflags="-m"
./main.go:4:13: make([]byte, size) escapes to heap
```

Because `createBuffer` creates the slice inside its own stack frame and returns it, that memory must outlive the function execution. The compiler forces it to escape to the heap.

##### Accepting a buffer (Downward Pointer)

```go
func fillBuffer(buf []byte) {
    for i := range buf {
        buf[i] = 1
    }
}

func main() {
    buf := make([]byte, 1024)
    fillBuffer(buf)
}
```

```bash
$ go build -gcflags="-m"
./main.go:4:17: buf does not escape
./main.go:12:13: make([]byte, 1024) does not escape
```

Here, `main` allocates the slice on its own stack and passes it down to `fillBuffer`. Since `main` outlives `fillBuffer`, the memory never leaves the stack.

**Key takeaway:** This design pattern is the backbone of Go's `io.Reader` interface (`Read(p []byte)`). By requiring callers to pass their own buffer rather than returning new slices, Go avoids unnecessary heap allocations in hot loops.

```go
// If the design were like this, EVERY read would allocate memory on the heap:
// Read() ([]byte, error)

// Go's standard library uses this design:
type Reader interface {
	Read(p []byte) (n int, err error)
}
```

## Benchmarks

Now, let's look at stack vs. heap benchmarks to analyze the consequences of heap allocation compared to stack allocation. Every benchmark was produced on `go1.27.1 darwin/arm64`. Run them yourself.


```go
func BenchmarkSumStack(b *testing.B) {
	var sink int
	for i := 0; i < b.N; i++ {
		sink = sumStack(i, i+1)
	}
	_ = sink
}

func BenchmarkSumHeap(b *testing.B) {
	var sink *int
	for i := 0; i < b.N; i++ {
		sink = sumHeap(i, i+1)
	}
	_ = sink
}
```

```bash
$ go test -bench=. -benchmem    
BenchmarkSumStack-8                     1000000000               0.9156 ns/op          0 B/op          0 allocs/op
BenchmarkSumHeap-8                      145620685                7.385 ns/op           8 B/op          1 allocs/op
```

- **Stack Allocation:** `sumStack` executes in under a nanosecond (`0.9156 ns/op`) with zero heap overhead (`0 B/op`, `0 allocs/op`). Memory is managed strictly within the CPU stack frame.
    
- **Heap Allocation:** `sumHeap` forces the integer to escape. This single allocation requires 8 bytes (`8 B/op` for a 64-bit integer) and triggers 1 heap allocation per call (`1 allocs/op`), making execution **~8x slower** due to heap allocation overhead (`mallocgc`).

```go
func BenchmarkBufferStack(b *testing.B) {
	var buf [1024]byte
	b.ResetTimer()
	for i := 0; i < b.N; i++ {
		fillBuffer(buf[:])
	}
}

func BenchmarkBufferHeap(b *testing.B) {
	for i := 0; i < b.N; i++ {
		_ = createBuffer(1024)
	}
}
```

```bash
$ go test -bench=. -benchmem    
BenchmarkBufferStack-8                   3745974               328.1 ns/op             0 B/op          0 allocs/op
BenchmarkBufferHeap-8                    2618264               473.0 ns/op          1024 B/op          1 allocs/op
```

- **Stack (`BenchmarkBufferStack`):** Reusing a preallocated buffer stays entirely on the stack frame, achieving zero allocations (`0 B/op`, `0 allocs/op`). The execution time reflects strictly the work of iterating and writing data.

- **Heap (`BenchmarkBufferHeap`):** Returning a newly created slice forces a `1KB` dynamic allocation on the heap for every single execution (`1024 B/op`, `1 allocs/op`). This allocation overhead makes the operation **~44% slower** and increases overall Garbage Collector pressure.

## Final Thoughts

Escape analysis isn't premature optimization, it's the mental model Go expects you to have.

Once you start thinking in variable lifetimes instead of memory sizes:
* You choose between values and pointers with intention, not habit.
* You design APIs that accept buffers rather than leaking allocations.
* You stop being surprised by your application's memory profile and GC pauses.

Understanding the stack and the heap doesn't mean avoiding heap allocations entirely; it means making them deliberate. 

The next time you wonder where your memory is going, don't guess: **ask the compiler (`-gcflags="-m"`). It always answers.**