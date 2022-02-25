---
title:  go 匿名函数和闭包
date: 2022-02-21 14:50:31
tags:
---

# [GO 匿名函数和闭包](https://segmentfault.com/a/1190000018689134)


匿名函数：顾名思义就是没有名字的函数。很多语言都有如：java，js,php等，其中js最钟情。匿名函数最大的用途是来模拟块级作用域,避免数据污染的。

今天主要讲一下Golang语言的匿名函数和闭包。

### 匿名函数

示例：

1、

```go
package main

import (
   "fmt"
)
func main() {
   f:=func(){
      fmt.Println("hello world")
   }
   f()//hello world
   fmt.Printf("%T\n", f) //打印 func()
}
```

2、带参数

```go
package main

import (
   "fmt"
)
func main() {
   f:=func(args string){
      fmt.Println(args)
   }
   f("hello world")//hello world
   //或
   (func(args string){
        fmt.Println(args)
    })("hello world")//hello world
    //或
    func(args string) {
        fmt.Println(args)
    }("hello world") //hello world
}
```

3、带返回值

```go
package main

import "fmt"

func main() {
   f:=func()string{
      return "hello world"
   }
   a:=f()
   fmt.Println(a)//hello world
}
```

4、多个匿名函数

```go
package main

import "fmt"

func main() {
   f1,f2:=F(1,2)
   fmt.Println(f1(4))//6
   fmt.Println(f2())//6
}
func F(x, y int)(func(int)int,func()int) {
   f1 := func(z int) int {
      return (x + y) * z / 2
   }

   f2 := func() int {
      return 2 * (x + y)
   }
   return f1,f2
}
```

### 闭包（closure）

闭包：说白了就是函数的嵌套，**内层的函数可以使用外层函数的所有变量，即使外层函数已经执行完毕。**

示例：

1、

```go
package main

import "fmt"

func main() {
    a := Fun()
    b:=a("hello ")
    c:=a("hello ")
    fmt.Println(b)//worldhello 
    fmt.Println(c)//worldhello hello 
}
func Fun() func(string) string {
    a := "world"
    return func(args string) string {
        a += args
        return  a
    }
}
```

2、

```go
package main

import "fmt"

func main() {
   a := Fun()
   d := Fun()
   b:=a("hello ")
   c:=a("hello ")
   e:=d("hello ")
   f:=d("hello ")
   fmt.Println(b)//worldhello
   fmt.Println(c)//worldhello hello
   fmt.Println(e)//worldhello
   fmt.Println(f)//worldhello hello
}
func Fun() func(string) string {
   a := "world"
   return func(args string) string {
      a += args
      return  a
   }
}
```

注意两次调用F()，维护的不是同一个a变量。

3、

```go
package main

import "fmt"

func main() {
   a := F()
   a[0]()//0xc00004c080 3
   a[1]()//0xc00004c080 3
   a[2]()//0xc00004c080 3
}
func F() []func() {
   b := make([]func(), 3, 3)
   for i := 0; i < 3; i++ {
      b[i] = func() {
         fmt.Println(&i,i)
      }
   }
   return b
}
```

闭包通过引用的方式使用外部函数的变量。例中只调用了一次函数F,构成一个闭包，i 在外部函数B中定义，所以闭包维护该变量 i ，a[0]、a[1]、a[2]中的 i 都是闭包中 i 的引用。因此执行,i 的值已经变为3，故再调用a[0]()时的输出是3而不是0。

4、如何避免上面的BUG ，用下面的方法，注意和上面示例对比。

```go
package main

import "fmt"

func main() {
    a := F()
    a[0]() //0xc00000a0a8 0
    a[1]() //0xc00000a0c0 1
    a[2]() //0xc00000a0c8 2
}
func F() []func() {
    b := make([]func(), 3, 3)
    for i := 0; i < 3; i++ {
        b[i] = (func(j int) func() {
            return func() {
                fmt.Println(&j, j)
            }
        })(i)
    }
    return b
}
或者
package main

import "fmt"

func main() {
    a := F()
    a[0]() //0xc00004c080 0
    a[1]() //0xc00004c088 1
    a[2]() //0xc00004c090 2
}
func F() []func() {
    b := make([]func(), 3, 3)
    for i := 0; i < 3; i++ {
        j := i
        b[i] = func() {
            fmt.Println(&j, j)
        }
    }
    return b
}
```

