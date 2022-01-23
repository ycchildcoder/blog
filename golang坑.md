---
title: golang坑
date: 2022-01-22 15:44:43
tags:
---



Golang是一门语言，它简洁、高效、易学习、开发效率高、还可以编译成机器码…
它毕竟是一门新兴语言，一边学习，一边踩坑。

# 多个`defer`出现的时候，多个`defer`之间按照LIFO（后进先出）的顺序执行

```go
package main

import "fmt"

func main(){
    defer func(){
        fmt.Println("1")
    }()

    defer func(){
        fmt.Println("2")
    }()

    defer func(){
        fmt.Println("3")
    }()


}
```

对应的输出是：

```
3
2
1
```

# 用`for range`来遍历数组或者map的时候，<u>*被遍历的指针是不变的，每次遍历仅执行struct值的拷贝*</u>

```go
import "fmt"

type student struct{
    Name string
    Age  int
}

func main(){
    var stus []student

    stus = []student{
        {Name:"one", Age: 18},
        {Name:"two", Age: 19},
    }

    data := make(map[int]*student)

    for i, v := range stus{
        data[i] = &v   //应该改为：data[i] = &stus[i]
    }

    for i, v := range data{
        fmt.Printf("key=%d, value=%v \n", i,v)
    }
}
```

所以，结果输出为：

```
key=0, value=&{two 19} 
key=1, value=&{two 19}
```

# 不管运行顺序如何，当参数为函数的时候，要先计算参数的值

```go
func main(){
    a := 1
    defer print(function(a))
    a = 2;
}

func function(num int) int{
    return num
}
func print(num int){
    fmt.Println(num)
}
```

输出：

```
1
```

# `make(chan int)` 和 `make(chan int, 1)`是不一样的

`chan`一旦被写入数据后，当前`goruntine`就会被阻塞，知道有人接收才可以（即 “ <- ch”），如果没人接收，它就会一直阻塞着。而如果chan带一个缓冲，就会把数据放到缓冲区中，直到缓冲区满了，才会阻塞

非缓冲channel和缓冲channel

```go
import "fmt"


func main(){
    ch := make(chan int) //改为 ch := make(chan int, 1) 就好了

    ch <- 1

    fmt.Println("success")
}
```

输出：

```
fatal error: all goroutines are asleep - deadlock!
```

# golang 的 select 的功能和 select, poll, epoll 相似， 就是监听 IO 操作，当 IO 操作发生时，触发相应的动作。

select 的代码形式和 switch 非常相似， 不过 select 的 case 里的操作语句只能是”IO操作”（不仅仅是取值`<-channel`，赋值`channel<-`也可以）， select 会一直等待等到某个 case 语句完成，也就是等到成功从channel中读到数据。 则 select 语句结束

```go
 import "fmt"


func main(){
    ch := make(chan int, 1)

    ch <- 1

    select {
    case msg :=<-ch:
        fmt.Println(msg)
    default:
        fmt.Println("default")
    }

    fmt.Println("success")
}
```

输出：

```
1
success
```

`default`可以判断chan是否已经满了

```go
import "fmt"


func main(){
    ch := make(chan int, 1)

    select {
    case msg :=<-ch:
        fmt.Println(msg)
    default:
        fmt.Println("default")
    }

    fmt.Println("success")
}
```

输出:

```
default
success
```

此时因为`ch`中没有写入数据，为空，所以 case不会读取成功。 则 select 执行 default 语句。

# Go语言中不存在未初始化的变量

变量定义基本方式为：

```
var 变量名字 类型 = 表达式
```

其中类型和表达式均可省略，如果初始化表达式被省略，将用零值初始化该变量。

-   数值变量对应的是0值
-   布尔变量对应的是false
-   字符串对应的零值是空字符串
-   接口或者引用类型（包括slice，map，chan）变量对应的是nil
-   数组或者结构体等聚合类型对应的零值是每个元素或字段对应该类型的零值。

