---
title: Go sync.Pool
date: 2022-01-29 16:41:46
tags:
---



# 是什么

`sync.Pool` 是 sync 包下的一个组件，可以作为保存临时取还对象的一个“池子”。个人觉得它的名字有一定的误导性，因为 Pool 里装的对象可以被无通知地被回收，可能 `sync.Cache` 是一个更合适的名字。

# 有什么用

对于很多需要重复分配、回收内存的地方，`sync.Pool` 是一个很好的选择。因为`Go`的gc算法是根据标记清除改进的三色标记法，频繁地分配、回收内存会给 GC 带来一定的负担，严重的时候会引起 CPU 的毛刺，而 `sync.Pool` 可以将暂时不用的对象缓存起来，待下次需要的时候直接使用，不用再次经过内存分配，复用对象的内存，减轻 GC 的压力，提升系统的性能。

当然需要注意的是：**存储在`Pool`中的对象随时都可能在不被通知的情况下被移除。所以并不是所有频繁使用、创建昂贵的对象都适用，比如DB连接、线程池**。

# 区别

为了缓解GC压力，go标准库在sync包中提供了一个Pool，但是这个Pool和我们一般意义上的Pool不太一样，主要有以下几点区别:
 1.Pool无法设置大小，所以理论上只受限于系统内存大小。
 2.Pool中的对象不支持自定义过期时间及策略，究其原因，Pool并不是一个Cache.
 3.Pool的设计初衷是为了缓解GC压力，所以Pool中的对象会在GC开始前全部清除。
 下面这段注释来源于pool.go:

```go
// A Pool is a set of temporary objects that may be individually saved and
// retrieved.
//
// Any item stored in the Pool may be removed automatically at any time without
// notification. If the Pool holds the only reference when this happens, the
// item might be deallocated.
//
// A Pool is safe for use by multiple goroutines simultaneously.
//
// Pool's purpose is to cache allocated but unused items for later reuse,
// relieving pressure on the garbage collector. That is, it makes it easy to
// build efficient, thread-safe free lists. However, it is not suitable for all
// free lists.
```

# 怎么用

首先，`sync.Pool` 是协程安全的，这对于使用者来说是极其方便的。使用前，设置好对象的 `New` 函数，用于在 `Pool` 里没有缓存的对象时，创建一个。之后，在程序的任何地方、任何时候仅通过 `Get()`、`Put()` 方法就可以取、还对象了。

下面是 2018 年的时候，《Go 夜读》上关于 `sync.Pool` 的分享，关于适用场景：

>   当多个 goroutine 都需要创建同⼀个对象的时候，如果 goroutine 数过多，导致对象的创建数⽬剧增，进⽽导致 GC 压⼒增大。形成 “并发⼤－占⽤内存⼤－GC 缓慢－处理并发能⼒降低－并发更⼤”这样的恶性循环。

>   在这个时候，需要有⼀个对象池，每个 goroutine 不再⾃⼰单独创建对象，⽽是从对象池中获取出⼀个对象（如果池中已经有的话）。

因此关键思想就是对象的复用，避免重复创建、销毁，下面我们来看看如何使用。

1.New

Pool struct 包含一个 New 字段，这个字段的类型是函数 func() interface{}。当调用 Pool 的 Get 方法从池中获取元素，没有更多的空闲元素可返回时，就会调用这个 New 方法来创建新的元素。如果你没有设置 New 字段，没有更多的空闲元素可返回时，Get 方法将返回 nil，表明当前没有可用的元素。

2.Get

如果调用这个方法，就会从 Pool取走一个元素，这也就意味着，这个元素会从 Pool 中移除，返回给调用者。不过，除了返回值是正常实例化的元素，Get 方法的返回值还可能会是一个 nil（Pool.New 字段没有设置，又没有空闲元素可以返回），所以你在使用的时候，可能需要判断。

3.Put

这个方法用于将一个元素返还给 Pool，Pool 会把这个元素保存到池中，并且可以复用。但如果 Put 一个 nil 值，Pool 就会忽略这个值。





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

### bytes.Buffer

再看一个例子这个例子使用bytes.Buffer

```go
GO
var bufferPool = sync.Pool{
	New: func() interface{} {
		return &bytes.Buffer{}
	},
}

var data = make([]byte, 10000)

func BenchmarkBufferWithPool(b *testing.B) {
	for n := 0; n < b.N; n++ {
		buf := bufferPool.Get().(*bytes.Buffer)
		buf.Write(data)
		buf.Reset()
		bufferPool.Put(buf)
	}
}

func BenchmarkBuffer(b *testing.B) {
	for n := 0; n < b.N; n++ {
		var buf bytes.Buffer
		buf.Write(data)
	}
}
```

但在实际使用的时候，最好做到物尽其用，尽可能不浪费，我们可以将 buffer 池分成几层。首先，小于 512 byte 的元素的 buffer 占一个池子；其次，小于 1K byte 大小的元素占一个池子；再次，小于 4K byte 大小的元素占一个池子。这样分成几个池子以后，就可以根据需要，到所需大小的池子中获取 buffer 了。

**这个例子中要注意一个坑：**

