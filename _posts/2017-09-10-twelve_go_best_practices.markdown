---
title:  "12个Go最佳实践"
subtitle: "A best practice is a method or technique that has consistently shown results superior
to those achieved with other means"
author: "FineJade"
avatar: "img/authors/avatar.jpg"
image: "img/golang.png"
date:   2017-09-10 16:00:00
comments: true
---

### 1.避免代码嵌套
例子：
`
type Gopher struct {
    Name     string
    AgeYears int
}
func (g *Gopher) WriteTo(w io.Writer) (size int64, err error) {
    err = binary.Write(w, binary.LittleEndian, int32(len(g.Name)))
    if err == nil {
        size += 4
        var n int
        n, err = w.Write([]byte(g.Name))
        size += int64(n)
        if err == nil {
            err = binary.Write(w, binary.LittleEndian, int64(g.AgeYears))
            if err == nil {
                size += 4
            }
            return
        }
        return
    }
    return
}
`
改进：
`
func (g *Gopher) WriteTo(w io.Writer) (size int64, err error) {
    err = binary.Write(w, binary.LittleEndian, int32(len(g.Name)))
    // handle err first
    if err != nil {
        return
    }
    size += 4
    n, err := w.Write([]byte(g.Name))
    size += int64(n)
    if err != nil {
        return
    }
    err = binary.Write(w, binary.LittleEndian, int64(g.AgeYears))
    if err == nil {
        size += 4
    }
    return
}
`
*可读性更强

### 2.尽量避免重复代码
继续改进：
`
type Gopher struct {
        Name     string
        AgeYears int
}

type binWriter struct {
        w    io.Writer
        size int64
        err  error
}

// Write writes a value to the provided writer in little endian form.
func (w *binWriter) Write(v interface{}) {
        if w.err != nil {
                return
        }
        if w.err = binary.Write(w.w, binary.LittleEndian, v); w.err == nil {
                w.size += int64(binary.Size(v))
        }
}

func (g *Gopher) WriteTo(w io.Writer) (int64, error) {
        bw := &binWriter{w: w}
        bw.Write(int32(len(g.Name)))
        bw.Write([]byte(g.Name))
        bw.Write(int64(g.AgeYears))
        return bw.size, bw.err
}
`
*使用binWriter减少Write()和size++

### 3.优先重要代码
* License信息, build tags, 包文档
* 包导入 优先内部包，然后第三方包
`
/*
	License
*/
// +build !linux,!darwin !cgo
// main package
import (
    "fmt"
    "io"
    "log"

    "golang.org/x/net/websocket"
)
`

### 4.代码注释
`
// Package playground registers an HTTP handler at "/compile" that
// proxies requests to the golang.org playground service.
package playground

/ Author represents the person who wrote and/or is presenting the document.
type Author struct {
    Elem []Elem
}

// TextElem returns the first text elements of the author details.
// This is used to display the author' name, job title, and company
// without the contact details.
func (p *Author) TextElem() (elems []Elem)
`
*可以生成代码文档  命令godoc

### 5.命名尽量简短
* 尽量用短的能解释清楚的词
  MarshalIndent比MarshalWithIndentation好
* 包名出现在标识前面，因此标识中不在需要包含包相关信息
  encode/json中的Encoder不应该用JSONEncoder(json.Encoder vs json.JSONEncoder)

### 6.Package比较大可拆成多个文件
* 将比较大的包，拆成多个文件。
  eg.标准库中 net/http 包有47个文件，共计 15734 行
* 分开代码文件和测试文件。
  net/http/cookie.go和net/http/cookie_test.go都属于包net/http
* 一个包有多个文件时，很方便创建doc.go来包含包文档注释

### 7.让Package可以通过go get方式获取
* 有些包可能被复用，有些包则不会

### 8.明确需求
让我们以之前的Gopher类型为例
`
type Gopher struct {
    Name     string
    Age      int32
    FurColor color.Color
}

// 我们定义这个方法
func (g *Gopher) DumpToFile(f *os.File) error{}

// 但是使用一个具体的类型会让代码难以测试，因此我们使用接口.
func (g *Gopher) DumpToReadWriter(rw io.ReadWriter) error {}

// 进而，使用接口，我们可以只请求我们需要的.
func (g *Gopher) DumpToWriter(f io.Writer) error {}
`