```
var s string 
fmt.Println(s) // ""
```

# `:=`注意的问题

-   使用`:=`定义的变量，仅能使用在函数内部。

-   在定义多个变量的时候

    ```
    :=
    ```

    周围不一定是全部都是刚刚声明的，有些可能只是赋值，例如下面的err变量

    ```go
    in, err := os.Open(infile)
    // TODO
    out, err := os.Create(outfile)
    ```

# `new`在Go语言中只是一个预定义的函数，它并不是一个关键字，我们可以将`new`作为变量或者其他

例如:

```
func delta(old, new int) int { 
    return new - old 
}
```

以上是正确的。

# 并不是使用`new`就一定会在堆上分配内存

编译器会自动选择在栈上还是在堆上分配存储空间，但可能令人惊讶的是，这个选择并不是由用`var`还是`new`声明变量的方式决定的。

请看例子:

```go
var global *int 

func f() {
    var x int x=1 
    global = &x
}

func g() {
    y := new(int)
    *y = 1 
}
```

`f()`函数中的`x`就是在堆上分配内存，而`g()`函数中的`y`就是分配在栈上。

# `init`函数在同一个文件中可以包含多个

在同一个包文件中，可以包含有多个`init`函数，**多个`init`函数的执行顺序和定义顺序（导入递归顺序）一致**。

# Golang中没有“对象”

```go
package main

import (
    "fmt"
)
type test struct {
    name string
}
func (t *test) getName(){
    fmt.Println("hello world")
}
func main() {
    var t *test
    t = nil
    t.getName()
}
```

能正常输出吗？会报错吗？

输出为：

```
hello world
```

可以正常输出。Go本质上不是面向对象的语言，Go中是不存在object的含义的，Go语言书籍中的对象也和Java、PHP中的对象有区别，不是真正的”对象”，是Go中struct的实体。

调用getName方法，在Go中还可以转换，转换为：Type.method(t Type, arguments)
所以，以上代码main函数中还可以写成：

```
func main() {
    (*test).getName(nil)
}
```

# Go中的指针`*`符号的含义

&的意思大家都明白的，取地址，假如你想获得一个变量的地址，只需在变量前加上&即可。

例如：

```
a := 1
b := &a
```

现在，我拿到a的地址了，但是我想取得a指针指向的值，该如何操作呢？用`*`号，`*b`即可。
*的意思是对指针取值。

下面对a的值加一

```
a := 1
b := &a
*b++
```

`*`和`&`可以相互抵消，同时注意，`*&`可以抵消，但是`&*`不可以；所以`a`和`*&a`是一样的，和`*&*&*&a`也是一样的。

# `os.Args`获取命令行指令参数，应该从数组的1坐标开始

`os.Args`的第一个元素，`os.Args[0]`, 是命令本身的名字

```go
package main
import (
    "fmt"
    "os"
)
func main() {
    fmt.Println(os.Args[0])
}
```

以上代码，经过`go build`之后，打包成一个可执行文件`main`，然后运行指令`./main 123`

输出：`./main`

# 数组切片slice的容量问题带来的bug

请看下列代码:

```go
import (
    "fmt"
)
func main(){
    array := [4]int{10, 20, 30, 40}
    slice := array[0:2]
    newSlice := append(slice, 50)
    newSlice[1] += 1
    fmt.Println(slice)
}
```

请问输出什么?
答案是:

```
[10 21]
```

如果稍作修改，将以上newSlice改为扩容三次，newSlice := append(append(append(slice, 50), 100), 150)如下:

```go
import (
    "fmt"
)
func main(){
    array := [4]int{10, 20, 30, 40}
    slice := array[0:2]
    newSlice := append(append(append(slice, 50), 100), 150)
    newSlice[1] += 1
    fmt.Println(slice)
}
```

输出为:

```
[10 20]
```

