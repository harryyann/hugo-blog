---
title: "Golang的mutex底层实现"
date: 2021-10-14T20:01:29+08:00
Tags: [golang]
Categories: [golang]
draft: false
---

# 互斥锁

## 互斥锁的数据结构

互斥锁是并发程序中对一个共享资源进行访问控制的常用手段，Golang中提供了mutex作为互斥锁的实现。mutex的数据结构如下：

```go
// 定义在src/sync/mutex.go

type Mutex struct{
    state int32
    sema uint32
}
```

包括两个字段：

- state：表示互斥锁的状态，比如是否被锁定等。
- sema：表示信号量，想要占用这个锁的协程会等待该信号量，解锁的协程会释放该信号量从而唤醒等待信号量的协程。

## Mutex的内存布局

![](/images/mutex/layout.png)

state字段是个32位的整型变量，内部实现时把该变量分成四份，用于记录mutex的四种状态，如上图。

- Locked：表示该mutex是否已被锁定，0没有锁定，1已被锁定。
- Woken：表示是否有协程已被唤醒，0没有协程唤醒，1已有协程被唤醒，正在加锁过程中。
- Starving：表示该mutex是否处于饥饿状态，0没有饥饿，1饥饿状态，说明有协程阻塞了超过1ms。
- Waiter：表示处于阻塞中等待锁的协程个数，协程解锁时根据此值来判断是否需要释放信号量。

goroutine之间抢占锁，实际上就是抢给locked赋值的权力，给Locked域置为1，则说明抢锁成功，抢不到就等待sema信号量，一旦持有锁的协程解锁，等待的协程会依次被唤醒。woken和Starving主要用于控制协程间的抢锁过程。

## Mutex加锁解锁的过程

#### 简单加锁

假设当前只有一个协程在加锁，没有其他协程干扰：

![](/images/mutex/simple.png)

加锁过程会判断Locked标志位是否为0，如果是0则把Locked位置为1，代表加锁成功，其他状态位没变化。



#### 加锁时锁已被占

![](/images/mutex/occupy.png)

当A已经占用了锁以后，B也想要占用锁时，waiter计数器增加了1，此时协程B会被阻塞，直到Locked变为0后才会被唤醒

#### 简单解锁

解锁时没有其他协程阻塞：

![](/images/mutex/simple_unlock.png)

由于没有其他协程阻塞等待加锁，所以此时解锁只需要把Locked位置置为0即可，不需要释放信号量。

#### 解锁时有其他goroutine阻塞

需要将其他goroutine唤醒：

![](/images/mutex/occupy_unlock.png)

协程A解锁需要分两个步骤：

- 一是把Locked位置为0。
- 二是查看到Waiter>0，所以释放一个信号量，唤醒一个阻塞的协程，被唤醒的协程B把Locked位置为1，于是B获得锁。

## 自旋

自旋时CPU会执行`PAUSE`指令，CPU对该指令什么都不做，相当于CPU空转，对程序而言相当于sleep了一会，当前实现是30个时钟周期。每次PAUSE后探测一次看看Locked是否被解锁。

#### 自旋的过程

1. 在尝试加锁时，如果当前该锁的Locked位为1，说明该锁被其他协程持有，尝试加锁的协程并不是马上转入阻塞，而是会通过轮询的方式持续探测该mutex的Locked位是否变为0，这个过程叫做自旋过程。
2. 自旋的持续时间很短，但如果在自旋过程中发现锁已经被释放，那么该协程可以立刻获得锁，此时即便有其他协程被唤醒也无法获取到锁只能再次被阻塞。

自旋的好处是，当加锁失败时不必立刻转入阻塞，有一定机会可以不久之后获取到锁，可以避免协程的切换。

#### 自旋的条件

无限制的自旋会给CPU带来巨大压力，所以自旋的条件至关重要：

- 自旋次数要足够小，通常为4次，即cpu会`PAUSE`四次去探测。
- CPU核数要大于1，否则自旋没意义，因为如果CPU只有一颗，此时其他协程不可能释放锁。
- 协程调度机制中的Process数要大于1，比如使用`GOMAXPROCS()`将处理器设置为1就不能启用自旋。
- 协程调度机制中的可运行队列必须为空，否则会延迟协程调度。

总而言之，协程自旋的条件是十分苛刻的，一定要在不忙的时候才会启用自旋。



#### 自旋的优势

充分利用CPU，避免协程切换，因为**当前申请加锁的协程拥有CPU**，如果经过短时间的自旋可以获得锁，当前协程就可以继续运行，不必进入阻塞状态，换成另一个协程，增加了上下文切换。

#### 自旋的问题

- 自旋过程中能抢到锁，一定意味着同一时刻有协程释放了锁。而当锁被释放时还会释放一个信号量来唤醒一个等待的协程，被唤醒的协程得到cpu后开始运行，此时发现协程已经被抢占了，所以只好再次阻塞，不过阻塞时会判断上次阻塞到本次阻塞经过了多少时间，如果超过1ms，就会将mutex标记为Starving状态。然后再阻塞这个协程。
- 当mutex被标记为饥饿状态，则会取消自旋过程，当当前协程交出锁以后，会按顺序唤醒协程，而新协程不会自旋，而是直接排队，新协程被唤醒后，mutex的等待计数也会减1。

## Mutex的模式

每个Mutex都有两个模式：**normal**和**starving**。

- normal模式：

  默认情况下为normal模式，协程如果加锁不成功不会立刻转入阻塞排队，而是判断是否满足自旋的条件，如果满足则会启动自旋，尝试抢锁。

- starving模式：

  自旋过程中能抢到锁，一定意味着同一时刻有协程释放了锁。而当锁被释放时还会释放一个信号量来唤醒一个等待的协程，被唤醒的协程得到CPU后开始运行，此时发现协程已经被抢占了，所以只好再次阻塞，不过阻塞时会判断上次阻塞到本次阻塞经过了多少时间，如果超过1ms，就会将mutex标记为starving状态。然后再阻塞这个协程。

  当mutex被标记为饥饿状态，则会取消自旋过程，当当前协程交出锁以后，会按顺序唤醒协程，而新协程不会自旋，而是直接排队，新协程被唤醒后，mutex的等待计数也会减1。

## state的woken状态

woken状态用于加锁和解锁过程的通信。比如，两个协程一个在解锁，而另一个通过自旋后正在加锁，此时把woken标记为1，用于通知解锁协程不用释放信号量唤醒其他协程了，这个锁我要了，你解完锁就完事了。

## 为什么重复解锁要panic

unlock的过程分为将locked置为0，然后判断waiter的值，waiter>0则释放信号量唤醒其他协程。如果多次unlock，那么可能每次都释放一个信号量，这样会唤醒多个协程，多个协程唤醒后会继续在lock()的逻辑下争抢锁，那么Lock()会增加复杂度，并且还有不必要的协程切换。