### 9.保持包独立性
`
import (
    "golang.org/x/talks/2013/bestpractices/funcdraw/drawer"
    "golang.org/x/talks/2013/bestpractices/funcdraw/parser"
)
// Parse the text into an executable function.
f, err := parser.Parse(text)
if err != nil {
    log.Fatalf("parse %q: %v", text, err)
}

// Create an image plotting the function.
m := drawer.Draw(f, *width, *height, *xmin, *xmax)

// Encode the image into the standard output.
err = png.Encode(os.Stdout, m)
if err != nil {
    log.Fatalf("encode image: %v", err)
}
`
*radix.v2[https://github.com/mediocregopher/radix.v2](https://github.com/mediocregopher/radix.v2)

### 10.避免方法内并发
方法内并发例子：
`
func doConcurrently(job string, err chan error) {
    go func() {
        fmt.Println("doing job", job)
        time.Sleep(1 * time.Second)
        err <- errors.New("something went wrong!")
    }()
}

func main() {
    jobs := []string{"one", "two", "three"}

    errc := make(chan error)
    for _, job := range jobs {
        doConcurrently(job, errc)
    }
    for _ = range jobs {
        if err := <-errc; err != nil {
            fmt.Println(err)
        }
    }
}
`
我想要串行执行怎么办？
改进：
`
func do(job string) error {
    fmt.Println("doing job", job)
    time.Sleep(1 * time.Second)
    return errors.New("something went wrong!")
}

func main() {
    jobs := []string{"one", "two", "three"}

    errc := make(chan error)
    for _, job := range jobs {
        go func(job string) {
            errc <- do(job)
        }(job)
    }
    for _ = range jobs {
        if err := <-errc; err != nil {
            fmt.Println(err)
        }
    }
}
`
*既可以并发调用，也可以串行调用

### 11.使用goroutines管理状态
例子：
`
type Server struct{ quit chan bool }

func NewServer() *Server {
    s := &Server{make(chan bool)}
    go s.run()
    return s
}

func (s *Server) run() {
    for {
        select {
        case <-s.quit:
            fmt.Println("finishing task")
            time.Sleep(time.Second)
            fmt.Println("task done")
            s.quit <- true
            return
        case <-time.After(time.Second):
            fmt.Println("running task")
        }
    }
}

func (s *Server) Stop() {
    fmt.Println("server stopping")
    s.quit <- true
    <-s.quit
    fmt.Println("server stopped")
}

func main() {
    s := NewServer()
    time.Sleep(2 * time.Second)
    s.Stop()
}
`
*用channel或者包含channel的结构体和Goroutine通讯

### 12.避免goroutine内存泄露
* 内存泄漏例子：
`
func broadcastMsg(msg string, addrs []string) error {
    errc := make(chan error)
    for _, addr := range addrs {
        go func(addr string) {
            errc <- sendMsg(msg, addr)
            fmt.Println("done")
        }(addr)
    }

    for _ = range addrs {
        if err := <-errc; err != nil {
            return err
        }
    }
    return nil
}
`
*goroutine阻塞在写操作上
*goroutine保持对channel的引用
*channel不会被垃圾回收器GC回收

* 使用带缓存的channel处理
`
func broadcastMsg(msg string, addrs []string) error {
    errc := make(chan error, len(addrs))
    for _, addr := range addrs {
        go func(addr string) {
            errc <- sendMsg(msg, addr)
            fmt.Println("done")
        }(addr)
    }

    for _ = range addrs {
        if err := <-errc; err != nil {
            return err
        }
    }
    return nil
}
`
*如果不知道channel大小，该如何处理？

* 使用quit channel
`
func broadcastMsg(msg string, addrs []string) error {
    errc := make(chan error)
    quit := make(chan struct{})

    defer close(quit)

    for _, addr := range addrs {
        go func(addr string) {
            select {
            case errc <- sendMsg(msg, addr):
                fmt.Println("done")
            case <-quit:
                fmt.Println("quit")
            }
        }(addr)
    }

    for _ = range addrs {
        if err := <-errc; err != nil {
            return err
        }
    }
    return nil
}

`

### 参考
* 视频[https://www.youtube.com/watch?v=8D3Vmm1BGoY](https://www.youtube.com/watch?v=8D3Vmm1BGoY)
* Slide[https://talks.golang.org/2013/bestpractices.slide#1](https://talks.golang.org/2013/bestpractices.slide#1)