这特么是什么鬼？
这就要从Golang切片的扩容说起了；切片的扩容，就是当切片添加元素时，切片容量不够了，就会扩容，扩容的大小遵循下面的原则：（如果切片的容量小于1024个元素，那么扩容的时候slice的cap就翻番，乘以2；一旦元素个数超过1024个元素，增长因子就变成1.25，即每次增加原来容量的四分之一。）如果扩容之后，还没有触及原数组的容量，那么，切片中的指针指向的位置，就还是原数组（这就是产生bug的原因）；如果扩容之后，超过了原数组的容量，那么，Go就会开辟一块新的内存，把原来的值拷贝过来，这种情况丝毫不会影响到原数组。
建议尽量避免bug的产生。

# `map`引用不存在的key，不报错

请问下面的例子输出什么，会报错吗？

```go
import (
    "fmt"
)

func main(){
    newMap := make(map[string]int)
    fmt.Println(newMap["a"])
}
```

答案是：

```
0
```

不报错。不同于PHP，Golang的map和Java的HashMap类似，Java引用不存在的会返回null，而Golang会返回初始值

```go
if v,ok := newMap["a"];ok {
	// do
}
```

# map使用range遍历顺序问题，并不是录入的顺序，而是随机顺序

请看下面的例子:

```go
import (
    "fmt"
)

func main(){
    newMap := make(map[int]int)
    for i := 0; i < 10; i++{
        newMap[i] = i
    }
    for key, value := range newMap{
        fmt.Printf("key is %d, value is %d\n", key, value)
    }
}
```

输出：

```
key is 1, value is 1
key is 3, value is 3
key is 5, value is 5
key is 7, value is 7
key is 9, value is 9
key is 0, value is 0
key is 2, value is 2
key is 4, value is 4
key is 6, value is 6
key is 8, value is 8
```

是杂乱无章的顺序。map的遍历顺序不固定，这种设计是有意为之的，能为能防止程序依赖特定遍历顺序。

如果想按顺序排序，可以进行key排序，然后再输出。

# channel作为函数参数传递，可以声明为只取(<- chan)或者只发送(chan <-)

一个函数在将channel作为一个类型的参数来声明的时候，可以将channl声明为只可以取值(<- chan)或者只可以发送值(chan <-)，不特殊说明，则既可以取值，也可以发送值。

例如：只可以发送值

```go
func setData(ch chan <- string){
    //TODO
}
```

如果在以上函数中存在<-ch则会编译不通过。

如下是只可以取值:

```go
func setData(ch <- chan  string){
    //TODO
}
```

如果以上函数中存在ch<-则在编译期会报错

# 使用channel时，注意goroutine之间的执行流程问题

```go
package main
import (
    "fmt"
)
func main(){
    ch := make(chan string)
    go setData(ch)
    fmt.Println(<-ch)
    fmt.Println(<-ch)
    fmt.Println(<-ch)
    fmt.Println(<-ch)
    fmt.Println(<-ch)
}
func setData(ch  chan  string){
    ch <- "test"
    ch <- "hello wolrd"
    ch <- "123"
    ch <- "456"
    ch <- "789"
}
```

以上代码的执行流程是怎样的呢？
一个基于无缓存channel的发送或者取值操作，会导致当前goroutine阻塞，一直等待到另外的一个goroutine做相反的取值或者发送操作以后，才会正常跑。
以上例子中的流程是这样的：

主goroutine等待接收，另外的那一个goroutine发送了“test”并等待处理；完成通信后，打印出”test”；两个goroutine各自继续跑自己的。
主goroutine等待接收，另外的那一个goroutine发送了“hello world”并等待处理；完成通信后，打印出”hello world”；两个goroutine各自继续跑自己的。
主goroutine等待接收，另外的那一个goroutine发送了“123”并等待处理；完成通信后，打印出”123”；两个goroutine各自继续跑自己的。
主goroutine等待接收，另外的那一个goroutine发送了“456”并等待处理；完成通信后，打印出”456”；两个goroutine各自继续跑自己的。
主goroutine等待接收，另外的那一个goroutine发送了“789”并等待处理；完成通信后，打印出”789”；两个goroutine各自继续跑自己的。