取出来的 bytes.Buffer 在使用的时候，我们可以往这个元素中增加大量的 byte 数据，这会导致底层的 byte slice 的容量可能会变得很大。这个时候，即使 Reset 再放回到池子中，这些 byte slice 的容量不会改变，所占的空间依然很大。而且，因为 Pool 回收的机制，这些大的 Buffer 可能不被回收，而是会一直占用很大的空间，这属于内存泄漏的问题。

比如 encoding、json 就出现了类似的问题：将容量已经变得很大的 Buffer 再放回 Pool 中，导致内存泄漏。

**解决方法是：在元素放回时，增加了检查逻辑，改成放回的超过一定大小的 buffer，就直接丢弃掉，不再PUT到池子中。**



sync.Pool 的数据结构：

![111](https://raw.githubusercontent.com/ycchildcoder/markdown/main/111.jpg)

![image-20220209141430465](https://raw.githubusercontent.com/ycchildcoder/markdown/main/image-20220209141430465.png)

![image-20220209141917178](https://raw.githubusercontent.com/ycchildcoder/markdown/main/image-20220209141917178.png)

![image-20220209142011469](https://raw.githubusercontent.com/ycchildcoder/markdown/main/image-20220209142011469.png)

![image-20220209142143406](https://raw.githubusercontent.com/ycchildcoder/markdown/main/image-20220209142143406.png)

![image-20220209142451160](https://raw.githubusercontent.com/ycchildcoder/markdown/main/image-20220209142451160.png)

### getSlow

如果在 shared 里没有获取到缓存对象，则继续调用 `Pool.getSlow()`，尝试从其他 P 的 poolLocal 偷取：

```golang
func (p *Pool) getSlow(pid int) interface{} {
	// See the comment in pin regarding ordering of the loads.
	size := atomic.LoadUintptr(&p.localSize) // load-acquire
	locals := p.local                        // load-consume
	// Try to steal one element from other procs.
	// 从其他 P 中窃取对象
	for i := 0; i < int(size); i++ {
		l := indexLocal(locals, (pid+i+1)%int(size))
		if x, _ := l.shared.popTail(); x != nil {
			return x
		}
	}

	// 尝试从victim cache中取对象。这发生在尝试从其他 P 的 poolLocal 偷去失败后，
	// 因为这样可以使 victim 中的对象更容易被回收。
	size = atomic.LoadUintptr(&p.victimSize)
	if uintptr(pid) >= size {
		return nil
	}
	locals = p.victim
	l := indexLocal(locals, pid)
	if x := l.private; x != nil {
		l.private = nil
		return x
	}
	for i := 0; i < int(size); i++ {
		l := indexLocal(locals, (pid+i)%int(size))
		if x, _ := l.shared.popTail(); x != nil {
			return x
		}
	}

	// 清空 victim cache。下次就不用再从这里找了
	atomic.StoreUintptr(&p.victimSize, 0)

	return nil
}
```

![image-20220209143030369](https://raw.githubusercontent.com/ycchildcoder/markdown/main/image-20220209143030369.png)

![image-20220209143148206](https://raw.githubusercontent.com/ycchildcoder/markdown/main/image-20220209143148206.png)





![1112](https://raw.githubusercontent.com/ycchildcoder/markdown/main/1112.png)

![1113](https://raw.githubusercontent.com/ycchildcoder/markdown/main/1113.png)

![image-20220209143326686](https://raw.githubusercontent.com/ycchildcoder/markdown/main/image-20220209143326686.png)

![1114](https://raw.githubusercontent.com/ycchildcoder/markdown/main/1114.png)



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

## Pool 的数据结构

```go
type Pool struct {  
   // 用于检测 Pool 池是否被 copy，因为 Pool 不希望被 copy。用这个字段可以在 go vet 工具中检测出被 copy（在编译期间就发现问题） 
   noCopy noCopy  // A Pool must not be copied after first use.

   // 实际指向 []poolLocal，数组大小等于 P 的数量；每个 P 一一对应一个 poolLocal
   local     unsafe.Pointer 
   localSize uintptr      // []poolLocal 的大小

   // GC 时，victim 和 victimSize 会分别接管 local 和 localSize；
   // victim 的目的是为了减少 GC 后冷启动导致的性能抖动，让分配对象更平滑；
   victim     unsafe.Pointer 
   victimSize uintptr       

   // 对象初始化构造方法，使用方定义
   New func() interface{}
}
```



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

## 思考

1、先 Put，或者主动 Put 会造成什么？
首先，Put 任意类型的元素都不会报错，因为存储的是 interface{} 对象，且内部没有进行类型的判断和断言。如果 Put 在业务上不做限制，那么 Pool 池中就可能存在各种类型的数据，就导致在 Get 后的代码会非常繁琐（需要进行类型判断，否则有 panic 隐患）。另外，Get 得到的对象是随机的，缓存对象的回收也是随机的，所以先 Put 一个对象根本就没有实际作用。

综上，sync.Pool 本质上可以是一个杂货铺，支持存放任何类型，所以 Get 出来和 Put 进去的对象类型要业务自己把控。

2、如果只调用 Get 不调用 Put 会怎么样？
首先就算不调用 Pool.Put，GC 也会去释放 Get 获取的对象（当没有人去再用它时）。但是只进行 Get 操作的话，就相当于一直在生成新的对象，Pool 池也失去了它最本质的功能。