---
title: Go 语言中进行目录遍历
date: 2022-01-21 10:50:31
tags:
---

Go 语言中进行目录遍历的原生方法主要是以下3种：
filepath.Walk()
ioutil.ReadDir()
os.File.Readdir()

性能是越底层越高（上层其实是对底层API的封装）。

------

filepath.Walk()遍历根目录(root)下的文件树，为树中的每个文件或目录(包括根目录)调用walkFn。所有在访问文件和目录时出现的错误都由walkFn过滤。遍历按词法顺序进行，这使得输出是确定的，但对于非常大的目录来说，遍历可能是低效的。filepath.Walk()不会跟进符号链接。

```go
package main

import (
    "flag"
    "fmt"
    "os"
    "path/filepath"
)

// https://rosettacode.org/wiki/Walk_a_directory/Recursively#Go

const (
    layout = "2006-01-02 15:04:05"
)

func VisitFile(fp string, fi os.FileInfo, err error) error {
    if err != nil {
        fmt.Println(err) // can't walk here,
        return nil       // but continue walking elsewhere
    }
    if fi.IsDir() {
        return nil // not a file.  ignore.
    }
    // 过滤输出内容
    matched, err := filepath.Match("*.txt", fi.Name())
    if err != nil {
        fmt.Println(err) // malformed pattern
        return err       // this is fatal.
    }
    if matched {
        // fmt.Println(fp)
        fmt.Printf("Name: %s, ModifyTime: %s, Size: %v\n", fp, fi.ModTime().Format(layout), fi.Size())
    }
    return nil
}

func main() {
    var path = flag.String("path", ".", "The path to traverse.")
    flag.Parse()

    filepath.Walk(*path, VisitFile)
}
```

 

**filepath.Walk()会自动遍历子目录**，但有些时候我们不希望这样，如果只想看当前目录，或手动指定某几级目录中的文件，这个时候，可以使用 ioutil.ReadDir 进行替代。

```go
package main

import (
    "flag"
    "fmt"
    "io/ioutil"
    "log"
)

func main() {
    var path = flag.String("path", ".", "The path to traverse.")
    flag.Parse()

    files, err := ioutil.ReadDir(*path)
    if err != nil {
        log.Fatal(err)
    }

    for _, file := range files {
        fmt.Println(file.Name())
    }
}
```

 

几个方法封装的一个演示和对比：

```go
package main

import (
    "fmt"
    "io/ioutil"
    "os"
    "path/filepath"
)

// https://stackoverflow.com/questions/14668850/list-directory-in-go/49196644#49196644

func main() {
    var (
        root  string
        files []string
        err   error
    )

    // root = "/home/manigandan/Desktop/Manigandan/sample"
    root = "."
    // filepath.Walk
    files, err = FilePathWalkDir(root)
    if err != nil {
        panic(err)
    }
    // ioutil.ReadDir
    files, err = IOReadDir(root)
    if err != nil {
        panic(err)
    }
    //os.File.Readdir
    files, err = OSReadDir(root)
    if err != nil {
        panic(err)
    }

    for _, file := range files {
        fmt.Println(file)
    }
}

func FilePathWalkDir(root string) ([]string, error) {
    var files []string
    err := filepath.Walk(root, func(path string, info os.FileInfo, err error) error {
        if !info.IsDir() {
            files = append(files, path)
        }
        return nil
    })
    return files, err
}

func IOReadDir(root string) ([]string, error) {
    var files []string
    fileInfo, err := ioutil.ReadDir(root)
    if err != nil {
        return files, err
    }

    for _, file := range fileInfo {
        files = append(files, file.Name())
    }
    return files, nil
}

func OSReadDir(root string) ([]string, error) {
    var files []string
    f, err := os.Open(root)
    if err != nil {
        return files, err
    }
    fileInfo, err := f.Readdir(-1)
    f.Close()
    if err != nil {
        return files, err
    }

    for _, file := range fileInfo {
        files = append(files, file.Name())
    }
    return files, nil
}
```