每次 操作仅将匿名函数放入到数组中，但并未执行，并且引用的变量都是 `i`，随着 `i` 的改变匿名函数中的 `i` 也在改变，所以当执行这些函数时，他们读取的都是环境变量 `i` 最后一次的值。**解决的方法就是每次复制变量 `i` 然后传到匿名函数中，让闭包的环境变量不相同。**

5、

```go
package main

import "fmt"

func main() {
   fmt.Println(F())//2
}
func F() (r int) {
   defer func() {
      r++
   }()
   return 1
}
```

输出结果为2，即先执行r=1 ,再执行r++。

### Go 中 defer 和 return 执行的先后顺序

1.  **多个defer的执行顺序为“后进先出”；**
2.  **defer、return、返回值三者的执行逻辑应该是：1） return最先执行，return负责将结果写入返回值中；2） 接着defer开始执行一些收尾工作；3） 最后函数携带当前返回值退出。**

 

如果函数的返回值是无名的（不带命名返回值），则go语言会在执行return的时候会执行一个类似创建一个临时变量作为保存return值的动作，而有名返回值的函数，由于返回值在函数定义的时候已经将该变量进行定义，在执行return的时候会先执行返回值保存操作，而后续的defer函数会改变这个返回值(虽然defer是在return之后执行的，但是由于使用的函数定义的变量，所以执行defer操作后对该变量的修改会影响到return的值



eg1：不带命名返回值的函数

```go
package main
 
import "fmt"
 
func main() {
    fmt.Println("return:", test())// defer 和 return之间的顺序是先返回值, i=0，后defer
}
 
func test() int {//这里返回值没有命名
    var i int
    defer func() {
        i++
        fmt.Println("defer1", i) //作为闭包引用的话，则会在defer函数执行时根据整个上下文确定当前的值。i=2
    }()
    defer func() {
        i++
        fmt.Println("defer2", i) //作为闭包引用的话，则会在defer函数执行时根据整个上下文确定当前的值。i=1
    }()
    return i
}　
```

test() 先返回 i=0

defer2先于defer1执行

输出结果为:

defer2 1

defer1 2

return: 0



eg2：带命名返回值的函数:

```go
package main
 
import "fmt"
 
func main() {
    fmt.Println("return:", test())
}
 
func test() (i int) { //返回值命名i
    defer func() {
        i++
        fmt.Println("defer1", i)
    }()
    defer func() {
        i++
        fmt.Println("defer2", i)
    }()
    return i
}
```

输出结果为:

defer2 1

defer1 2

return: 2

# 理解return 返回值的运行机制:

为了弄清上述两种情况的区别，我们首先要理解return 返回值的运行机制:
return 并非原子操作，**分为赋值，和返回值两步操作**
eg1 : 实际上return 执行了两步操作，因为返回值没有命名，所以
return 默认指定了一个返回值（假设为s），首先将i赋值给s,后续
的操作因为是针对i,进行的，所以不会影响s, 此后因为s不会更新，所以
return s 不会改变
相当于：
var i int
s := i
return s

eg2 : 同上，s 就相当于 命名的变量i, 因为所有的操作都是基于
命名变量i(s),返回值也是i, 所以每一次defer操作，都会更新
返回值i

6、递归函数

还有一种情况就是必须用都闭包，就是递归函数。

```go
package main

import "fmt"

func F(i int) int {
   if i <= 1 {
      return 1
   }
   return i * F(i-1)
}

func main() {
   var i int = 3
   fmt.Println(i, F(i))// 3 6
}
```

7、斐波那契数列(Fibonacci)

这个数列从第3项开始，每一项都等于前两项之和。

```java
package main

import "fmt"

func fibonaci(i int) int {
    if i == 0 {
        return 0
    }
    if i == 1 {
        return 1
    }
    return fibonaci(i-1) + fibonaci(i-2)
}

func main() {
    var i int
    for i = 0; i < 10; i++ {
        fmt.Printf("%d\n", fibonaci(i))
    }
}
```

### 小结：

匿名函数和闭包其实是一回事儿，匿名函数就是闭包。匿名函数给编程带来灵活性的同时也容易产生bug，在使用过程当中要多注意函数的参数，及可接受的参数的问题。