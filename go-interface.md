---
title: golang interface
date: 2022-01-28 17:42:23
tags:
---


golang中的接口分为带方法的接口和空接口。 带方法的接口在底层用iface表示，空接口的底层则是eface表示。下面我们透过底层分别看一下这两种类型的接口原理。

以下是接口的原型：

```
//runtime/runtime2.go

//非空接口
type iface struct {
	tab  *itab
	data unsafe.Pointer
}
type itab struct {
	inter  *interfacetype
	_type  *_type
	link   *itab
	hash   uint32 // copy of _type.hash. Used for type switches.
	bad    bool   // type does not implement interface
	inhash bool   // has this itab been added to hash?
	unused [2]byte
	fun    [1]uintptr // variable sized
}

//******************************

//空接口
type eface struct {
	_type *_type
	data  unsafe.Pointer
}

//========================
//这两个接口共同的字段_type
//========================

//runtime/type.go
type _type struct {
	size       uintptr
	ptrdata    uintptr // size of memory prefix holding all pointers
	hash       uint32
	tflag      tflag
	align      uint8
	fieldalign uint8
	kind       uint8
	alg        *typeAlg
	// gcdata stores the GC type data for the garbage collector.
	// If the KindGCProg bit is set in kind, gcdata is a GC program.
	// Otherwise it is a ptrmask bitmap. See mbitmap.go for details.
	gcdata    *byte
	str       nameOff
	ptrToThis typeOff
}
//_type这个结构体是golang定义数据类型要用的，讲到反射文章的时候在具体讲解这个_type。
复制代码
```

# 1.iface

### 1.1 变量类型是如何转换成接口类型的？

看下方代码:

```
package main
type Person interface {
   run()
}

type xitehip struct {
   age uint8
}
func (o xitehip)run() {
}

func main()  {
   var xh Person = xitehip{age:18}
   xh.run()
}

复制代码
```

xh变量是Person接口类型，那xitehip的struct类型是如何转换成接口类型的呢？ 看一下生成的汇编代码：

```
0x001d 00029 (main.go:13)	PCDATA	$2, $0
0x001d 00029 (main.go:13)	PCDATA	$0, $0
0x001d 00029 (main.go:13)	MOVB	$0, ""..autotmp_1+39(SP)
0x0022 00034 (main.go:13)	MOVB	$18, ""..autotmp_1+39(SP)
0x0027 00039 (main.go:13)	PCDATA	$2, $1
0x0027 00039 (main.go:13)	LEAQ	go.itab."".xitehip,"".Person(SB), AX
0x002e 00046 (main.go:13)	PCDATA	$2, $0
0x002e 00046 (main.go:13)	MOVQ	AX, (SP)
0x0032 00050 (main.go:13)	PCDATA	$2, $1
0x0032 00050 (main.go:13)	LEAQ	""..autotmp_1+39(SP), AX
0x0037 00055 (main.go:13)	PCDATA	$2, $0
0x0037 00055 (main.go:13)	MOVQ	AX, 8(SP)
0x003c 00060 (main.go:13)	CALL	runtime.convT2Inoptr(SB)
0x0041 00065 (main.go:13)	MOVQ	16(SP), AX
0x0046 00070 (main.go:13)	PCDATA	$2, $2
0x0046 00070 (main.go:13)	MOVQ	24(SP), CX
复制代码
```

从汇编发现有个转换函数： runtime.convT2Inoptr(SB) 我们去看一下这个函数的实现：

```
func convT2Inoptr(tab *itab, elem unsafe.Pointer) (i iface) {
        t := tab._type
        if raceenabled {
                raceReadObjectPC(t, elem, getcallerpc(), funcPC(convT2Inoptr))
        }
        if msanenabled {
                msanread(elem, t.size)
        }
        x := mallocgc(t.size, t, false)//为elem申请内存
        memmove(x, elem, t.size)//将elem所指向的数据赋值到新的内存中
        i.tab = tab //设置iface的tab
        i.data = x //设置iface的data
        return
}
复制代码
```

从以上实现我们发现编译器生成的struct原始数据会复制一份，然后将新的数据地址赋值给iface.data从而生成了完整的iface，这样如下原始代码中的xh就转换成了Person接口类型。

```
var xh Person = xitehip{age:18}
```

### 1.2 指针变量类型是如何转换成接口类型的呢？

还是上面的例子只是将

```
var xh Person = xitehip{age:18}
复制代码
```

换成了

```
var xh Person = &xitehip{age:18}
复制代码
```

那指针类型的变量是如何转换成接口类型的呢？ 见下方汇编代码：

```
0x001d 00029 (main.go:13)	PCDATA	$2, $1
0x001d 00029 (main.go:13)	PCDATA	$0, $0
0x001d 00029 (main.go:13)	LEAQ	type."".xitehip(SB), AX
0x0024 00036 (main.go:13)	PCDATA	$2, $0
0x0024 00036 (main.go:13)	MOVQ	AX, (SP)
0x0028 00040 (main.go:13)	CALL	runtime.newobject(SB)
0x002d 00045 (main.go:13)	PCDATA	$2, $1
0x002d 00045 (main.go:13)	MOVQ	8(SP), AX
0x0032 00050 (main.go:13)	MOVB	$18, (AX)
复制代码
```

发现了这个函数：

```
runtime.newobject(SB)
复制代码
```

去看一下具体实现：

```
// implementation of new builtin
// compiler (both frontend and SSA backend) knows the signature
// of this function
func newobject(typ *_type) unsafe.Pointer {
        return mallocgc(typ.size, typ, true)
}
复制代码
```

编译器自动生成了iface并将&xitehip{age:18}创建的对象的地址（通过newobject）赋值给iface.data。就是xitehip这个结构体没有被复制



### 1.4 接口调用规则

把上面的例子添加一个eat()接口方法并实现它（注意这个接口方法的实现的接受者是指针）。

```
package main
type Person interface {
	run()
	eat(string)
}
type xitehip struct {
	age uint8
}
func (o xitehip)run() { // //接收方o是值
}
func (o *xitehip)eat(food string) { //接收方o是指针
}
func main()  {
	var xh Person = &xitehip{age:18} //xh是指针
	xh.eat("ma la xiao long xia!")
	xh.run()
}
复制代码
```

这个例子的xh变量的实际类型是个指针，那它是如何调用非指针方法run的呢？

见下表总结：



| 调用方 | 接收方   | 能否编译 |
| ------ | -------- | -------- |
| 值     | 值       | true     |
| 值     | 指针     | *false*  |
| 指针   | 值       | true     |
| 指针   | 指针     | true     |
| 指针   | 指针和值 | true     |
| 值     | 指针和值 | *false*  |

从上表可以得出如下结论：

>   调用方是值时，只要接收方有指针方法那编译器不允许通过编译。

# 2 eface

空接口相对于非空接口没有了方法列表。

```
type eface struct {
	_type *_type
	data  unsafe.Pointer
}
复制代码
```

第一个属性由itab换成了_type,这个结构体是golang中的变量类型的基础，所以空接口可以指定任意变量类型。

### 2.1 示例：

```
package main

import "fmt"

type xitehip struct {
}
func main()  {
	var a interface{} = xitehip{}
	var b interface{} = &xitehip{}
	fmt.Println(a)
	fmt.Println(b)
}
```