记住：Golang的channel是用来goroutine之间通信的，且通信过程中会阻塞。

# Golang中函数被看做是值,函数值不可以比较，也不可以作为map的key

请问以下代码能编译通过吗？

```go
import (
	"fmt"
)

func main(){
	array := make(map[int]func ()int)

	array[func()int{ return 10}()] = func()int{
		return 12
	}

	fmt.Println(array)
}
```

**答案：**

```
可以正常编译通过。
```

稍作改动，改为如下的情况,还能编译通过吗？



```go
import (
	"fmt"
)

func main(){
	array := make(map[func ()int]int)

	array[func()int{return 12}] = 10

	fmt.Println(array)
}
```



**答案：**

```
不能编译通过。
```

在Go语言中，函数被看做是第一类值：(first-class values)：函数和其他值一样，可以被赋值，可以传递给函数，可以从函数返回。也可以被当做是一种“函数类型”。例如：有函数`func square(n int) int { return n * n }`，那么就可以赋值`f := square`,而且还可以`fmt.Println(f(3))`（将打印出“9”）。
Go语言函数有两点很特别：

-   **函数值类型不能作为map的key**
-   **函数值之间不可以比较，函数值只可以和nil作比较，函数类型的零值是`nil`**

# 匿名函数作用域陷阱

请看下列代码输出什么？



```go
import (
	"fmt"
)

func main(){
	var msgs []func()
	array := []string{
		"1", "2", "3", "4",
	}

	for _, e := range array{

		msgs = append(msgs, func(){
			fmt.Println(e)
		})
	}

	for _, v := range msgs{
		v()
	}
}
```



答案：

```
4
4
4
4
```

在上述代码中，**匿名函数中记录的是循环变量的内存地址，而不是循环变量某一时刻的值。**

想要输出1、2、3、4需要改为：

```go
import (
	"fmt"
)

func main(){
	var msgs []func()
	array := []string{
		"1", "2", "3", "4",
	}

	for _, e := range array{
		elem := e
		msgs = append(msgs, func(){
			fmt.Println(elem)
		})
	}

	for _, v := range msgs{
		v()
	}
}
```

其实就加了条`elem := e`看似多余，其实不，这样一来，每次循环后每个匿名函数中保存的就都是当时局部变量`elem`的值，这样的局部变量定义了4个，每次循环生成一个。

# `[3]int` 和 `[4]int` 不算同一个类型

请看一下代码，请问输出`true`还是`false`

```go
import (
    "fmt"
    "reflect"
)

func main(){
    arrayA := [...]int{1, 2, 3}
    arrayB := [...]int{1, 2, 3, 4}
    fmt.Println(reflect.TypeOf(arrayA) == reflect.TypeOf(arrayB))
}
```

答案是：

```
false
```

**数组长度是数组类型的一个组成部分**，因此[3]int和[4]int是两种不同的数组类型。

# 数组还可以指定一个索引和对应值的方式来初始化。

例如：

```go
import (
    "fmt"
)

func main(){
    arrayA := [...]int{0:1, 2:1, 3:4}
    fmt.Println(arrayA)
}
```

会输出：

```
[1 0 1 4]
```

有点像`PHP`数组的感觉，但是又不一样：`arrayA`的长度是多少呢？

```go
import (
    "fmt"
)

func main(){
    arrayA := [...]int{0:1, 2:1, 3:4}
    fmt.Println(len(arrayA))
}
```

答案是：

```
4
```

没错，定义了一个数组长度为4的数组，指定索引的数组长度和最后一个索引的数值相关，例如:`r := [...]int{99:-1}`就定义了一个含有100个元素的数组`r`，最后一个元素输出化为-1，其他的元素都是用0初始化。

