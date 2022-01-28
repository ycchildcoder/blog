---
title: go 注意事项
date: 2022-01-27 17:50:31
tags:
---

### 请说出下面代码，执行时为什么会报错

```go
type Student struct {
	name string
}

func main() {
	m := map[string]Student{"people": {"zhoujielun"}}
	m["people"].name = "wuyanzu"
}
```
解析：
map的value本身是不可寻址的，因为map中的值会在内存中移动，并且旧的指针地址在map改变时会变得无效。故如果需要修改map值，可以将`map`中的非指针类型`value`，修改为指针类型，比如使用`map[string]*Student`.


### 2、写出打印的结果。

```go
type People struct {
	name string `json:"name"`
}

func main() {
	js := `{
		"name":"11"
	}`
	var p People
	err := json.Unmarshal([]byte(js), &p)
	if err != nil {
		fmt.Println("err: ", err)
		return
	}
	fmt.Println("people: ", p)
}
```

**解析：**

按照 golang 的语法，小写开头的方法、属性或 `struct` 是私有的，同样，在`json` 解码或转码的时候也无法上线私有属性的转换。

题目中是无法正常得到`People`的`name`值的。而且，私有属性`name`也不应该加`json`的标签。

### 3、sync.Map 的用法

```go
package main

import (
	"fmt"
	"sync"
)

func main(){
	var m sync.Map
	m.Store("address",map[string]string{"province":"江苏","city":"南京"})
        v,_ := m.Load("address")
	fmt.Println(v["province"]) 
}
```

- A，江苏；
- B`，v["province"]`取值错误；
- C，`m.Store`存储错误；
- D，不知道

## 解析

`invalid operation: v["province"] (type interface {} does not support indexing)` 因为 `func (m *Map) Store(key interface{}, value interface{})` 所以 `v`类型是 `interface {}` ，这里需要一个类型断言

```
fmt.Println(v.(map[string]string)["province"]) //江苏
```

```go
package main

import (
	"fmt"
	"sync"
)

func main() {
	var m sync.Map
	m.Store("address", map[string]string{"province": "江苏", "city": "南京"})
	v, _ := m.Load("address")

	tmp, ok := v.(map[string]string)
	if ok {
		fmt.Println(tmp["province"])
	}
	//if tmp,ok := v.(map[string]string);ok{
	//	
	//}
	//fmt.Println(v["province"])
}
```

### 4、知道golang的内存逃逸吗？什么情况下会发生内存逃逸？

## 回答

golang程序变量会携带有一组校验数据，用来证明它的整个生命周期是否在运行时完全可知。如果变量通过了这些校验，它就可以在栈上分配。否则就说它 逃逸 了，必须在堆上分配。

能引起变量逃逸到堆上的典型情况：

- **在方法内把局部变量指针返回** 局部变量原本应该在栈中分配，在栈中回收。但是由于返回时被外部引用，因此其生命周期大于栈，则溢出。
- **发送指针或带有指针的值到 channel 中。** 在编译时，是没有办法知道哪个 `goroutine` 会在 `channel` 上接收数据。所以编译器没法知道变量什么时候才会被释放。
- **在一个切片上存储指针或带指针的值。** 一个典型的例子就是 `[]*string` 。这会导致切片的内容逃逸。尽管其后面的数组可能是在栈上分配的，但其引用的值一定是在堆上。
- **slice 的背后数组被重新分配了，因为 append 时可能会超出其容量( cap )。** slice 初始化的地方在编译时是可以知道的，它最开始会在栈上分配。如果切片背后的存储要基于运行时的数据进行扩充，就会在堆上分配。
- **在 interface 类型上调用方法。** 在 interface 类型上调用方法都是动态调度的 —— 方法的真正实现只能在运行时知道。想像一个 io.Reader 类型的变量 r , 调用 r.Read(b) 会使得 r 的值和切片b 的背后存储都逃逸掉，所以会在堆上分配。



### 5、对已经关闭的的 chan 进行读写，会怎么样？为什么？

## 回答

- 读已经关闭的 chan 能一直读到东西，但是读到的内容根据通道内关闭前是否有元素而不同。
  - 如果 chan 关闭前，buffer 内有元素还未读 , 会正确读到 chan 内的值，且返回的第二个 bool 值（是否读成功）为 true。
  - 如果 chan 关闭前，buffer 内有元素已经被读完，chan 内无值，接下来所有接收的值都会非阻塞直接成功，返回 channel 元素的零值，但是第二个 bool 值一直为 false。
