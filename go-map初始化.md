---
title: golang map初始化及使用
date: 2022-01-21 15:42:23
tags:
---



特点：

-   类似其它语言中的哈希表或字典，以key-value形式存储数据
-   key必须是支持==或!=比较运算的类型，不可以是函数、map或slice
-   Map通过key查找value比线性搜索快很多
-   Map使用make()创建，支持:=这种简写方式
-   make([keyType]valueType,cap),cap表示容量，可省略
-   超出容量时会自动扩容，但尽量提供一个合理的初始值
-   使用len()获取元素个数
-   键值对不存在时自动添加，使用delete()删除某键值对
-   使用for range对map和slice进行迭代



```go
// 先声明map
var m1 map[string]string
// 再使用make函数创建一个非nil的map，nil map不能赋值
m1 = make(map[string]string)
// 最后给已声明的map赋值
m1["a"] = "aa"
m1["b"] = "bb"

// 直接创建
m2 := make(map[string]string)
// 然后赋值
m2["a"] = "aa"
m2["b"] = "bb"

// 初始化 + 赋值一体化
m3 := map[string]string{
	"a": "aa",
	"b": "bb",
}

// ==========================================
// 查找键值是否存在
if v, ok := m1["a"]; ok {
	fmt.Println(v)
} else {
	fmt.Println("Key Not Found")
}

// 遍历map
for k, v := range m1 {
	fmt.Println(k, v)
}

// 遍历map
for k := range m1 {
	fmt.Println(k, m1[k])
}
```

![img](https://raw.githubusercontent.com/ycchildcoder/markdown/main/a80408a137b13f934b0dd6f2b6c5cc03.jpg)