# 不能对map中的某个元素进行取地址`&`操作

```
a := &ages["bob"] // compile error: cannot take address of map element
```

**map中的元素不是一个变量，不能对map的元素进行取地址操作，**禁止对map进行取地址操作的原因可能是map随着元素的增加map可能会重新分配内存空间，这样会导致原来的地址无效

# 当map为nil的时候，不能添加值

```go
func main() {
    var sampleMap map[string]int
    sampleMap["test"] = 1
    fmt.Println(sampleMap)
}
```

输出报错：

```
panic: assignment to entry in nil map
```

必须使用make或者将map初始化之后，才可以添加元素。

以上代码可以改为:

```go
func main() {
    var sampleMap map[string]int
    sampleMap = map[string]int {
        "test1":1,
    }
    sampleMap["test"] = 1
    fmt.Println(sampleMap)
}
```

可以正确输出:

```
map[test1:1 test:1]
```

# `&dilbert.Position`和`(&dilbert).Position`是不同的

```
&dilbert.Position`相当于`&(dilbert.Position)`而非`(&dilbert).Position
```

请看例子：

请问输出什么？

```go
func main(){
    type Employee struct {
        ID int
        Name string
        Address string
        DoB time.Time
        Position string
        Salary int
        ManagerID int
    }
    var dilbert Employee

    dilbert.Position = "123"

    position := &dilbert.Position
    fmt.Println(position)

}
```

输出：

```
0xc42006c220
```

输出的是内存地址

修改一下，把`&dilbert.Position`改为`(&dilbert).Position`

```go
func main(){
    type Employee struct {
        ID int
        Name string
        Address string
        DoB time.Time
        Position string
        Salary int
        ManagerID int
    }
    var dilbert Employee

    dilbert.Position = "123"

    position := (&dilbert).Position
    fmt.Println(position)

}
```

输出：

```
123
```

# Go语言中函数返回的是值的时候，不能赋值

请看下面例子:

```go
type Employee struct {
    ID int
    Name string
    Address string
    DoB time.Time
    Position string
    Salary int
    ManagerID int
}

func EmployeeByID(id int) Employee {
    return Employee{ID:id}
}

func main(){
    EmployeeByID(1).Salary = 0
}
```

请问能编译通过吗？

运行，输出报错：`cannot assign to EmployeeByID(1).Salary`

在本例子中，函数`EmployeeById(id int)`返回的是值类型的，它的取值`EmployeeByID(1).Salary`也是一个值类型；值类型是什么概念？值类型就是和赋值语句`var a = 1`或`var a = hello world`等号`=`右边的`1`、`Hello world`是一个概念，他是不能够被赋值的，只有变量能够被赋值。

修改程序如下：

```go
type Employee struct {
    ID int
    Name string
    Address string
    DoB time.Time
    Position string
    Salary int
    ManagerID int
}

func EmployeeByID(id int) Employee {
    return Employee{ID:id}
}

func main(){
    var a = EmployeeByID(1)
    a.Salary = 0
}
```

这就可以编译通过了

# 在声明方法时，如果一个类型名称本身就是一个指针的话，不允许出现在方法的接收器中

请看下面的例子，请问会编译通过吗？

```go
import (
	"fmt"
)

type littleGirl struct{
	Name string
	Age int
}

type girl *littleGirl

// girl是一个指针，不能作为方法接受
func(this girl) changeName(name string){
	this.Name = name
}

func main(){
	littleGirl := girl{Name:"Rose", Age:1}
	
	girl.changeName("yoyo")
	fmt.Println(littleGirl)
}
```

**答案:**

```
不能编译通过，会提示“invalid receiver type girl(girl is a pointer type)”
```