- 写已经关闭的 chan 会 panic



### Go编码注意事项

1. new和make的区别,前者返回的是**指针**，后者返回引用，还能给这个内存的**类型初始化其零值**，且make关键字只能创建channel、slice和map这三个引用类型。

2. 如果User结构想实现Test方法，以下写法：`func (this *User) Test() `，User的实例和*User都可以调到Test方法，不同的是作为接口时User没有实现Test方法。

3. interface　**作为两个成员实现，一个是类型和一个值**，`var x interface{} = (*interface{})(nil)`　接口指针x不等于nil  。下面一段代码深入展示：

```go
type User struct {
    Id   int
    Name string
    Tester
}
type Tester interface {
    Test()
}
func (this *User) Test() {
	fmt.Println(this)
}
func create() Tester {
    var x *User = nil
    return x
}
func Test(t *testing.T) {
    var x Tester = create()
    if x != nil {
       t.Log("not nil ")
    }
    var u *User = x.(*User)
    if u == nil {
       t.Log("nil ")
    }
}
```

4. 继承通过嵌入实现，也可说go语言没有继承语法。

5. import关键字前面的"."和"_"的用法，点号表示调用这个包的函数时可以省去包名，下划线表示纯引入包，因为go语法内没有使用这个包是不能导入的，包引入了，系统会自动调用包的init函数。

6. select case必须是chan的io操作，为了避免饥饿问题，当多个通道都同时监听到数据时，select机制会随机性选择一个通道读取，一个通道被多个select语句监听时，同理。

7. 关闭通道时所有select 监听都会收到通道关闭信号，某种意义上**关闭通道是广播事件**。
8. goroutine的panic如果没有捕获，整个应用程序会crash ,所以安全起见每个复杂的**go线程都要recover**。

9. 在函数退出时，defer的调用顺序是写在后面的先被调用。

10. init函数在main之前调用，被编译器自动调用，每个包理论上允许有多个init函数,编码上尽量避免同一个包内出现多个init函数。

11. panic可以中断原有的控制流程，进入一个令人恐慌的流程中,这一过程继续向上，直到发生panic的goroutine中所有调用的函数返回，此时程序退出。恐慌可以直接调用panic产生。也可以由运行时错误产生，例如访问越界的数组.
    recover的用法,recover可以让进入令人恐慌的流程中的goroutine恢复过来。**recover仅在defer函数中有效**。在正常的执行过程中，调用recover会返回nil，并且没有其它任何效果。

12. Array 和Slice的区别，**Array就是一个数据块，值类型**而非引用类型，传参时会进行内存拷贝，Slice是个`reflect.SliceHeader`结构体。Slice由make函数或者Array[:]创建。

13. 闭包要注意循环调用时，upvalue值一不留意可能只是循环退出的值。如下代码：

```go
func Test(t *testing.T) {
    var data int
    for i:= 0;i<10;i++{
    data ++
        go func(){
            listen2(data)
        }()
    }
    <- time.After(time.Second)
}
func listen2(data int) {
    fmt.Print( data)
}
```

>  输出：`26101010101010106`，跟你期望的输出可能不一样。


14. 普通类型向接口类型的转换是隐式的,定义该接口变量直接赋值。接口类型向普通类型转换需要**类型断言**：value, ok := element.(T)。

15. Go设计上模糊了堆跟栈的边界，go编译器帮程序员做了对象逃逸分析，优化了内存分配，t := T{}是可以在函数里返回的，并不是像C语言中在栈里分配内存了。

16. 无论以接口或接口指针传递参数，接口指向的值都会被拷贝传递，引用类型（Map/Chan/Slice）拷贝该引用对象，值类型拷贝整个值（string除外）。

17. go线程的调用时机是由go runtime决定的。

```go
func Test(t *testing.T) {
	for i:= 0;i<10;i++{
		go listen2(i)
	}
	<- time.After(time.Second)
}
func listen2(data int) {
	fmt.Print( data)
}
```

>  输出：`3456781209`


18. 调用log.Fatal系列函数后，会再调用 os.Exit(1)　退出程序，`Fatal is equivalent to Print() followed by a call to os.Exit(1)`。

19. 如果管道关闭则退出for循环，因为管道关闭不会阻塞导致for进入死循环,如下：

