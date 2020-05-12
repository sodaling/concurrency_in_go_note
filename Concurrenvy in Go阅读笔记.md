# Concurrenvy in Go阅读笔记

## chapter 1

### Race Conditions

A race condition occurs when two or more operations must execute in the correct
order, but the program has not been written so that this order is guaranteed to be
maintained.
Most of the time, this shows up in what’s called a data race, where one concurrent
operation attempts to read a variable while at some undetermined time another con‐
current operation is attempting to write to the same variable.

> 一个concurrent在读的时候另外一个concurrent在写。

```go
var data int
go func() {
data++
}()
if data == 0 {
fmt.Printf("the value is %v.\n", data)
}
```

Here, lines 3 and 5 are both trying to access the variable data, but there is no guaran‐
tee what order this might happen in. There are three possible outcomes to running
this code。

1. Nothing is printed. In this case, line 3 was executed before line 5.
2. “the value is 0” is printed. In this case, lines 5 and 6 were executed before line 3.
3. “the value is 1” is printed. In this case, line 5 was executed before line 3, but line 3
   was executed before line 6.

> 在代码的第3行和第5行都在尝试接触这个变量。但是并没有保证这个变量的接触顺序。因此会有3种可能性发生。
>
> 1. 没有东西输出：第3行在第5行之前执行。
> 2. 输出0：第5，6行执行在第3行之前。
> 3. 输出1：第5行在第3行之前执行，但是第3行在第6行之前执行。

### Atomicity

The first thing that’s very important is the word “context.” Something may be atomic
in one context, but not another. Operations that are atomic within the context of your
process may not be atomic in the context of the operating system; operations that are
atomic within the context of the operating system may not be atomic within the con‐
text of your machine; and operations that are atomic within the context of your
machine may not be atomic within the context of your application. In other words,
the atomicity of an operation can change depending on the currently defined scope.
This fact can work both for and against you!

> 你的程序操作是原子的，但是是在你程序这个特定的context下。但是在操作系统这个context下你的程序未必是院子的。

**i++**

This is about as simple an example as anyone can contrive, and yet it easily demon‐
strates the concept of atomicity. It may look atomic, but a brief analysis reveals several
operations:

1. Retrieve the value of i .
2. Increment the value of i .
3. Store the value of i .

While each of these operations alone is atomic, the combination of the three may not
be, depending on your context. This reveals an interesting property of atomic opera‐
tions: combining them does not necessarily produce a larger atomic operation. Mak‐
ing the operation atomic is dependent on which context you’d like it to be atomic
within. If your context is a program with no concurrent processes, then this code is
atomic within that context. If your context is a goroutine that doesn’t expose i to
other goroutines, then this code is atomic.

> i++看上去是原子化的，但是实际分了3个动作。只有才`` If your context is a program with no concurrent processes, then this code is
> atomic within that context. If your context is a goroutine that doesn’t expose i to
> other goroutines, then this code is atomic.``的context下，这3个动作组合起来的I++才是原子化的。

### Memory Access Synchronization

```go
var data int
go func() { data++}()
if data == 0 {
fmt.Println("the value is 0.")
} else {
fmt.Printf("the value is %v.\n", data)
}
```

We’ve added an else clause here so that regardless of the value of data we’ll always
get some output. Remember that as it is written, there is a data race and the output of
the program will be completely nondeterministic.
In fact, there’s a name for a section of your program that needs exclusive access to a
shared resource. This is called a critical section. In this example, we have three critical
sections:

1. Our goroutine, which is incrementing the data variables.
2. Our if statement, which checks whether the value of data is 0.
3. Our fmt.Printf statement, which retrieves the value of data for output.

> critical section：程序中需要排他的访问被共享资源的section。

one way to solve this problem is to syn‐
chronize access to the memory between your critical sections. Let’s see what that
looks like.
The following code is not idiomatic Go (and I don’t suggest you attempt to solve your
data race problems like this), but it very simply demonstrates memory access syn‐
chronization.

> 强行同步访问内存。但是这不是golang理想的写法。

```go
var memoryAccess sync.Mutex
var value int
go func() {
memoryAccess.Lock()
value++
memoryAccess.Unlock()
}()
memoryAccess.Lock()
if value == 0 {
fmt.Printf("the value is %v.\n", value)
} else {
fmt.Printf("the value is %v.\n", value)
}
memoryAccess.Unlock()
```

> 其实就是利用sync.Mutex锁上了，利用排它锁来解决data race的问题。