Go语言中规定，**只有类型（Type）和指向他们的指针（*Type）才是可能会出现在接收器声明里的两种接收器，**为了避免歧义，明确规定，如果一个类型名本身就是一个指针的话，是不允许出现在接收器中的。

# 函数允许nil指针作为参数，也允许用nil作为方法的接收器

请看下面的例子，请问能编译通过吗？

```go
import (
	"fmt"
)

type littleGirl struct{
	Name string
	Age int
}


func(this littleGirl) changeName(name string){
	fmt.Println(name)
}

func main(){
	little := littleGirl{Name:"Rose", Age:1}

	little = nil
	little.changeName("yoyo")
	fmt.Println(little)
}
```

**答案:**

```
不能编译通过，显示"cannot use nil as type littleGirl in assignment"
```

**Go语言中，允许方法用nil指针作为其接收器，也允许函数将nil指针作为参数**。而上述代码中的`littleGirl`不是指针类型，改为`*littleGirl`，然后变量`little`赋值为`&littleGirl{Name:"Rose", Age:1}`就可以编译通过了。
并且，nil对于对象来说是合法的零值的时候，比如map或者slice，也可以编译通过并正常运行。

# Golang的时间格式化

不同于PHP的`date("Y-m-d H:i:s", time())`，Golang的格式化奇葩的很，不能使用诸如`Y-m-d H:i:s`的东西，而是使用`2006-01-02 15:04:05`这个时间的格式，请记住这个时间，据说这是Golang的诞生时间。

```go
time := time.Now()

time.Format("20060102") //相当于Ymd

time.Format("2006-01-02")//相当于Y-m-d

time.Format("2006-01-02 15:04:05")//相当于Y-m-d H:i:s

time.Format("2006-01-02 00:00:00")//相当于Y-m-d 00:00:00
```

# 不要对Go并发函数的执行时机做任何假设

请看下列的列子：

```go
import (
	"fmt"
	"runtime"
	"time"
)

func main(){
	names := []string{"lily", "yoyo", "cersei", "rose", "annei"}
	for _, name := range names{
		go func(){
			fmt.Println(name)
		}()
	}
	runtime.GOMAXPROCS(1)
	runtime.Gosched()
}
```

请问输出什么？

答案:

```
annei
annei
annei
annei
annei
```

为什么呢？是不是有点诧异？
输出的都是“annei”，而“annei”又是“names”的最后一个元素，那么也就是说程序打印出了最后一个元素的值，**而name对于匿名函数来讲又是一个外部的值**。因此，我们可以做一个推断：虽然每次循环都启用了一个协程，但是这些协程都是引用了外部的变量，当协程创建完毕，再执行打印动作的时候，name的值已经不知道变为啥了，因为主函数协程也在跑，大家并行，但是在此由于names数组长度太小，当协程创建完毕后，主函数循环早已结束，所以，打印出来的都是遍历的names最后的那一个元素“annei”。
如何证实以上的推断呢？
其实很简单，每次循环结束后，停顿一段时间，等待协程打印当前的name便可。

```go
import (
	"fmt"
	"runtime"
	"time"
)

func main(){
	names := []string{"lily", "yoyo", "cersei", "rose", "annei"}
	for _, name := range names{
		go func(){
			fmt.Println(name)
		}()
		time.Sleep(time.Second)
	}
	runtime.GOMAXPROCS(1)
	runtime.Gosched()
}
```

打印结果：

```
lily
yoyo
cersei
rose
annei
```

以上我们得出一个结论，不要对“go函数”的执行时机做任何的假设，除非你确实能做出让这种假设成为绝对事实的保证。

# 假设T类型的方法上接收器既有`T`类型的，又有`*T`指针类型的，那么就不可以在不能寻址的T值上调用`*T`接收器的方法

请看代码,试问能正常编译通过吗？

```go
import (
	"fmt"
)
type Lili struct{
	Name string
}

func (Lili *Lili) fmtPointer(){
	fmt.Println("poniter")
}

func (Lili Lili) fmtReference(){
	fmt.Println("reference")
}


func main(){
	li := Lili{}
	li.fmtPointer()
}
```