```go
// 错误的做法
func Test_Select_Chan(t *testing.T) {
	readerChannel:= make(chan int )
	go func(readerChannel chan int ) {
		for {
			select {
			// 判断管道是否关闭
			case _, ok := <-readerChannel:
				if !ok {
					break
				}
			}
			t.Log("for")
		}
	}(readerChannel)
	close(readerChannel)
	<- time.After(time.Second*2)
}
// 正确的做法
func Test_Select_Chan1(t *testing.T) {
	readerChannel:= make(chan int )
	go func(readerChannel chan int ) {
		for {
			select {
			// 判断管道是否关闭
			case _, ok := <-readerChannel:
				if !ok {
					goto BB
					//return
				}
			}
			t.Log("for")
		}
	BB:
	}(readerChannel)
	close(readerChannel)
	<- time.After(time.Second*2)
}
```

**for select 组合不带标签的break语法是跳不出循环，如果要跳出循环，要设置goto 标签或者直接return返回**。

20. `map,slice,array,chan`的数据存取值类型数据都是值拷贝赋值，这个跟很多脚本语言不同，一定要注意:

```go
var list []mydata
var hash map[string]mydata
type mydata struct {
    A int
}
func Test(t *testing.T) {
	list = make([]mydata, 1)
	data := list[0]
	data.A = 10
	hash = make(map[string]mydata)
	hash["test"] = mydata{}
	data = hash["test"]
	data.A = 10
	t.Log(list[0].A, hash["test"].A)
}
```

>  输出：`0 0`

21. 外部可见的属性必须是首字母大写，当转换到json数据是跟预期有偏差，必须添加json标签，如下：

```go
type CMD struct{
        Cmd string `json:"cmd"`
        Data Data `json:"data"`
        UserId string `json:"userId"`
    }
```

22. 包的循环引用编译错误，解决方法：提取公共部分到独立的包或者定义接口依赖注入。

23. `return XXX`不是一条原子指令：

```go
func Test(t *testing.T) {
	t.Log(test())
	t.Log(test1())
}
func test() (result int) {
	defer func() {
		result++
	}()
	return 1
}

func test1() (result int) {
	t := 5
	defer func() {
		t = t + 5
	}()
	return t
}
```

>  输出：
>  `2`
>  `5`

`return XXX `不是一条原子指令，**函数返回过程是，先对返回值赋值，再调用defer函数，然后返回调用函数**
所以test方法的`return 1`可以拆分为：

```go
result = 1
func()(result int){
    result ++
}()
return
```

24. 内置copy方法，拷贝数组时，如果要整数组拷贝，目标数组长度要和源数组长度相同，否则剩下的数据不会被拷贝:

```go
list := []int{12, 1242, 35, 23, 534, 23, 1}
listNew := make([]int, 1, len(list))
copy(listNew, list)
for i := 0; i < len(listNew); i++ {
fmt.Println(listNew[i])
}
```

>  输出：`12`

25. 分割切片slice时，新切片引用的内存和老切片引用的是同一块内存:

```go
list := []int{12, 1242, 35, 23, 534, 23, 1}
listNew := list[:3]
listNew2 := list[:5]
listNew[1] = 999
fmt.Println(listNew2[1])
```

>  输出：`999`

26. **编译时设置编译参数去掉调试信息**，可以让生成体积更小：
    `go build  -o target.exe -ldflags "-w -s" source.go`

27. Go语言里，string字符串类是不可变值类型，字符串的"+"连接操作、字符串和字符数组之间的转换string([]byte) 都会生成新的内存存放新字符串，当要对字符串频繁操作时做好先转换成字符数组。但是字符串作为参数传参时，此处go编译器作了优化，不会导致内存拷贝，引用的是同一块内存。benchmark如下：

```go
func Benchmark_aa(b *testing.B) {
	very_long_string:= ""
	for i := 0; i < b.N; i++ {
		very_long_string += "test " + "and test "
	}
}
func Benchmark_bb(b *testing.B) {
	very_long_string:= []byte{}
	for i := 0; i < b.N; i++ {
		very_long_string = append(very_long_string,[]byte("test ")...)
		very_long_string = append(very_long_string,[]byte("and test ")...)
	}
}
```

>  输出：
>  `200000	    135817 ns/op`
>  `50000000	        39.3 ns/op`

