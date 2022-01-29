---
title: Go结构体mixture
date: 2022-01-28 14:50:31
tags:
---
# [Golang报错mixture of field:value and value initializers]

Golang 在使用匿名成员初始化时，如果出现`mixture of field:value and value initializers`
是因为初始化的方式不对，见代码：

```go
package main

import (
    "fmt"
)

type Person struct {
    Name string
    Age  int
    Sex  string
}

type Student struct {
    Person
    Id    string
    Grade string
}


func main() {
    s1 := Student{Person: Person{Name: "张三", Age: 13, Sex: "男"}, Id: "13321", Grade: "三年级"}
    fmt.Printf("%+v\n", s1)

    s2 := Student{Person{"张三", 13, "男"}, "12312", "三年级"}
    fmt.Println(s2)

    s3 := Student{Person{Name: "张三", Age: 13, Sex: "男"}, Id: "13321", Grade: "三年级"} //报错 mixture of field:value and value initializers（字段的混合：值和值初始化器）
    fmt.Println(s3)
}
```

s3直接导致代码编译不过去，想要指定字段就必须按 s1的方式 Person:Person{xxx:"xxx"}，要么就不指定按照s2的方式

```go
func TestSplit(t *testing.T) {
   type args struct {
      s   string
      sep string
   }
   tests := []struct {
      name       string
      args       args
      wantResult []string
   }{
      // TODO: Add test cases.
       {name: "001", args: args{s: "123", sep: "1"}, wantResult: []string{"", "23"}},  // 都field：value
      {"002", args{s: "123", sep: "2"}, []string{"1", "3"}}, // 都 value
   }
   for _, tt := range tests {
      t.Run(tt.name, func(t *testing.T) {
         if gotResult := Split(tt.args.s, tt.args.sep); !reflect.DeepEqual(gotResult, tt.wantResult) {
            t.Errorf("Split() = %v, want %v", gotResult, tt.wantResult)
         }
      })
   }
}
```



Go 语言的数组可以存储一组相同类型的数据，而结构体可以将不同类型的变量数据组合在一起，每一个变量都是结构体的成员。

#### 创建并初始化一个结构体

可以使用下面的语法创建一个结构体：

```go
type StructName struct{
    field1 fieldType1
    field2 fieldType2
}
```

创建一个含有 `firstName`、`lastName`、`salary` 和 `fullTime` 成员变量的结构体 `Employee`。

```go
type Empolyee struct{
	firstName string
	lastName string
	salary int
	fullTime bool
}
```

相同类型的成员变量可以放在一行，所以，上面的代码可以简写成：

```go
type Empolyee struct{
	firstName,lastName string
	salary int
	fullTime bool
}
```

使用类型别名 `Employee` 创建一个结构体变量 `ross`

```go
var ross Empolyee
ross.firstName = "ross"
ross.lastName = "Bingo"
ross.salary = 1000
ross.fullTime = true
fmt.Println(ross)
```

输出：

```
{ross Bingo 1000 true}
```

上面的代码创建了结构体变量 `ross`，并为每一个成员变量赋值。使用`.`访问结构体的成员。
 还可以使用**字面量**的方式初始化结构体：

```go
1、方式一
ross := Empolyee{
	"ross",
	"Bingo",
	1000,
	true,
}
输出：{ross Bingo 1000 true}

2、方式二
ross := Empolyee{
	lastName:"Bingo",
	firstName:"ross",
	salary:1000,
}
输出：{ross Bingo 1000 false}
```

**方式一，初始化时省略了成员变量名称，但是必须按顺序地将给出所有的成员的值。必须记住所有成员的类型且按顺序赋值**，这给开发人员带来了额外的负担且代码的维护性差，一般不采用这种方式；
 提倡采用**方式二，不用关心成员变量的顺序，给需要初始化的成员赋值，未赋值的成员默认就是类型对应的零值**。

**注意**：**方式一和方式二初始化方式不可以混用**

```go
ross := Empolyee{
	firstName:"ross",
	lastName:"Bingo",
	1000,
	fullTime:true,
}
```

编译出错：`mixture of field:value and value initializers`

**成员变量的顺序对于结构体的同一性很重要，如果将上面的 `firstName`、`lastName` 互换顺序或者将 `fullTime`、`salary` 互换顺序，都是在定义一个不同的结构体类型**

### 结构体指针

初始化结构体的时候，可以声明一个指向结构体的指针：

```go
ross_pointer := &Empolyee{
	firstName:"ross",
	lastName:"Bingo",
	salary:1000,
	fullTime:true,
}
```

上面的代码，创建了一个指向 `Empolyee` 结构体的指针 `ross_pointer`。可以通过指针访问结构体的成员：

```go
fmt.Println(*ross_pointer)
fmt.Println("firstName:",(*ross_pointer).firstName)
fmt.Println("firstName:",ross_pointer.lastName)
```

输出：

```go
{ross Bingo 1000 true}
firstName: ross
firstName: Bingo
```

`ross_pointer` 是一个结构体变量，所以 `(*ross_pointer).firstName` 和 `ross_pointer.lastName` 都是正确的访问方式 。

### 匿名成员

定义结构体时可以只指定成员类型，不用指定成员名，Go 会自动地将成员类型作为成员名。这种结构体成员称为**匿名成员**。这个结构体成员的类型必须是命名类型或者是指向命名类型的指针。

