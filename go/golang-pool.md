---
title: Go sync.Pool
date: 2022-01-29 16:41:46
tags:
---



# 是什么

`sync.Pool` 是 sync 包下的一个组件，可以作为保存临时取还对象的一个“池子”。个人觉得它的名字有一定的误导性，因为 Pool 里装的对象可以被无通知地被回收，可能 `sync.Cache` 是一个更合适的名字。

# 有什么用

对于很多需要重复分配、回收内存的地方，`sync.Pool` 是一个很好的选择。频繁地分配、回收内存会给 GC 带来一定的负担，严重的时候会引起 CPU 的毛刺，而 `sync.Pool` 可以将暂时不用的对象缓存起来，待下次需要的时候直接使用，不用再次经过内存分配，复用对象的内存，减轻 GC 的压力，提升系统的性能。

# 怎么用

首先，`sync.Pool` 是协程安全的，这对于使用者来说是极其方便的。使用前，设置好对象的 `New` 函数，用于在 `Pool` 里没有缓存的对象时，创建一个。之后，在程序的任何地方、任何时候仅通过 `Get()`、`Put()` 方法就可以取、还对象了。

下面是 2018 年的时候，《Go 夜读》上关于 `sync.Pool` 的分享，关于适用场景：

>   当多个 goroutine 都需要创建同⼀个对象的时候，如果 goroutine 数过多，导致对象的创建数⽬剧增，进⽽导致 GC 压⼒增大。形成 “并发⼤－占⽤内存⼤－GC 缓慢－处理并发能⼒降低－并发更⼤”这样的恶性循环。

>   在这个时候，需要有⼀个对象池，每个 goroutine 不再⾃⼰单独创建对象，⽽是从对象池中获取出⼀个对象（如果池中已经有的话）。

因此关键思想就是对象的复用，避免重复创建、销毁，下面我们来看看如何使用。

## 简单的例子

首先来看一个简单的例子：

```golang
package main
import (
	"fmt"
	"sync"
)

var pool *sync.Pool

type Person struct {
	Name string
}

func initPool() {
	pool = &sync.Pool {
		New: func()interface{} {
			fmt.Println("Creating a new Person")
			return new(Person)
		},
	}
}

func main() {
	initPool()

	p := pool.Get().(*Person)
	fmt.Println("首次从 pool 里获取：", p)

	p.Name = "first"
	fmt.Printf("设置 p.Name = %s\n", p.Name)

	pool.Put(p)

	fmt.Println("Pool 里已有一个对象：&{first}，调用 Get: ", pool.Get().(*Person))
	fmt.Println("Pool 没有对象了，调用 Get: ", pool.Get().(*Person))
}
```

运行结果：

```golang
Creating a new Person
首次从 pool 里获取： &{}
设置 p.Name = first
Pool 里已有一个对象：&{first}，Get:  &{first}
Creating a new Person
Pool 没有对象了，Get:  &{}
```

首先，需要初始化 `Pool`，唯一需要的就是设置好 `New` 函数。当调用 Get 方法时，如果池子里缓存了对象，就直接返回缓存的对象。如果没有存货，则调用 New 函数创建一个新的对象。

另外，我们发现 Get 方法取出来的对象和上次 Put 进去的对象实际上是同一个，Pool 没有做任何“清空”的处理。但我们不应当对此有任何假设，因为在实际的并发使用场景中，无法保证这种顺序，最好的做法是在 Put 前，将对象清空。

## fmt 包如何用

这部分主要看 `fmt.Printf` 如何使用：

```golang
func Printf(format string, a ...interface{}) (n int, err error) {
	return Fprintf(os.Stdout, format, a...)
}
```

继续看 `Fprintf`：

```golang
func Fprintf(w io.Writer, format string, a ...interface{}) (n int, err error) {
	p := newPrinter()
	p.doPrintf(format, a)
	n, err = w.Write(p.buf)
	p.free()
	return
}
```

`Fprintf` 函数的参数是一个 `io.Writer`，`Printf` 传的是 `os.Stdout`，相当于直接输出到标准输出。这里的 `newPrinter` 用的就是 Pool：

```golang
// newPrinter allocates a new pp struct or grabs a cached one.
func newPrinter() *pp {
	p := ppFree.Get().(*pp)
	p.panicking = false
	p.erroring = false
	p.wrapErrs = false
	p.fmt.init(&p.buf)
	return p
}

var ppFree = sync.Pool{
	New: func() interface{} { return new(pp) },
}
```

