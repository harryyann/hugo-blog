---
title: "Golang编程模式：functional options"
date: 2021-07-12T19:28:17+08:00
Tags: [golang,编程模式]
Categories: [golang]
draft: true
---

## 配置选项问题

当我们需要创建一个对象时，往往需要对这个对象的某些成员参数进行配置，比如下面这个结构体：

```go
type Server struct {
    Addr string
    Port int
    Protocol string
    Timeout time.Duration
    MaxConns int
    TLS *tls.Config
}
```

最简单的方式是直接调用这个结构体，直接填入参数：

```go
s := Server{
    Addr: "127.0.0.1",
    Port: 80,
}
```

如果并不想对外直接暴露结构体，那么可以写一个工厂方法返回一个*Server对象：

```go
func NewDefaultServer(addr string, port int) (*Server, error) {
 return &Server{
     addr, port, "tcp", 30 * time.Second, 100, nil}, nil 
}
```

针对不同的配置方式方式，可能需要写多个不同的工厂方法，非常麻烦：

```go
func NewTLSServer(addr string, port int, tls *tls.Config) (*Server, error) { 
    return &Server{addr, port, "tcp", 30 * time.Second, 100, tls}, nil 
}

func NewServerWithTimeout(addr string, port int, timeout time.Duration) (*Server, error) {
    return &Server{addr, port, "tcp", timeout, 100, nil}, nil 
}
```

解决方式主要有以下几种：

## 通过配置对象解决配置选项的问题

创建一个关于Server配置的结构体，Server中只保留必要的参数

```go
type Config struct { 
    Protocol string 
    Timeout time.Duration 
    Maxconns int 
    TLS *tls.Config 
}

type Server struct { 
    Addr string 
    Port int 
    Conf *Config 
}
```

然后编写一个接收必填参数，和可选的Config参数的函数即可

```go
func NewServer(addr string, port int, conf *Config) (*Server, error) {
    if conf != nil{{
         return &Server{addr, port, conf.Protocol, conf.Timeout, conf.Maxconns, conf,TLS}
    }
    return &Server{addr, port}
}
```

调用时就可以根据情况决定是否传入Config中的参数

```go
s := NewServer("127.0.0.1", 80, nil)
```

但是这种方式并不是十分优雅。

## Builder模式

Builder模式是从java中借鉴过来的，可以通过链式函数调用的方式进行对象的配置。

首先创建一个类（结构体）用来包装Server

```go
type ServerBuilder struct{
    Server
}
```

然后通过一些方法可以给内部的Server赋值

```go
func (sb *ServerBuilder) Create(addr string, port int) *ServerBuilder {
    sb.Server.Addr = addr 
    sb.Server.Port = port 
    //其它代码设置其它成员的默认值 
    return sb 
}

func (sb *ServerBuilder) WithProtocol(protocol string) *ServerBuilder {
    sb.Server.Protocol = protocol 
    return sb 
}

// 其他参数的方法，都是一个模子

// 最后有一个build方法返回内部赋值过的Server对象
func (sb *ServerBuilder) Build() (Server, err) {
    return sb.Server, nil
}
```

最后就可以用链式的方式使用了：

```go
s, err := ServerBuilder{}.Create("127.0.0.1", 80).WithProtol("tcp").WithMaxConn(1024).Build()
```

也可以不使用ServerBuilder对象，或者直接在Server对象上编写这些方法并且错误放在Server结构体中，即Server中加一个Error的参数：

```go
type Server struct{
    ...
    Error error 
}
```

然后调用方式就成了

```go
s := Server{}.Create("127.0.0.1", 80).WithProtocol("udp").Build()
if s.Error != nil {
    // handle error    
}
```

这种方式就已经很好了，但还不是这篇文章的主角：functional option。

## Functional options模式

