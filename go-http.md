---
title: golang http请求
date: 2022-01-23 21:42:23
tags:
---



Golang提供了官方的http包，对于http操作非常的方便和简洁。


# get 请求

get请求有好几种方式

#### 直接使用`net/http`包内的函数请求

```go
import "net/http"
...
resp, err := http.Get("http://wwww.baidu.com")
```

#### 利用http.client结构体来请求

```go
import "net/http"
...
clt := http.Client{}
resp, err := clt.Get("http://wwww.baidu.com")
```

#### 最本质的请求方式

如果稍微看一下源码，就会发现以上两种方式都是用了一下这种最本质的请求方式，使用`http.NewRequest`函数

```go
req, err := http.NewRequest("GET", "http://www.baidu.com", nil)

//然后http.client 结构体的 Do 方法
//http.DefaultClient可以换为另外一个http.client
resp, err := http.DefaultClient.Do(req)
```

**Go的get请求面上有好几种请求方式，实则只有一种：**

**1、使用`http.NewRequest`函数获得`request`实体**

**2、利用`http.client`结构体的`Do`方法，将`request`实体传入`Do`方法中。**

# post请求

和get请求类似，post请求也有多种方法，**但本质还是使用了`http.NewRequest`函数和`http.client`的`Do`方法。**

#### 使用`net/http`包带的post方法

```go
import (
"net/http"
"net/url"
)
...
data := url.Values{"start":{"0"}, "offset":{"xxxx"}}
body := strings.NewReader(data.Encode())
resp, err := http.Post("xxxxxxx", "application/x-www-form-urlencoded", body)
```

或者还可以

```go
import (
"net/http"
"net/url"
)
...
var r http.Request
r.ParseForm()
r.Form.Add("xxx", "xxx")
body := strings.NewReader(r.Form.Encode())
http.Post("xxxx", "application/x-www-form-urlencoded", body)
```

要是还是觉得复杂，还可以：

```go
import (
"net/http"
"net/url"
)
...
data := url.Values{"start":{"0"}, "offset":{"xxxx"}}
http.PostForm("xxxx", data)
```

#### 使用实例化的http client的post方法

其实本质上直接使用包的函数和实例化的http client是一样的，包的函数底层也仅仅是实例化了一个`DefaultClient`，然后调用的`DefaultClient`的方法。下面给出使用实例化的http client的post方法：

```go
import (
"net/http"
"net/url"
)
...
data := url.Values{"start":{"0"}, "offset":{"xxxx"}}
body := strings.NewReader(data.Encode())
clt := http.Client{}
resp, err := clt.Post("xxxxxxx", "application/x-www-form-urlencoded", body)
```

还有

```
import (
"net/http"
"net/url"
)
...
var r http.Request
r.ParseForm()
r.Form.Add("xxx", "xxx")
body := strings.NewReader(r.Form.Encode())
clt := http.Client{}
clt.Post("xxxx", "application/x-www-form-urlencoded", body)
```

简单的，但仅限于form表单

```
import (
"net/http"
"net/url"
)
...
data := url.Values{"start":{"0"}, "offset":{"xxxx"}}
clt := http.Client{}
clt.PostForm("xxxx", data)
```

#### 使用`net/http`包的`NewRequest`函数

其实不管是get方法也好，post方法也好，所有的get、post的的http 请求形式，最终都是会调用`net/http`包的`NewRequest`函数，多种多样的请求形式，也仅仅是封装的不同而已。

```go
import (
"net/http"
"net/url"
)
...

data := url.Values{"start":{"0"}, "offset":{"xxxx"}}
body := strings.NewReader(data.Encode())

req, err := http.NewRequest("POST", "xxxxx", body)
req.Header.Set("Content-Type", "application/x-www-form-urlencoded")

clt := http.Client{}
clt.Do(req)
```

# 添加request header

`net/http`包没有封装直接使用请求带header的get或者post方法，所以，要想请求中带header，只能使用`NewRequest`方法。

```go
import (
"net/http"

)
...

req, err := http.NewRequest("POST", "xxxxx", body)
//此处还可以写req.Header.Set("User-Agent", "myClient")
req.Header.Add("User-Agent", "myClient")

clt := http.Client{}
clt.Do(req)
```

有一点需要注意：在添加header操作的时候，`req.Header.Add`和`req.Header.Set`都可以，但是在修改操作的时候，只能使用`req.Header.Set`。
这俩方法是有区别的，Golang底层Header的实现是一个`map[string][]string`，`req.Header.Set`方法如果原来Header中没有值，那么是没问题的，如果又值，会将原来的值替换掉。而`req.Header.Add`的话，是在原来值的基础上，再`append`一个值，例如，原来header的值是“s”，我后`req.Header.Add`一个”a”的话，变成了`[s a]`。但是，获取header值的方法`req.Header.Get`确只取第一个，所以，如果原来有值，重新`req.Header.Add`一个新值的话，`req.Header.Get`得到的值不变。

# 打印response响应

Golang打印response没有PHP那么爽，哎，编译型语言就是麻烦。

```go
import (
	"net/http"
	"net/url"
	"io/ioutil"
)
...
content, err := ioutil.ReadAll(resp.Body)
respBody := string(content)
```

