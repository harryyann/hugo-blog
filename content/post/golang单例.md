---
title: "Golang实现单例模式"
date: 2021-07-13T19:28:17+08:00
Tags: [golang,编程模式]
Categories: [golang]
draft: false
---

## 单例模式

单例模式是保证一个类，确保在一个系统中永远只有一个实例。在go中，就是保证一个结构体，永远只有一个实例化的对象。

当一个应用程序中某个对象需要是全局唯一的，就要使用单例模式。单例模式常用在以下场景中：

- 配置类
- 日志类
- 必须要以共享模式访问的资源类
- 需要频繁的创建和删除对象，而对象又可以复用的场景

单例模式不适用的场景：

- 需要频繁变化的对象
- 为了节省资源将连接池设为单例模式，可能会导致连接不够用

## Golang中实现单例模式的方式

### 一、懒汉模式

由于golang语言的特点，一个最简单的单例实现方式就是通过一个全局变量保存对象指针，然后除了在初始化的时候创建对象，其他时候都是直接获取这个全局变量即可。如下：

```go
// 定义一个全局变量
var single *Person

type Person struct {
    name string
    age uint
}

func GetInstance()*Person{
    // single不为nil就直接返回single，否则就初始化single
    if single != nil{
        return single
    }
	single = &Person{}
    return single
}
```

懒汉模式的缺点，并发不安全：

- 如果全局变量需要被修改的话，这个并发操作是不安全的，比如上面这个例子，多个goroutine同时覆写single对象的age参数，可能就会导致最终的结果不符合我们的要求。更严重的是如果是多个goroutine同时初始化single对象，由于GetInstance函数没有锁，那么也有可能创建出了多个single对象并相互覆盖，这就很糟糕了。

### 二、带锁的懒汉模式

由懒汉模式的缺点可以很容易想到，既然全局变量并发有问题，那么给它加锁不就行了。如下：

```go
var single *Person
var mutex sync.Mute

type Person struct {
    name string
    age uint
}

func GetInstance()*Person{
    mutex.Lock()
    defer mutext.Unlock()
    if single != nil{
        return single
    }
    single = &Person{}
    return single
}
```

如上，获取single实例时，加上锁

但是这种加锁的方式也有缺点：

- 如果single已经初始化过了，其实没必要加锁的，这样就导致了所有goroutine执行GetInstance时都变成了串行。

### 三、加锁模式的优化

既然但single已经被初始化时没必要加锁，那我就在需要的地方加锁不就好了，这里要介绍以下C++的**Check-Lock-Check模式**：

```c++
if check(){
	lock(){
		if check(){
			// 执行锁住的代码
		}
	}
}
```

这种模式的想法是，先检查，再加锁，并在锁住的操作中再次检查。原因是如果两个goroutine同时通过了第一次check()，那么如果在lock()后不加第二次check()，那么在其中一个goroutine在执行完lock中的内容并释放锁之后，第二个goroutine还会拿到lock()并覆写第一个goroutine的初始化操作。

那么我们的代码就可以改成：

```go
var single *Person
var mutex sync.Mute

type Person struct {
    name string
    age uint
}

func GetInstance()*Person{
    if single == nil{
        mutex.Lock()
    	defer mutext.Unlock()
        // 在拿到锁之后，再检查一次，防止多个goroutine同时执行到这里，轮着覆写single
    	if single != nil{
        	single = &Person{}
    	}
    }   
    return single
}
```

但是加锁模式还是有问题，因为给single对象赋值的代码并不是原子操作，所以判断`single==nil`时可能有其他goroutine已经开始写single了。

### 四、利用原子操作

对于不是原子操作的问题，我们可以用一个原子标志位来解决：

```go
var single *Person
var initialized uint32
var mutex sync.Mutex

type Person struct {
    name string
}

func GetInstance()*Person{
    // 原子读操作
    if atomic.LoadUInt32(&initialized) == 0{
        mutex.Lock()
    	defer mutex.Unlock()
    	if atomic.LoadUInt32(&initialized) == 0{
        	single = &Person{}
        	// 原子写，设置single已经初始化
        	atomic.StoreUint32(&initialized, 1)
    	} 
    }
    return single
}
```

这种方式唯一的缺点就是代码多，然后需要维护的全局变量也多，也不好读。

### 五、使用sync.Once

于是我们隆重介绍golang中官方推荐的实现单例模式的方式：`sync.Once`，它的底层实现基本就是上面那种利用原子操作的方式，但是代码十分简单：

```go
var single *Person
var once sync.Once

func GetInstance()*Person{
    once.Do(func(){
        single = &Person{}
    })
    return single
}
```

**优势：**

- 保证变量永远只会被初始化一次
- 不会有并发冲突
- 非常好写

**原理介绍：**

`sync.Once`的底层数据结构：

```go
type Once struct {
    // done indicates whether the action has been performed.
    // It is first in the struct because it is used in the hot path.
    // The hot path is inlined at every call site.
    // Placing done first allows more compact instructions on some architectures (amd64/x86),
    // and fewer instructions (to calculate offset) on other architectures.
    done uint32  // 标志位
    m    Mutex   // 互斥锁
}
```

可以看到就是我们之前手写时创建的一个原子操作标志位和一个互斥锁。通过原子标志位来判断是否是否已经初始化，通过锁来保证并发安全。