回到 `Fprintf` 函数，拿到 pp 指针后，会做一些 format 的操作，并且将 p.buf 里面的内容写入 w。最后，调用 free 函数，将 pp 指针归还到 Pool 中：

```golang
// free saves used pp structs in ppFree; avoids an allocation per invocation.
func (p *pp) free() {
	if cap(p.buf) > 64<<10 {
		return
	}

	p.buf = p.buf[:0]
	p.arg = nil
	p.value = reflect.Value{}
	p.wrappedErr = nil
	ppFree.Put(p)
}
```

归还到 Pool 前将对象的一些字段清零，这样，通过 Get 拿到缓存的对象时，就可以安全地使用了。

## pool_test

通过 test 文件学习源码是一个很好的途径，因为它代表了“官方”的用法。更重要的是，测试用例会故意测试一些“坑”，学习这些坑，也会让自己在使用的时候就能学会避免。

`pool_test` [文件](https://github.com/golang/go/blob/release-branch.go1.14/src/sync/pool_test.go)里共有 7 个测试，4 个 BechMark。

`TestPool` 和 `TestPoolNew` 比较简单，主要是测试 Get/Put 的功能。我们来看下 `TestPoolNew`：

```golang
func TestPoolNew(t *testing.T) {
	// disable GC so we can control when it happens.
	defer debug.SetGCPercent(debug.SetGCPercent(-1))

	i := 0
	p := Pool{
		New: func() interface{} {
			i++
			return i
		},
	}
	if v := p.Get(); v != 1 {
		t.Fatalf("got %v; want 1", v)
	}
	if v := p.Get(); v != 2 {
		t.Fatalf("got %v; want 2", v)
	}

	// Make sure that the goroutine doesn't migrate to another P
	// between Put and Get calls.
	Runtime_procPin()
	p.Put(42)
	if v := p.Get(); v != 42 {
		t.Fatalf("got %v; want 42", v)
	}
	Runtime_procUnpin()

	if v := p.Get(); v != 3 {
		t.Fatalf("got %v; want 3", v)
	}
}
```

首先设置了 `GC=-1`，作用就是停止 GC。那为啥要用 defer？函数都跑完了，还要 defer 干啥。注意到，`debug.SetGCPercent` 这个函数被调用了两次，而且这个函数返回的是上一次 GC 的值。因此，defer 在这里的用途是还原到调用此函数之前的 GC 设置，也就是恢复现场。

接着，调置了 Pool 的 New 函数：直接返回一个 int，变且每次调用 New，都会自增 1。然后，连续调用了两次 Get 函数，因为这个时候 Pool 里没有缓存的对象，因此每次都会调用 New 创建一个，所以第一次返回 1，第二次返回 2。

然后，调用 `Runtime_procPin()` 防止 goroutine 被强占，目的是保护接下来的一次 Put 和 Get 操作，使得它们操作的对象都是同一个 P 的“池子”。并且，这次调用 Get 的时候并没有调用 New，因为之前有一次 Put 的操作。

最后，再次调用 Get 操作，因为没有“存货”，因此还是会再次调用 New 创建一个对象。



## GC

对于 Pool 而言，并不能无限扩展，否则对象占用内存太多了，会引起内存溢出。

>   几乎所有的池技术中，都会在某个时刻清空或清除部分缓存对象，那么在 Go 中何时清理未使用的对象呢？

答案是 GC 发生时。

在 pool.go 文件的 init 函数里，注册了 GC 发生时，如何清理 Pool 的函数：

```golang
// src/sync/pool.go

func init() {
	runtime_registerPoolCleanup(poolCleanup)
}
```

编译器在背后做了一些动作：

```golang
// src/runtime/mgc.go

// Hooks for other packages

var poolcleanup func()

// 利用编译器标志将 sync 包中的清理注册到运行时
//go:linkname sync_runtime_registerPoolCleanup sync.runtime_registerPoolCleanup
func sync_runtime_registerPoolCleanup(f func()) {
	poolcleanup = f
}
```

具体来看下：

```golang
func poolCleanup() {
	for _, p := range oldPools {
		p.victim = nil
		p.victimSize = 0
	}

	// Move primary cache to victim cache.
	for _, p := range allPools {
		p.victim = p.local
		p.victimSize = p.localSize
		p.local = nil
		p.localSize = 0
	}

	oldPools, allPools = allPools, nil
}
```

`poolCleanup` 会在 STW 阶段被调用。整体看起来，比较简洁。主要是将 local 和 victim 作交换，这样也就不致于让 GC 把所有的 Pool 都清空了，有 victim 在“兜底”。