答案：

```
能正常编译通过，并输出"poniter"
```

感觉有点诧异，请接着看以下的代码，试问能编译通过？

```go
import (
	"fmt"
)
type Lili struct{
	Name string
}

func (Lili *Lili) fmtPointer(){
	fmt.Println("poniter")
}

func (Lili Lili) fmtReference(){
	fmt.Println("reference")
}


func main(){
	Lili{}.fmtPointer()
}
```

答案：

```
不能编译通过。
“cannot call pointer method on Lili literal”
“cannot take the address of Lili literal”
```

是不是有点奇怪？这是为什么呢？**其实在第一个代码示例中，main主函数中的“li”是一个变量，li的虽然是类型Lili，但是li是可以寻址的**，&li的类型是`*Lili`，因此可以调用*Lili的方法。

# 一个包含nil指针的接口不是nil接口

请看下列代码，试问返回什么

```go
import (
	"bytes"
	"fmt"
	"io"
)

const debug = true

func main(){
	var buf *bytes.Buffer
	if debug{
		buf = new(bytes.Buffer)
	}
	f(buf)
}
func f(out io.Writer){

	if out != nil{
		fmt.Println("surprise!")
	}
}
```

答案是输出：surprise。
ok，让我们吧`debug`开关关掉，及`debug`的值变为`false`。那么输出什么呢？是不是什么都不输出？

```go
import (
	"bytes"
	"fmt"
	"io"
)

const debug = false

func main(){
	var buf *bytes.Buffer
	if debug{
		buf = new(bytes.Buffer)
	}
	f(buf)
}
func f(out io.Writer){

	if out != nil{
		fmt.Println("surprise!")
	}
}
```

答案是：依然输出surprise。

这是为什么呢？
这就牵扯到一个概念了，是关于接口值的。**概念上讲一个接口的值分为两部分：一部分是类型，一部分是类型对应的值，**他们分别叫：**动态类型和动态值。**类型系统是针对编译型语言的，类型是编译期的概念，因此类型不是一个值。
在上述代码中，给f函数的out参数赋了一个`*bytes.Buffer`的空指针，所以out的动态值是nil。然而它的动态类型是*bytes.Buffer，意思是：“A non-nil interface containing a nil pointer”，所以“out!=nil”的结果依然是true。
但是，对于直接的``*bytes.Buffer``类型的判空不会出现此问题。

```go
import (
	"bytes"
	"fmt"
)

func main(){
	var buf *bytes.Buffer
	if buf == nil{
		fmt.Println("right")
	}
}
```

还是输出: right
**只有 接口指针 传入函数的接口参数时，才会出现以上的坑。**
修改起来也很方便，把`*bytes.Buffer`改为`io.Writer`就好了。

```go
import (
	"bytes"
	"fmt"
	"io"
)
const debug = false
func main(){
	var buf  io.Writer //原来是var buf *bytes.Buffer
	if debug{
		buf = new(bytes.Buffer)
	}
	f(buf)
}
func f(out io.Writer){
	if out != nil{
		fmt.Println("surprise!")
	}
}
```

# 将map转化为json字符串的时候，json字符串中的顺序和map赋值顺序无关

请看下列代码，请问输出什么？若为json字符串，则json字符串中key的顺序是什么？

```go
func main() {
	params := make(map[string]string)

	params["id"] = "1"
	params["id1"] = "3"
	params["controller"] = "sections"

	data, _ := json.Marshal(params)
	fmt.Println(string(data))
}
```

答案：输出`{"controller":"sections","id":"1","id1":"3"}`
**利用Golang自带的json转换包转换，会将map中key的顺序改为字母顺序，而不是map的赋值顺序。**map这个结构哪怕利用`for range`遍历的时候,其中的key也是无序的，可以理解为map就是个无序的结构，和php中的array要区分开来

