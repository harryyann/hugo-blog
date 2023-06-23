---
title: "Golang的channel底层实现"
date: 2021-09-18T20:01:29+08:00
Tags: [golang]
Categories: [golang]
draft: false
---

# 一、Channel的底层数据结构

channel是golang在语言层面提供的goroutine通信机制，也是我们经常使用的数据结构，它的数据结构如下：

```go
// 在src/runtime/chan.go中定义

type hchan struct{
    // 环形队列相关
    qcount uint         // 当前队列中剩余元素的个数
    dataqsiz uint       // 环形队列的长度，即可以存放的元素个数
    buf unsafe.Pointer  // 环形队列的指针
    sendx uint          // 队列下标，指示元素写入时存放到队列中的位置
    recvx uint          // 队列下标，指示元素从队列的该位置处读出
    
    // 元素信息
    elemtype *_type     // 元素类型
    elemsize uint16     // 每个元素的大小
    
    closed uint32       // 标识关闭状态
    
    // 等待队列相关
    recvq waitq         // 等待读消息的goroutine队列
    sendq waitq         // 等待写消息的goroutine队列
    
    lock mutex          // 互斥锁，chan不允许并发读写
}
```

如上，chan的字段可以分成5类

- 环形队列相关
- 存储消息的信息
- 等待队列相关
- 关闭状态的标识
- 互斥锁

## 环形队列

chan内部使用了一个环形队列作为其缓冲区(buf字段)，队列的长度就是创建chan时指定的长度(dataqsiz字段)：

![](/images/chan/buffer.png)

- dataqsize指示了队列长度为6，即可缓存6个元素。超过6个就得阻塞，等有空位了才能插入。

- buf指向这个环形队列的内存地址。
- qcount表示队列中还有几个空位。
- sendx指示后续写入的数据存储的位置，长度为6的话取值就是0-5。
- recvx表示chan接收端从该位置处读取数据，取值也是0-5。

## 等待队列

当从channel中读数据，而chan中没数据，或往chan中写数据，而chan中没有空位时(即qcount=0)，当前这个goroutine会被阻塞。被阻塞的goroutine将会挂在channel的等待队列中。

等待队列有两个：recvq就是读阻塞的等待队列，sendq就是写阻塞的等待队列。

- 因读阻塞的goroutine将会被向chan中写入数据的goroutine唤醒

  读阻塞是因为channel中没数据，当有goroutine向chan中写入数据时，读操作自然就解阻塞了。

- 因写阻塞的goroutine将会被从chan中读数据的goroutine唤醒

  写阻塞是因为channel满了，写不了，当有goroutine从chan中读出一个数据时，自然就有空位去写了。

![](/images/chan/wait.png)

如上图，每个G都是一个goroutine，这几个goroutine都挂在recvq这个读等待队列中，即chan中没数据，所有的goroutine想从chan中读时都阻塞在这了。

一般情况下，recvq和sendq中至少一个为空的，即要么都在阻塞读，要么都在阻塞写。

## 消息的相关信息

chan结构体中有两个字段用于标识chan中存储的消息信息：elemtype和elemsize

- elemtype代表消息的数据类型，用于数据传递过程中的赋值
- elemsize代表消息的大小，用于在buf中定位元素的位置

## 锁

lock字段是一个mutex，明显channel是并发安全的。channel同一时刻仅允许被一个goroutine读写，每次读写都要先获取到锁。用完了再释放。

# channel的读写过程

## 写数据的过程

写数据可以分成三种情况：有goroutine在读阻塞、没有goroutine在读阻塞、缓冲区满了。

#### 有goroutine读阻塞

当读等待队列recvq不为空，说明有goroutine在阻塞，此时说明缓冲区没有数据或者这个channel没有缓冲区，这种情况下写入时会从recvq中取出一个G，然后把数据写入到缓冲区，最后把这个G唤醒，结束发送过程。

