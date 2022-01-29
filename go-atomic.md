---
title: golang atomic原子操作
---

# 原子操作(atomic包)

### 原子操作

代码中的加锁操作因为涉及内核态的上下文切换会比较耗时、代价比较高。针对基本数据类型我们还可以使用原子操作来保证并发安全，因为原子操作是Go语言提供的方法它在用户态就可以完成，因此性能比加锁操作更好。Go语言中原子操作由内置的标准库sync/atomic提供。

### atomic包

| 方法                                                         | 解释           |
| ------------------------------------------------------------ | -------------- |
| func LoadInt32(addr `*int32`) (val int32) <br>func LoadInt64(addr `*int64`) (val int64)<br>func LoadUint32(addr`*uint32`) (val uint32)<br>func LoadUint64(addr`*uint64`) (val uint64)<br>func LoadUintptr(addr`*uintptr`) (val uintptr)<br>func LoadPointer(addr`*unsafe.Pointer`) (val unsafe.Pointer) | 读取操作       |
| func StoreInt32(addr `*int32`, val int32) <br>func StoreInt64(addr `*int64`, val int64) <br>func StoreUint32(addr `*uint32`, val uint32) <br>func StoreUint64(addr `*uint64`, val uint64) <br>func StoreUintptr(addr `*uintptr`, val uintptr) <br>func StorePointer(addr `*unsafe.Pointer`, val unsafe.Pointer) | 写入操作       |
| func AddInt32(addr `*int32`, delta int32) (new int32) <br>func AddInt64(addr `*int64`, delta int64) (new int64) <br>func AddUint32(addr `*uint32`, delta uint32) (new uint32) <br>func AddUint64(addr `*uint64`, delta uint64) (new uint64) <br>func AddUintptr(addr `*uintptr`, delta uintptr) (new uintptr) | 修改操作       |
| func SwapInt32(addr `*int32`, new int32) (old int32) <br>func SwapInt64(addr `*int64`, new int64) (old int64) <br>func SwapUint32(addr `*uint32`, new uint32) (old uint32) <br>func SwapUint64(addr `*uint64`, new uint64) (old uint64) <br>func SwapUintptr(addr `*uintptr`, new uintptr) (old uintptr) <br>func SwapPointer(addr `*unsafe.Pointer`, new unsafe.Pointer) (old unsafe.Pointer) | 交换操作（初始化）       |
| func CompareAndSwapInt32(addr `*int32`, old, new int32) (swapped bool) <br>func CompareAndSwapInt64(addr `*int64`, old, new int64) (swapped bool) <br>func CompareAndSwapUint32(addr `*uint32`, old, new uint32) (swapped bool) <br>func CompareAndSwapUint64(addr `*uint64`, old, new uint64) (swapped bool) <br>func CompareAndSwapUintptr(addr `*uintptr`, old, new uintptr) (swapped bool) <br>func CompareAndSwapPointer(addr `*unsafe.Pointer`, old, new unsafe.Pointer) (swapped bool) | 比较并交换操作(相等进行交换) |

### 增或减

　　 顾名思义，原子增或减即可实现对被操作值的增大或减少。因此该操作只能操作数值类型。

　　 被用于进行增或减的原子操作都是以“Add”为前缀，并后面跟针对具体类型的名称。

```go
//方法源码
func AddUint32(addr *uint32, delta uint32) (new uint32)
```

#### 增

栗子：（在原来的基础上加n）

```css
atomic.AddUint32(&addr,n)
```

#### 减

栗子：(在原来的基础上加n（n为负数))

```go
atomic.AddUint32(&addr,uint32(int32(n)))
//或
atomic.AddUint32(&addr,^uint32(-n-1))
```

### 比较并交换

　　 比较并交换----Compare And Swap 简称CAS

　　 **他是假设被操作的值未曾被改变（即与旧值相等），并一旦确定这个假设的真实性就立即进行值替换**

　　 如果想安全的并发一些类型的值，我们总是应该优先使用CAS

```go
//方法源码
func CompareAndSwapInt32(addr *int32, old, new int32) (swapped bool)
```

栗子：**（如果addr和old相同,就用new代替addr）**

```go
ok:=atomic.CompareAndSwapInt32(&addr,old,new)
```

### 载入

　　 如果一个写操作未完成，有一个读操作就已经发生了，这样读操作使很糟糕的。

　　 为了原子的读取某个值sync/atomic代码包同样为我们提供了一系列的函数。这些函数都以"Load"为前缀，意为载入。

```go
//方法源码
func LoadInt32(addr *int32) (val int32)
```

#### 栗子

```kotlin
fun addValue(delta int32){
    for{
        v:=atomic.LoadInt32(&addr)
        if atomic.CompareAndSwapInt32(&v,addr,(delta+v)){
            break;
        }
    }
}
```

### 存储

　　 与读操作对应的是写入操作，sync/atomic也提供了与原子的值载入函数相对应的原子的值存储函数。这些函数的名称均以“Store”为前缀

　　 在原子的存储某个值的过程中，任何cpu都不会进行针对进行同一个值的读或写操作。如果我们把所有针对此值的写操作都改为原子操作，那么就不会出现针对此值的读操作读操作因被并发的进行而读到修改了一半的情况。

　　 原子操作总会成功，因为他不必关心被操作值的旧值是什么。

```go
//方法源码
func StoreInt32(addr *int32, val int32)
```

#### 栗子

```css
atomic.StoreInt32(被操作值的指针,新值)
atomic.StoreInt32(&value,newaddr)
```

### 交换

　　 原子交换操作，这类函数的名称都以“Swap”为前缀。

　　 与CAS不同，交换操作直接赋予新值，不管旧值。

　　 会返回旧值

```go
//方法源码
func SwapInt32(addr *int32, new int32) (old int32)
```

#### 栗子

```csharp
atomic.SwapInt32(被操作值的指针,新值)（返回旧值）
oldval：=atomic.StoreInt32(&value,newaddr)
```

sync.atomic 的实现原理大致是**向 CPU 发送对某一个块内存的 LOCK 信号，然后就将此内存块加锁，从而保证了内存块操作的原子性**。这种对 CPU 发送信号对内存加锁的方式，比 sync.Mutex 这种在语言层面对内存加锁的方式更底层，因此也更高效。

除此之外，sync.atomic 避免了加锁、解锁的过程，一行代码就可以完成线程安全操作，也比 sync.Mutex 更简洁，代码可读性更强。



`atomic`会比`mutex`性能好很多呢？作者 [Dmitry Vyukov](https://github.com/dvyukov) 总结了这两者的一个区别：

`Mutex`由**操作系统**实现，而`atomic`包中的原子操作则由**底层硬件**直接提供支持。在 CPU 实现的指令集里，有一些指令被封装进了`atomic`包，这些指令在执行的过程中是不允许中断（interrupt）的，因此原子操作可以在`lock-free`的情况下保证并发安全，并且**它的性能也能做到随 CPU 个数的增多而线性扩展。**



