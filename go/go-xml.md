---
title: Go xml
date: 2022-01-29 16:41:46
tags:
---

导入头文件
```
import "encoding/xml"
```

### func Unmarshal —— 用于解析XML文件

```
func Unmarshal(data []byte, v interface{}) error
```

Unmarshal解析XML编码的数据并将结果存入v指向的值。v只能指向结构体、切片或者和字符串。良好格式化的数据如果不能存入v，会被丢弃。

因为Unmarshal使用**reflect包**，它只能填写导出字段。本函数好似用大小写敏感的比较来匹配XML元素名和结构体的字段名/标签键名。

Unmarshal函数使用如下规则将XML元素映射到结构体字段上。这些规则中，字段标签指的是结构体字段的标签键'xml'对应的值（参见上面的例子）：

```
* 如果结构体字段的类型为字符串或者[]byte，且标签为",innerxml"，
  Unmarshal函数直接将对应原始XML文本写入该字段，其余规则仍适用。
* 如果结构体字段类型为xml.Name且名为XMLName，Unmarshal会将元素名写入该字段
* 如果字段XMLName的标签的格式为"name"或"namespace-URL name"，
  XML元素必须有给定的名字（以及可选的名字空间），否则Unmarshal会返回错误。
* 如果XML元素的属性的名字匹配某个标签",attr"为字段的字段名，或者匹配某个标签为"name,attr"
  的字段的标签名，Unmarshal会将该属性的值写入该字段。
* 如果XML元素包含字符数据，该数据会存入结构体中第一个具有标签",chardata"的字段中，
  该字段可以是字符串类型或者[]byte类型。如果没有这样的字段，字符数据会丢弃。
* 如果XML元素包含注释，该数据会存入结构体中第一个具有标签",comment"的字段中，
  该字段可以是字符串类型或者[]byte类型。如果没有这样的字段，字符数据会丢弃。
* 如果XML元素包含一个子元素，其名称匹配格式为"a"或"a>b>c"的标签的前缀，反序列化会深入
  XML结构中寻找具有指定名称的元素，并将最后端的元素映射到该标签所在的结构体字段。
  以">"开始的标签等价于以字段名开始并紧跟着">" 的标签。
* 如果XML元素包含一个子元素，其名称匹配某个结构体类型字段的XMLName字段的标签名，
  且该结构体字段本身没有显式指定标签名，Unmarshal会将该元素映射到该字段。
* 如果XML元素的包含一个子元素，其名称匹配够格结构体字段的字段名，且该字段没有任何模式选项
  （",attr"、",chardata"等），Unmarshal会将该元素映射到该字段。
* 如果XML元素包含的某个子元素不匹配以上任一条，而存在某个字段其标签为",any"，
  Unmarshal会将该元素映射到该字段。
* 匿名字段被处理为其字段好像位于外层结构体中一样。
* 标签为"-"的结构体字段永不会被反序列化填写。
```



Unmarshal函数将XML元素写入string或[]byte时，会将该元素的字符数据串联起来作为值，目标[]byte不能是nil。

Unmarshal函数将属性写入string或[]byte时，会将属性的值以字符串/切片形式写入。

Unmarshal函数将XML元素写入切片时，会将切片扩展并将XML元素的子元素映射入新建的值里。

Unmarshal函数将XML元素/属性写入bool值时，会将对应的字符串转化为布尔值。

Unmarshal函数将XML元素/属性写入整数或浮点数类型时，会将对应的字符串解释为十进制数字。不会检查溢出。

Unmarshal函数将XML元素写入xml.Name类型时，会记录元素的名称。

Unmarshal函数将XML元素写入指针时，会申请一个新值并将XML元素映射入该值。

举例：

xml文件为：

```xml
<?xml version="1.0" encoding="utf-8"?>
<servers version="1">
    <server>
		<serverPod>10.10.10.10</serverPod>
        <serverName ISP="CNC">Shanghai_VPN</serverName>
        <serverIP ISP="CNC">127.0.0.1</serverIP>
    </server>
    <server>
		<serverPod>10.10.10.10</serverPod>
        <serverName ISP="CNC">Beijing_VPN</serverName>
        <serverIP ISP="CNC">127.0.0.2</serverIP>
    </server>
</servers>
```



举例：

```go
package main

import (
	"encoding/xml"
	"fmt"
	"io/ioutil"
	"log"
	"os"
)

const (
	servers = "E:\\go\\src\\awesomeProject\\001\\server.xml"
)

type Recurlyservers struct { //后面的内容是struct tag，标签，是用来辅助反射的
	XMLName     xml.Name `xml:"servers"`      //将元素名写入该字段
	Version     string   `xml:"version,attr"` //将version该属性的值写入该字段
	Svs         []server `xml:"server"`
	Description string   `xml:",innerxml"` //Unmarshal函数直接将对应原始XML文本写入该字段
}

type server struct {
	XMLName    xml.Name   `xml:"server"`
	ServerPod  string     `xml:"serverPod"`
	ServerName ServerName `xml:"serverName"`
	ServerIP   ServerIP   `xml:"serverIP"`
}

type ServerName struct {
	XMLName xml.Name `xml:"serverName"`
	ISP     string   `xml:"ISP,attr"`
	SName   string   `xml:",chardata"`
}

type ServerIP struct {
	XMLName xml.Name `xml:"serverIP"`
	ISP     string   `xml:"ISP,attr"`
	SIP     string   `xml:",chardata"`
}

func main() {
	file, err := os.Open(servers)
	if err != nil {
		log.Fatal(err)
	}

	defer file.Close()
	data, err := ioutil.ReadAll(file)
	if err != nil {
		log.Fatal(err)
	}

	v := Recurlyservers{}
	err = xml.Unmarshal(data, &v)
	if err != nil {
		log.Fatal(err)
	}
	fmt.Println(v)

	fmt.Printf("XMLName: %#v\n", v.XMLName)
	fmt.Printf("Version: %q\n", v.Version)

	fmt.Printf("Server: %v\n", v.Svs)
	for i, svs := range v.Svs {
		fmt.Println(i)
		fmt.Printf("Server XMLName: %#v\n", svs.XMLName)
		fmt.Printf("Server ServerName: %q\n", svs.ServerName)
		fmt.Printf("Server ServerIP: %q\n", svs.ServerIP)
		fmt.Printf("Server ServerPod: %s\n", svs.ServerPod)
	}
	fmt.Printf("Description: %q\n", v.Description)

}

```

