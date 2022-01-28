---
title: golang ldflags
date: 2022-01-25 15:42:23
tags:
---

#### 含义

[Using `ldflags` with `go build`](https://links.jianshu.com/go?to=https%3A%2F%2Fwww.digitalocean.com%2Fcommunity%2Ftutorials%2Fusing-ldflags-to-set-version-information-for-go-applications)

`ld` stands for linker, the program that links together the different pieces of the compiled source code into the final binary.

`ldflags`, stands for linker flags. It passes a flag to the underlying Go toolchain linker, [cmd/link](https://links.jianshu.com/go?to=https%3A%2F%2Fgolang.org%2Fcmd%2Flink), that allows you to change the values of imported packages at build time from the command line.

使用方式为：

```bash
# 格式
go build -ldflags="-flag"
# 示例
go build -ldflags="-X 'package_path.variable_name=new_value'"
# 实例
go build -ldflags="-X 'main.Version=v1.0.0'"
#复杂实例
go build -v -ldflags="-X 'main.Version=v1.0.0' -X 'app/build.User=$(id -u -n)' -X 'app/build.Time=$(date)'"
```

### 规范

-   **外层使用双引号，确保传递的flag中的内容即使包含空格也不截断命令；**
-   **key-value值使用单引号**
-   **要改变的变量需要是包级别的string类型变量。不能是const类型**
-   **变量是否export都可以（大小写开头的变量都支持）**



在make文件main.go

```go
package main

var (
	version   string
	date      string
	goversion string
)

func init() {
	if version == "" {
		version = "no version"
	}
	if date == "" {
		date = "(Mon YYYY)"
	}
	if goversion == "" {
		goversion = "no go version"
	}
}

func main() {
	println(version, date, goversion)
}
```



脚本

```bash
version=0.0.1
d=`date "+(%b %Y)"`
exec=a.out

.PHONY: all

all:
	@echo "${version}"
	@echo "${d}"
	@echo " make <cmd>"
	@echo ""
	@echo "commands:"
	@echo " build          - runs go build"
	@echo " build_version  - runs go build with ldflags version=${version} & date=${d}"
	@echo ""

build: clean
	@go build -v -o ${exec}

build_version: check_version 
	@go build -v -ldflags "-X 'main.version=${version}' -X 'main.date=${d}' -X 'main.goversion=$(shell go version)'" -o ${exec}_${version}

clean:
	@rm -f ${exec}
	@rm -f ${exec}_${version}

check_version:
	@if [ -a "${exec}_${version}" ]; then \
		echo "${exec}_${version} already exists"; \
		rm -f ${exec}_${version}; \
	 fi;
```

Makefile

1、行首以tab 开头，否则报错Makefile:8: *** missing separator. Stop.

2、d=`date "+(%b %Y)"` 或 d=$(shell date "+(%b %Y)")

[Setting Go variables from the outside](https://links.jianshu.com/go?to=https%3A%2F%2Fblog.cloudflare.com%2Fsetting-go-variables-at-compile-time%2F)

在`go run`命令中也可以直接使用（因为会默认先执行`go build`)

```
go run -ldflags="-X main.who CloudFlare" hello.go
```

