---
title: go 数组和切片
date: 2022-01-24 15:50:31
tags:
---


Go语言的`array`和`slice`不同点，今天我们就从底层触发。

# 数组

几乎所有计算机语言，数组的实现都是相似的：一段连续的内存，Go语言也一样，Go语言的数组底层实现就是一段连续的内存空间。每个元素有唯一一个索引(或者叫`下标`)来访问。

数组是具有相同唯一类型的一组已编号且长度固定的数据项序列，这种类型可以是任意的原始类型例如整型、字符串或者自定义类型。

如下图所示，数组的内部实现逻辑图:

![img](https://raw.githubusercontent.com/ycchildcoder/markdown/main/goarray.png)

由于内存连续，CPU很容易计算索引(即数组的`下标`)，可以快速迭代数组里的所有元素。
Go语言的数组不同于C语言或者其他语言的数组，**C语言的数组变量是指向数组第一个元素的指针；而Go语言的数组是一个值，Go语言中的数组是值类型，一个数组变量就表示着整个数组，意味着Go语言的数组在传递的时候，传递的是原数组的拷贝**。你可以理解为Go语言的数组是一种有序的`struct`

# slice

Go 数组的长度不可改变，在特定场景中这样的集合就不太适用，Go 中提供了一种灵活，功能强悍的内置类型切片("动态数组")，与数组相比切片的长度是不固定的，可以追加元素，在追加时可能使切片的容量增大。

切片是一个很小的对象，是对数组进行了抽象，并提供相关的操作方法。**切片有三个属性字段：长度、容量和指向数组的指针**。

![image](https://raw.githubusercontent.com/ycchildcoder/markdown/main/slice_1011.png)

上图中，`ptr`指的是指向array的pointer，`len`是指切片的长度, `cap`指的是切片的容量。现在，我想你对数组和切片有了一个本质的认识。

## 切片有多种声明方式，每种初始化方式对应的逻辑图是怎样的呢？

### 对于`s := make([]byte, 5)`和`s := []byte{...}`的方式

![image](https://raw.githubusercontent.com/ycchildcoder/markdown/main/slice200.png)

### 对于`s = s[2:4]`的方式

![image](https://raw.githubusercontent.com/ycchildcoder/markdown/main/slice300.png)

len=2， cap=3。 共用之前的切片，内存没有重新分配。

### 对于`nil`的切片即`var s []byte`对应的逻辑图是

<img src="https://raw.githubusercontent.com/ycchildcoder/markdown/main/slice400.png" alt="image" style="zoom: 50%;" />

在此有一个说明：`nil`切片和`空`切片是不太一样的，空切片即`s := make([]byte, 0)`或者`s := []byte{}`出来的切片
空切片的逻辑图为：

<img src="https://raw.githubusercontent.com/ycchildcoder/markdown/main/slice500.png" alt="image" style="zoom:50%;" />

**空切片指针不为nil，而nil切片指针为nil。但是，不管是空切片还是nil切片，对其调用内置函数`append()`、`len`和`cap`的效果都是一样的，感受不到任何区别。**

## 扩容

slice这种数据结构便于使用和管理数据集合，可以理解为是一种“动态数组”，`slice`也是围绕动态数组的概念来构建的。既然是动态数组，那么slice是如何扩容的呢？

请记住以下两条规则：

-   **如果切片的容量小于1024个元素，那么扩容的时候slice的cap就翻番，乘以2**；
-   **一旦元素个数超过1024个元素，增长因子就变成1.25**，即每次增加原来容量的四分之一。
-   **如果扩容之后，还没有触及原数组的容量，那么，切片中的指针指向的位置，就还是原数组**，如果扩容之后，**超过了原数组的容量**，那么，Go就会开辟一块新的内存，把原来的值拷贝过来，这种情况丝毫不会影响到原数组。

知道了一下规则，请看下面程序,试问输出结果：

```go
package main

import (
	"fmt"
)

func main() {
	arr := [4]int{10, 20, 30, 40}
	slice := arr[0:2] // 左闭右开  10,20
	testSlice1 := slice
	testSlice2 := append(append(append(slice, 1), 2), 3) // 10,20,1,2，-> 扩容重新分配内存，翻倍 10,20,1,2,3 cap=8
	// 旧切片调整为10,20
	slice[0] = 11 // slice 还是旧切片， 11,20
	fmt.Println(len(slice), cap(slice), slice)
	fmt.Println(len(testSlice1), cap(testSlice1), testSlice1)
	fmt.Println(len(testSlice2), cap(testSlice2), testSlice2)
	fmt.Println(testSlice1[0])
	fmt.Println(testSlice2[0])
}
```

输出：

```
2 4 [11 20]
2 4 [11 20]
5 8 [10 20 1 2 3]
11
10

```