生成XML文件使用下面的两个函数：

### func Marshal

```
func Marshal(v interface{}) ([]byte, error)
```

Marshal函数返回v的XML编码。

Marshal处理数组或者切片时会序列化每一个元素。Marshal处理指针时，会序列化其指向的值；如果指针为nil，则啥也不输出。Marshal处理接口时，会序列化其内包含的具体类型值，如果接口值为nil，也是不输出。Marshal处理其余类型数据时，会输出一或多个包含数据的XML元素。

XML元素的名字按如下优先顺序获取：

```
- 如果数据是结构体，其XMLName字段的标签
- 类型为xml.Name的XMLName字段的值
- 数据是某结构体的字段，其标签
- 数据是某结构体的字段，其字段名
- 被序列化的类型的名字
```

一个结构体的XML元素包含该结构体所有导出字段序列化后的元素，有如下例外：



```
- XMLName字段，如上所述，会省略
- 具有标签"-"的字段会省略
- 具有标签"name,attr"的字段会成为该XML元素的名为name的属性
- 具有标签",attr"的字段会成为该XML元素的名为字段名的属性
- 具有标签",chardata"的字段会作为字符数据写入，而非XML元素
- 具有标签",innerxml"的字段会原样写入，而不会经过正常的序列化过程
- 具有标签",comment"的字段作为XML注释写入，而不经过正常的序列化过程，该字段内不能有"--"字符串
- 标签中包含"omitempty"选项的字段如果为空值会省略
  空值为false、0、nil指针、nil接口、长度为0的数组、切片、映射
- 匿名字段（其标签无效）会被处理为其字段是外层结构体的字段
```



如果一个字段的标签为"a>b>c"，则元素c将会嵌套进其上层元素a和b中。如果该字段相邻的字段标签指定了同样的上层元素，则会放在同一个XML元素里。

参见MarshalIndent的例子。如果要求Marshal序列化通道、函数或者映射会返回错误。

### func MarshalIndent

```
func MarshalIndent(v interface{}, prefix, indent string) ([]byte, error)
```

MarshalIndent功能类似Marshal。但每个XML元素会另起一行并缩进，该行以prefix起始，后跟一或多个indent的拷贝（根据嵌套层数）。它只是多了缩进的设定，使得生成的XML文件可读性更高

两个函数第一个参数是用来生成XML的结构定义类型数据，都是返回生成的XML数据流

举例生成上面解析的XML文件：



```go
package main

import (
	"bytes"
	"encoding/xml"
	"fmt"
	"io/ioutil"
	"os"
)

type Servers struct { //后面的内容是struct tag，标签，是用来辅助反射的
	XMLName xml.Name `xml:"servers"`      //将元素名写入该字段
	Version string   `xml:"version,attr"` //将version该属性的值写入该字段
	Svs     []server `xml:"server"`
}

type server struct {
	ServerName string `xml:"serverName"`
	ServerIP   string `xml:"serverIP"`
}

func main() {
	v := &Servers{Version: "1"}
	v.Svs = append(v.Svs, server{"Shanghai_VPN", "127.0.0.1"})
	v.Svs = append(v.Svs, server{"Beijing_VPN", "127.0.0.2"})
	//每个XML元素会另起一行并缩进，每行以prefix(这里为两个空格)起始，后跟一或多个indent（这里为四个空格）的拷贝（根据嵌套层数）
	//即第一层嵌套只递进四个空格，第二层嵌套则递进八个空格
	output, err := xml.MarshalIndent(v, "", "    ")
	if err != nil {
		fmt.Printf("error : %v\n", err)
	}
	os.Stdout.Write([]byte(xml.Header)) //输出预定义的xml头  <?xml version="1.0" encoding="UTF-8"?>
	os.Stdout.Write(output)
	//buf := bytes.NewBuffer(nil)
	buf := &bytes.Buffer{}
	buf.Write([]byte(xml.Header))
	buf.Write(output)
	fmt.Println(buf.String())
	ioutil.WriteFile("E:\\go\\src\\awesomeProject\\001\\server01.xml", output, 0666)
}

```

需要os.Stdout.Write([]byte(xml.Header))这句代码是因为上面的两个函数输出的信息都是不带XML头的，为了生成正确的xml文件，需要使用xml包预定义的Header变量