we haven’t actuallysolved our race condition! The order of operations in this program is still nondeter‐
ministic; we’ve just narrowed the scope of the nondeterminism a bit. In this example,
either the goroutine will execute first, or both our if and else blocks will.

It is true that you can solve some problems by synchronizing access to the memory,
but as we just saw, it doesn’t automatically solve data races or logical correctness. Fur‐
ther, it can also create maintenance and performance problems.

By synchronizing access to the memory in this manner,
you are counting on all other developers to follow the same convention now and into the future. That’s a pretty tall order.

> 这种同步的内存访问只是约定，需要保证所有开发者去遵守。

Synchronizing access to the memory in this manner also has performance ramifac‐
tions.

### Deadlocks, Livelocks, and Starvation

**Deadlock**
A deadlocked program is one in which all concurrent processes are waiting on one
another. In this state, the program will never recover without outside intervention.

```go
type value struct {
mu
sync.Mutex
value int
}
var wg sync.WaitGroup
printSum := func(v1, v2 *value) {
defer wg.Done()
v1.mu.Lock()
defer v1.mu.Unlock()
time.Sleep(2*time.Second)
v2.mu.Lock()
defer v2.mu.Unlock()
fmt.Printf("sum=%v\n", v1.value + v2.value)
}
var a, b value
wg.Add(2)
go printSum(&a, &b)
go printSum(&b, &a)
wg.Wait()
```

> 一个锁了a，另一个锁了b。互相等待

#### Livelock

```go
	cadence := sync.NewCond(&sync.Mutex{})
	go func() {
		for range time.Tick(1 * time.Millisecond) {
			cadence.Broadcast()
		}
	}()
	takeStep := func() {
		cadence.L.Lock()
		cadence.Wait()
		cadence.L.Unlock()
	}
	tryDir := func(dirName string, dir *int32, out *bytes.Buffer) bool {
		fmt.Fprintf(out, " %v", dirName)
		atomic.AddInt32(dir, 1)
		takeStep()
		if atomic.LoadInt32(dir) == 1 {
			fmt.Fprint(out, ". Success!")
			return true
		}
		takeStep()
		atomic.AddInt32(dir, -1)
		return false
	}
	var left, right int32
	tryLeft := func(out *bytes.Buffer) bool { return tryDir("left", &left, out) }
	tryRight := func(out *bytes.Buffer) bool { return tryDir("right", &right, out) }
	walk := func(walking *sync.WaitGroup, name string) {
		var out bytes.Buffer
		defer func() { fmt.Println(out.String()) }()
		defer walking.Done()
		fmt.Fprintf(&out, "%v is trying to scoot:", name)
		for i := 0; i < 5; i++ {
			if tryLeft(&out) || tryRight(&out) {
				return
			}
		}
		fmt.Fprintf(&out, "\n%v tosses her hands up in exasperation!", name)
	}
	var peopleInHallway sync.WaitGroup
	peopleInHallway.Add(2)
	go walk(&peopleInHallway, "Alice")
	go walk(&peopleInHallway, "Barbara")
	peopleInHallway.Wait()
```

> 你曾经在走廊走向另一个人吗？她移动到一边让你通过，但你也做了同样的事情。所以你转到另一边，但她也是这样做的。想象一下这个情形永远持续下去，你就明白了活锁。
>
> 两个人都往左，两个人都往右，永远都是碰头，没法继续走下去。

#### Starvation

```go
var wg sync.WaitGroup
var sharedLock sync.Mutex
const runtime = 1*time.Second
greedyWorker := func() {
defer wg.Done()
var count int
for begin := time.Now(); time.Since(begin) <= runtime; {
sharedLock.Lock()
time.Sleep(3*time.Nanosecond)
sharedLock.Unlock()
count++
}
fmt.Printf("Greedy worker was able to execute %v work loops\n", count)
}
politeWorker := func() {
defer wg.Done()
var count int
for begin := time.Now(); time.Since(begin) <= runtime; {
sharedLock.Lock()
time.Sleep(1*time.Nanosecond)
sharedLock.Unlock()
sharedLock.Lock()
time.Sleep(1*time.Nanosecond)
sharedLock.Unlock()
sharedLock.Lock()
time.Sleep(1*time.Nanosecond)
sharedLock.Unlock()
count++
}
    fmt.Printf("Polite worker was able to execute %v work loops.\n", count)
}
wg.Add(2)
go greedyWorker()
go politeWorker()
wg.Wait()
```

> 过度持有资源导致完成工作效率低甚至可能根本完成不了工作。就是因为greedy worker过度持有资源导致完成了更加多。导致完成一样的工作greedy worker需要更加多的资源。