```go
type Week struct{
	string
	int
	bool
}
func main() {
	week := Week{"Friday",1000,true}
	fmt.Println(week)
}
```

上面的代码定义了结构体 `Week` ，有 `string`、`int` 和 `bool` 三个成员变量，变量名与类型相同。 这种定义方式可以和指定成员名混合使用，例如：

```go
type Empolyee struct{
	firstName,lastName string
	salary int
	bool
}
```

### 结构体嵌套

Go 有**结构体嵌套**机制，一个结构体可以作为另一个结构体类型的成员。

```go
type Salary struct {
	basic int
	workovertime int
}

type Empolyee struct{
	firstName,lastName string
	salary Salary
	bool
}

func main() {
	ross := Empolyee{
	    	firstName:"Ross",
            lastName:"Bingo",
            bool:true,
            salary:Salary{1000,100},
	}
	fmt.Println(ross.salary.basic);
}
```

我们新定义了结构体类型 `Salary`，将 `Empolyee` 成员类型修改成结构体类型 `Salary`。 创建了结构体 `ross`，想要访问成员 `salary` 里面的成员还是可以采用 `.` 的方式，例如：`ross.salary.basic`。
 如果结构体嵌套层数过多时，想要访问最里面结构体成员时，采用上面这种访问方式就会牵扯很多中间变量，造成代码很臃肿。**可以采用上面的匿名成员方式简化这种操作。**
 采用匿名成员方式重新定义结构体类型 `Empolyee`

```go
type Empolyee struct{
	firstName,lastName string
	Salary    // 匿名成员
	bool
}
func main() {
	ross := Empolyee{
		firstName:"Ross",
		lastName:"Bingo",
		bool:true,
		Salary:Salary{1000,100},
	}
	fmt.Println(ross.basic);         // 访问方式一
	fmt.Println(ross.Salary.basic);  // 访问方式二
	ross.basic = 1200
	fmt.Println(ross.basic)          // update
}
```

上面两种方式是等价的。通过这种方式，简化了访问过程。
 需要**注意**的是，**被嵌套的匿名结构体成员中，不能与上一层结构体成员同名**。

```go
type Empolyee struct {
	firstName, lastName string
	Salary
	basic int
	bool
}
func main() {
	ross := Empolyee{
		firstName: "Ross",
		lastName:  "Bingo",
		bool:      true,
		Salary:    Salary{1000, 100},
	}
	fmt.Println(ross.basic)
	fmt.Println(ross.workovertime)
	fmt.Println(ross.Salary.basic)
}
```

通过结构体访问

### 可导出的成员

一个 Go 包中的**变量、函数首字母大写，那这个变量或函数是可以导出的。**这是 Go 最主要的访问控制机制。如果一个结构体的成员变量名首字母大写，那这个成员也是可导出的。一个结构体可以同时包含可导出和不可导出的成员变量。
 在路径 `WORKSPACE/src/org/employee.go` 创建一个名为 `org` 的包，添加如下代码：

```go
// employee.go
package org
type Employee struct {
	FirstName,LastName string
	salary int
	fullTime bool
}
```

上面的 `Employee` 结构体，只有变量 `FirstName`、`LastName` 是可导出的。当然，`Employee` 也是可导出的。 在 `main` 包中导入 `org` 包：

```go
// main.go
package main
import (
	"org"
	"fmt"
)
func main() {
	ross := org.Employee{
		FirstName:"Ross",
		LastName:"Bingo",
		salary:1000,     
	}
	fmt.Println(ross)
}
```

上面的代码编译出错，因为成员变量 `salary` 是不可导出的： `unknown field 'salary' in struct literal of type org.Employee`

因为 `Employee` 来自包 `org`，所以用 `org.Employee` 去创建结构体 `ross`。可以采用类型别名简化：

```go
package main

import (
	"org"
	"fmt"
)

type Employee org.Employee; 

func main() {
	ross := Employee{
		FirstName:"Ross",
		LastName:"Bingo",
	}
	fmt.Println(ross)
}
```

输出：

```
{Ross Bingo 0 false}
```

### 结构体比较

如果结构体的所有成员都是可比较的，则这个结构体就是可比较的。可以使用 `==` 和 `!=` 作比较，其中 `==` 是按照顺序比较两个结构体变量的成员变量。

```go
type Employee struct {
	FirstName,LastName string
	salary int
	fullTime bool
}
func main() {
	ross := Employee{
		FirstName:"Ross",
		LastName:"Bingo",
	}
	jack := Employee{
		FirstName:"Jack",
		LastName:"Lee",
	}
	fmt.Println(ross == jack)
}
```

输出：

```
false
```

不同类型的结构体变量是不能比较的：

```go
type User struct {
	username string
}
type Employee struct {
	FirstName,LastName string
	salary int
	fullTime bool
}
func main() {
	ross := Employee{
		FirstName:"Ross",
		LastName:"Bingo",
	}
	user := User{
		username:"Seekload",
	}
	fmt.Println(ross == user)
}
```

编译出错： `invalid operation: ross == user (mismatched types Employee and User)` .
 然而，如果有成员是不能比较的，例如：`map`，则这个结构体是不能比较的。