28. Go build/run/test 有个参数 -race  ，设置-race运行时会进行数据竞态检测，并把关键代码打印输出，不过数据竞态不能全依赖race检测,不一定能全部检测出来。
29. 如果非必要必要使用反射reflect和unsafe包内的函数，一定要使用时，要用runtime.KeepAlive函数告知SSA编译器在指定的代码段内不要回收该内存块。
30. 不要打印整个map对象或者对象里有嵌套map的对象，打印函数会不加锁遍历map的每个元素，如果此时外部刚好有方法对map进行写操作，map就进入并发读写，runtime会panic。
31. 注意range 循环迭代时key 的地址，for k,v:=range list  其中**k 在迭代时指向同一个地址**。

32. append追加切片用法:

- 如果slice还有剩余的空间，可以添加这些新元素，那么append就将新的元素放在slice后面的空余空间中
- 如果slice的空间不足以放下新增的元素，那么就需要重现创建一个数组；这时可能是alloc、也可能是realloc的方式分配这个新的数组
- 也就是说，这个新的slice可能和之前的slice在同一个起始地址上，也可能不是一个新的地址
- 如果容量不足触发realloc，重新分配一个新的地址
- 分配了新的地址之后，再把原来slice中的元素逐个拷贝到新的slice中并返回
- 触发realloc时，容量小于1024，会扩展到原来的1倍，如果容量小大于1024，会扩展原来的1/4

33. 很多打印函数打印结构体时回调用该结构体的String方法，所以String不能再打印本身这个对象。如下：

```go
type S struct {
}
func (this S)String()  string{
	return fmt.Sprint( "S struct: ",this )
}
func Test_print(t *testing.T)  {
	var s S
	t.Log(s.String())
}
// print 调用String
```

34.循环语句里正整型迭代值边界问题，迭代值边界递减到负值，下面的代码会进入死循环：

```go
for i:= uint8(10);i>=0;i--{
	t.Log(i)
}
```

35. recover只处理本goroutine调用栈，goroutine的panic如果没有捕获，整个应用程序会crash ,所以安全起见**每个复杂的go线都要recover**。

36. map 并发读写的错误无法用panic捕获。

37. 对于小对象，直接将值类型的对象交由 map 保存，远比用该对象指针高效。这不但减少了堆内存分配，关键还在于**垃圾回收器不会扫描非指针类型 key/value 对象。**

38. Go 使用 channel 实现 CSP 模型。处理双方仅关注通道和数据本身，无需理会对方身份和数量，以此实现结构性解耦。在各文宣中都有 “Don't communicate by sharing memory, share memory by communicating.” 这类说法。但这并非鼓励我们不分场合，教条地使用 channel。在我看来，channel 多数时候适用于结构层面，而非单个区域的数据处理。原话中 “communicate” 本就表明一种 “message-passing”，而非 “lock-free”。所以，它并非用来取代 mutex，各自有不同的使用场景。

39. 变量逃逸和函数内联状态分析 `go build -gcflags "-m" -o main.exe main.go `。

40. go汇编指令  `go build  -gcflags "-N -l" -o main.exe main.go && go tool objdump -s "main\.main" main.exe`  关闭内联优化：`go build  -gcflags "-N -l"`。

41. 关于defer机制，编译器通过 `runtime.deferproc` “注册” 延迟调用，除目标函数地址外，还会复制相关参数（包括 receiver）。在函数返回前，执行 `runtime.deferreturn` 提取相关信息执行延迟调用。这其中的代价自然不是普通函数调用一条 CALL 指令所能比拟的，单个函数里过多的 defer 调用可尝试合并。最起码，在并发竞争激烈时，`mutex.Unlock `不应该使用 defer，而应尽快执行，仅保护最短的代码片段。

42. 对map预设容量，map会按需扩张，但须付出数据拷贝和重新哈希成本。如有可能，应尽可能预设足够容量空间，避免此类行为发生。

43. go语言调用c语言：

- import "C"  这句代码必须紧跟伪注释的C语言代码后面不能有换行
- go语言的CGO会自动链接编译.c文件，但是必须用go build 编译指令，不能指定main.go文件

44. Go语言命名规范:

- **文件命名，全小写+下划线**
- **结构体名字、变量名字采用驼峰命名(根据包外可见性确定首字母是否大写)**
- **常量命名，全大写 +下划线**

45. 删除切片的指定下标元素

- 低效的做法

``` go
		for j := 0; j < len(array); j++ {
			if INDEX == j {
				array = append(array[:j], array[j+1:]...)
				break
			}
		}
```

- 高效的做法

``` go
		
			for j :=INDEX; j < len(array)-1; j++ {
				array[j] = array[j+1]
			}
			array = array[:len(array)-1]
	
```











