# Json反序列化数字到interface{}类型的值中，默认解析为float64类型

请看以下程序，程序想要输出json数据中整型`id`加上`3`的值,请问程序会报错吗？

~~~go
func main(){
	jsonStr := `{"id":1058,"name":"RyuGou"}`
	var jsonData map[string]interface{}
	json.Unmarshal([]byte(jsonStr), &jsonData)

	sum :=  jsonData["id"].(int) + 3
	fmt.Println(sum)
}

``` 
答案是会报错，输出结果为：
~~~

panic: interface conversion: interface {} is float64, not int

```
使用 Golang 解析 JSON  格式数据时，若以 interface{} 接收数据，则会按照下列规则进行解析：
```

bool, for JSON booleans

float64, for JSON numbers

string, for JSON strings

[]interface{}, for JSON arrays

map[string]interface{}, for JSON objects

nil for JSON null

```go
应该改为：
​```go
func main(){
	jsonStr := `{"id":1058,"name":"RyuGou"}`
	var jsonData map[string]interface{}
	json.Unmarshal([]byte(jsonStr), &jsonData)

	sum :=  int(jsonData["id"].(float64)) + 3
	fmt.Println(sum)
}
```

# 即使在有多个变量、且有的变量存在有的变量不存在、且这些变量共同赋值的情况下，也不可以使用`:=`来给全局变量赋值

`:=`往往是用来声明局部变量的，在多个变量赋值且有的值存在的情况下，`:=`也可以用来赋值使用,例如:

```go
msgStr := "hello wolrd"
msgStr, err := "hello", errors.New("xxx")//err并不存在
```

但是，假如全局变量也使用类似的方式赋值，就会出现问题，请看下列代码，试问能编译通过吗？

```go
var varTest string

func test(){
	varTest, err := function()
	fmt.Println(err.Error())
}

func function()(string, error){
	return "hello world", errors.New("error")
}


func main(){
	test()
}
```

答案是：通不过。输出：

```
varTest declared and not used
```

但是如果改成如下代码，就可以通过：

```go
var varTest string

func test(){
	err := errors.New("error")
	varTest, err = function()
	fmt.Println(err.Error())
}

func function()(string, error){
	return "hello world", errors.New("error")
}


func main(){
	test()
}
```

输出：

```
error
```

这是什么原因呢？
答案其实很简单，在`test`方法中，如果使用`varTest, err := function()`这种方式的话，相当于在函数中又定义了一个和全局变量`varTest`名字相同的局部变量，而这个局部变量又没有使用，所以会编译不通过。

# *interface 是一个指向interface的指针类型，而不是interface类型

请问以下代码，能编译通过吗？

```go
import (
	"fmt"
)

type Father interface {
	Hello()
}


type Child struct {
	Name string
}

func (s Child)Hello()  {

}

func main(){
	var buf  Child
	buf = Child{}
	f(&buf)
}
func f(out *Father){
	if out != nil{
		fmt.Println("surprise!")
	}
}
```

答案是：不能编译通过。输出：

```
*Father is pointer to interface, not interface
```

注意了：接口类型的变量可以被赋值为实现接口的结构体的实例，但是并不能代表接口的指针可以被赋值为实现接口的结构体的指针实例。即：

```
var buf Father = Child{}
```

是对的，但是

```
var buf *Father = new(Child)
```

却是不对的。应该改为：

```
var buf Father = Child{}
var pointer *Father = &buf
```

要想让问题最开始的代码编译通过要将以上代码修改为：

```go
import (
	"fmt"
)

type Father interface {
	Hello()
}


type Child struct {
	Name string
}

func (s Child)Hello()  {

}

func main(){
	var buf  Father
	buf = Child{}
	f(&buf)
}
func f(out *Father){
	if out != nil{
		fmt.Println("surprise!")
	}
}
```