即该情形下，不仅要写入缓冲区，还要唤醒阻塞的goroutine。

#### 没有goroutine读阻塞

这种情况比较简单，没有读阻塞就说明缓冲区有空余位置，直接将数据写入缓冲区，就可以结束发送过程。

#### 缓冲区满了

如果缓冲区没有空余位置，则此时写入时就是写阻塞，需要将当前G加入到sendq中，同时该goroutine进入睡眠状态，等待被唤醒，被唤醒后才能接着写。

![](/images/chan/write.png)



# 二、读数据的过程

读数据分为四种情况：有写阻塞且chan没有缓冲区、有写阻塞且chan有缓冲区、没有写阻塞且缓冲区中有数据、没有写阻塞且缓冲区中无数据。

#### 有写阻塞且chan没有缓冲区

chan没有缓冲区的情况，每个写入都会是写阻塞，每个读取都是读阻塞，此时不管是要写数据还是读数据，都需要另一个goroutine同时在读或写。换句话说，没有缓冲区的chan的读写，实际上就是一个goroutine把数据直接交给了另一个goroutine。

所以这种情况下读数据，一定要有另一个goroutine在写阻塞中才能读到，此时本goroutine会直接从sendq中取出第一个写阻塞的G，从G中取出数据，然后把G唤醒，即可。

#### 有写阻塞且chan有缓冲区

chan有缓冲区，且写阻塞，此时一定是chan的缓冲区满了，此时需要先从缓冲区的头部取出一个数据，然后把sendq中第一个写阻塞的G的数据写入到缓冲区的尾部，最后唤醒这个G即可。

#### 没有写阻塞且缓冲区中有数据

这种是最简单的情况，也是最常见的情况，这种情况直接从缓冲区头部取出一条数据，即可。

#### 没有写阻塞且缓冲区中无数据

这种情况下可能是刚开始创建chan时，还没有goroutine向chan中写入数据，此时从chan中读取数据就会被阻塞，这个goroutine会把自己的G写入到recvq中，同时自己进入睡眠状态，等待被写操作的G唤醒。



![](/images/chan/read.png)

# 三、Channel的关闭的问题

关闭channel时会对等待队列中的G做如下操作：

- 把读阻塞recvq中的G全部唤醒，并向这些G中发送chan数据类型的0值。
- 把sendq中的G全部唤醒，但是这些写入的G都会panic。

## channel引发panic的场景

- 关闭一个值为nil的channel
- 关闭已经关闭的channel
- 向已经关闭的channel写数据

## 从一个已关闭的channel读数据的情形

- 如果这个chan中还有值，那么会把这些值读完
- 如果这个chan的值已被读完，那么依然可以读取到这个chan中存储数据类型的基本值，如int会读出0，string会读出""，bool会读出false

从chan中读数据可以用两个变量接收，第二个变量为bool值，表示接收到的值是不是有效的，为true是有效：

```go
c, ok := <- channel
if ok{
    有效
}
```

# 四、Channel的常见用法

## 单向chan

指只用于发送，或者只用于接收的chan，实际上没有单向的chan，这里说的只是使用限制，即限制一个chan在一个函数内为只读或者只写，通过函数传参时指定：

```go
func readChan(c <- chan int){} // 通过形参指定函数中只能读c的数据
func writeChan(c chan<- int){} // 通过形参限定函数内部只能写入c 
```

## 配合select多路复用

select多路复用可以监控多个channel，当其中某个chan有数据时，就从中读取数据：

```go
for{
    select {
    case a := chan1:
        // do something
    case b := chan2:
        // do something
    default:
        // if chan1 and chan2 has block, go this way
    }
}
```

## 配合range

通过range可以持续的从一个chan中取出数据，像是在遍历一个数组一样，当channel中没有数据时则会阻塞在某一次读取中

```go
for e := range chan1{
    fmt.Println(e)
}
```

