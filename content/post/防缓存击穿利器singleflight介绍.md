---
title: "防缓存击穿利器singleflight介绍"
date: 2021-11-21T10:18:27+08:00
Description: ""
Tags: [golang,缓存击穿]
Categories: [golang]
DisableComments: false
draft: false
---

# 缓存击穿



一般一个并发量大的服务，往往会在服务和数据库之间增加一层缓存，来减轻数据库的压力，因为数据库在高并发时往往会是整个系统的瓶颈。大概的过程就是查询时，先从缓存中查询，缓存中查不到，则从数据库中查询，并把数据回填到缓存中，再返回给用户，那么下次用户访问同一份数据时，就可以命中缓存而不用访问数据库，以此减轻数据库的压力。而在系统刚启动时一般也会采用**缓存预热**的方式，把数据库中的一些可能的热点数据预先加载到缓存中，以防止系统刚刚启动时数据库的压力过大。

而缓存中的key往往都需要有过期时间，当缓存中(Redis或者Memcached)的某个热点key，在承受大量并发时突然过期，导致查询这个key的所有请求穿透缓存到达数据库，增大数据库的压力，严重时甚至可能让数据库宕机，这就是**缓存击穿**。就像下图这样，所有请求都透传到了数据库：

![](/images/singleflight/problem.png)

解决这个问题比较重型的武器是分布式锁，即缓存miss后访问数据库需要先获取分布式锁，拿到分布式锁后才能从数据库中查到想要的数据，没拿到分布式锁就需要等待其他占有锁的线程完成从数据库中取数据和缓存的回填操作后，再从缓存中获取数据并返回。

但是这种解决方式消耗资源过大，且并且不是每个系统都需要分布式锁。[singleflight](https://pkg.go.dev/golang.org/x/sync/singleflight)就是一个解决该问题的利器。

# Singleflight的使用

singleflight 库，直译过来就是单飞，这个库的主要作用就是将一组相同的请求合并成一个请求，实际上只会去请求一次，然后对所有的请求返回相同的结果即可。当并发访问数据库中某个数据时，可以通过singleflight保证在一定的时间周期内，同一个key的并发访问实际上只有一次请求真正访问数据库，以此来大大降低数据库的压力，如下图：

![](/images/singleflight/singleflight.png)

## 使用方式

首先获取这个包

```bash
$ go get golang.org/x/sync/singleflight
```

Singleflight的全部代码只有一个文件`golang.org/x/sync/singleflight/singleflight.go`，十分简洁。

核心的结构体是**Group**，它有如下三个方法：

```go
// Do():  相同的 key，fn 同时只会执行一次，返回执行的结果给fn执行期间，所有使用该 key 的调用
// v: fn 返回的数据
// err: fn 返回的err
// shared: 表示返回数据是调用 fn 得到的还是其他相同 key 调用返回的
func (g *Group) Do(key string, fn func() (interface{}, error)) (v interface{}, err error, shared bool) {
    
// DoChan(): 类似Do方法，以 chan 返回结果
func (g *Group) DoChan(key string, fn func() (interface{}, error)) <-chan Result {
// Forget(): 失效 key，后续对此 key 的调用将执行 fn，而不是等待前面的调用完成
func (g *Group) Forget(key string)
```

- Do()：是执行方法，接收一个key，一个func()，对于同一个key，func()只会执行一次。func()方法返回一个interface{}和一个error
- DoChan()：和Do类似，但是是以chan返回结果
- Forget()：失效一个key，后续对这个key调用Do()时，还要执行func()，而不是等待其他的Do调用完成

#### Do()方法的使用

```go
func main() {
	wg := sync.WaitGroup{}

	// 定义一个Group对象
	g := singleflight.Group{}
	
	key := "test-key"

	// 开10个goroutine，并发调用Do方法
	for i := 1; i < 10; i ++{
		wg.Add(1)
		go func(i int){
			v, err, shared := g.Do(key, func() (interface{}, error) {
				// 每一个执行方法，都打印一下key，和协程号
				fmt.Println(key, i)
				return i, nil
			})
			if err != nil{
				fmt.Println(err)
				wg.Done()
				return
			}
			fmt.Printf("goroutine: %d, v: %v, shared: %v\n", i, v, shared)
			wg.Done()
		}(i)
	}
	
	// 等待所有协程执行完
	wg.Wait()	
}
```

上面这个程序的打印结果是这样的：

```go
test-key 5
goroutine: 5, v: 5, shared: true
goroutine: 3, v: 5, shared: true
goroutine: 1, v: 5, shared: true
goroutine: 6, v: 5, shared: true
goroutine: 2, v: 5, shared: true
goroutine: 9, v: 5, shared: true
goroutine: 8, v: 5, shared: true
goroutine: 7, v: 5, shared: true
goroutine: 4, v: 5, shared: true
```

可以看到，只有5号 goroutine 执行了打印方法，所有的goroutine拿到的v都是5，都是共享的。

但偶尔也会有这样的打印：

```go
test-key 9
goroutine: 9, v: 9, shared: true
goroutine: 8, v: 9, shared: true
goroutine: 4, v: 9, shared: true
goroutine: 3, v: 9, shared: true
goroutine: 1, v: 9, shared: true
goroutine: 6, v: 9, shared: true
goroutine: 5, v: 9, shared: true
goroutine: 7, v: 9, shared: true
test-key 2
goroutine: 2, v: 2, shared: false
```

9 号 goroutine和 2 号 goroutine执行了方法，且 2 号 goroutine 没有和其他 goroutine 共享，这就说明Do()方法自己是有一定超时控制的，超过一定时间后将自动失效掉这个key。

#### DoChan()的使用：

Do()的问题是通过阻塞的方式来控制下游请求，如果某一个得到数据的协程执行时间过长，就有可能导致其他协程超时，造成大量协程错误。因此singleflight提供了DoChan()方法，用channel来返回结果，因此就可以通过select多路复用的方法实现超时控制：

```go
// 其中一个协程
ch := g.DoChan(key, func() (interface{}, error) {
 	// do execution stuff... 
    return v, err
})

// 设置500ms超时时间
timeout := time.After(500 * time.Millisecond)

// 保存结果
var result singleflight.Result

select {
	case <- timeout:
		// 超时控制
		return
	case result = <- ch:
		// 接收到协程的执行结果
		...
}
```

#### Forget()的使用：

由于Do()或者DoChan()方法，都能控制一定时间内只有一个协程执行下游服务的请求，其他协程等待这个协程，但某些情况下，需要降低一下这个等待时间，比如当一个协程超过了10ms还没等到其他协程的结果时，就失效这个key，让另一个请求不再等待，可以发过去，稍微提高下游的一点并发，保证请求的成功率：

```go
v, _, shared := g.Do(key, func() (interface{}, error) {
    go func() {
    	// 执行时开一个新协程，等了超过10ms就让这个key失效
        time.Sleep(10 * time.Millisecond)
        fmt.Printf("Deleting key: %v\n", key)
        g.Forget(key)
    }()
    ......
   	// 这里执行访问下游服务的逻辑
    return v, err
})
```

# Singleflight的底层原理

#### 主要结构体

```go
// Group represents a class of work and forms a namespace in
// which units of work can be executed with duplicate suppression.
type Group struct {
	mu sync.Mutex       // protects m
	m  map[string]*call // lazily initialized
}

// call is an in-flight or completed singleflight.Do call
type call struct {
	wg sync.WaitGroup

	// These fields are written once before the WaitGroup is done
	// and are only read after the WaitGroup is done.
	val interface{}
	err error

	// forgotten indicates whether Forget was called with this call's key
	// while the call was still in flight.
	forgotten bool

	// These fields are read and written with the singleflight
	// mutex held before the WaitGroup is done, and are read but
	// not written after the WaitGroup is done.
	dups  int
	chans []chan<- Result
}

// Result holds the results of Do, so they can be passed
// on a channel.
type Result struct {
	Val    interface{}
	Err    error
	Shared bool
}
```

Group结构体包括两个参数，一个mutex互斥锁，和一个map[string]*call，map的键就是我们执行Do()时传入的key，指向一个call对象，看注释map这个参数是懒加载的。

call结构体是一个正在执行的或者已经完成的函数调用，这里存储调用的接口，错误等信息：

- wg：sync.WaitGroup对象，用来阻塞协程
- val：interface{}，记录结果，在wg.Wait()解除阻塞之前，只会被写入一次，保证“单飞”
- err：error，记录错误信息
- forgotten：bool，用来标识是否对key执行了Forget()方法，当这个call还在阻塞中时
- dups：int，用来记录执行次数，当wg完成前被更新，当wg完成后就只会被读取
- chans：[]chan<- Result：是系列记录Result的chan

Result结果体保存着执行结果，用于DoChan()方法。

#### Do函数

```go
// Do executes and returns the results of the given function, making
// sure that only one execution is in-flight for a given key at a
// time. If a duplicate comes in, the duplicate caller waits for the
// original to complete and receives the same results.
// The return value shared indicates whether v was given to multiple callers.
func (g *Group) Do(key string, fn func() (interface{}, error)) (v interface{}, err error, shared bool) {
    // 首先加了个锁，防止多个Do同时执行
	g.mu.Lock()
    
    // 然后是map的初始化，懒加载就体现在这里了
	if g.m == nil {
		g.m = make(map[string]*call)
	}
    
    // 检查key是否在map中存在，要存在就得实现单飞了
	if c, ok := g.m[key]; ok {
        
       	// 记录一下执行次数
		c.dups++
        
        // 然后解锁
		g.mu.Unlock()
        
        // 阻塞住
		c.wg.Wait()

        // 解除阻塞后，先检查一下错误，区分了两种错误
		if e, ok := c.err.(*panicError); ok {
			panic(e)
		} else if c.err == errGoexit {
			runtime.Goexit()
		}
        
        // 检查完错误，就可以返回了，这里面的流程实际上就是等，并没有真的访问下游
		return c.val, c.err, true
	}
    
    // 如果没有这个key，就创建一个call出来
	c := new(call)
    // 完后把wg加1，并把key指向这个call对象，然后解锁
	c.wg.Add(1)
	g.m[key] = c
	g.mu.Unlock()
	
    // 最后执行这个call，执行完了就返回
	g.doCall(c, key, fn)
	return c.val, c.err, c.dups > 0
}
```

然后看看执行call的doCall()方法怎么实现的：

```go
// doCall handles the single call for a key.
func (g *Group) doCall(c *call, key string, fn func() (interface{}, error)) {
    // 一开始是定义了两个变量，判断是doCall方法是否是正常执行的，和有没有recover
	normalReturn := false
	recovered := false

    // 这里的注释说，使用了两个defer来区分panic，实际上是要将本身的panic和我们传入的fn()执行时的panic区分开，避免死锁
	// use double-defer to distinguish panic from runtime.Goexit,
	// more details see https://golang.org/cl/134395
	defer func() {
        
        // 先判断一下，如果没有正常执行完，且也没有recover的话，就说明需要直接退出了，这种是致命的panic
		// the given function invoked runtime.Goexit
		if !normalReturn && !recovered {
			c.err = errGoexit
		}

        // 然后要操作map了，map并发不安全，得先锁上
		c.wg.Done()
		g.mu.Lock()
		defer g.mu.Unlock()
        // 如果说没有执行Forget(),就把这个key删了
		if !c.forgotten {
			delete(g.m, key)
		}
		
		if e, ok := c.err.(*panicError); ok {
            // 如果是panic类型的错误，且已经被recover捕获到了，而且还有chan等着数据呢，就永远阻塞住，
            // 要是没有chan，就是执行的Do()方法，就可以直接panic
			// In order to prevent the waiting channels from being blocked forever,
			// needs to ensure that this panic cannot be recovered.
			if len(c.chans) > 0 {
				go panic(e)
				select {} // Keep this goroutine around so that it will appear in the crash dump.
			} else {
				panic(e)
			}
		} else if c.err == errGoexit {
            // 要退出的话，就不用做什么操作了
			// Already in the process of goexit, no need to call again
		} else {
            // 如果不是panic，或者errGoexit错误，就可以正常向所有chan中写Result了
			// Normal return
			for _, ch := range c.chans {
				ch <- Result{c.val, c.err, c.dups > 0}
			}
		}
	}()

	func() {
		defer func() {
			if !normalReturn {
				// Ideally, we would wait to take a stack trace until we've determined
				// whether this is a panic or a runtime.Goexit.
				//
				// Unfortunately, the only way we can distinguish the two is to see
				// whether the recover stopped the goroutine from terminating, and by
				// the time we know that, the part of the stack trace relevant to the
				// panic has been discarded.
				if r := recover(); r != nil {
					c.err = newPanicError(r)
				}
			}
		}()
		
        // 这里在一个匿名函数里执行fn()，并且顺利的执行了fn，就是正常的，并且前面还用了一个recover()
        // 找到fn()执行过程中的panic
		c.val, c.err = fn()
		normalReturn = true
	}()

	if !normalReturn {
		recovered = true
	}
}
```

#### DoChan()函数

DoChan()和Do()类似，区别就是一个是同步等待，一个是异步返回：

```go
// DoChan is like Do but returns a channel that will receive the
// results when they are ready.
//
// The returned channel will not be closed.
func (g *Group) DoChan(key string, fn func() (interface{}, error)) <-chan Result {
	ch := make(chan Result, 1)
	g.mu.Lock()
	if g.m == nil {
		g.m = make(map[string]*call)
	}
	if c, ok := g.m[key]; ok {
		c.dups++
		c.chans = append(c.chans, ch)
		g.mu.Unlock()
		return ch
	}
	c := &call{chans: []chan<- Result{ch}}
	c.wg.Add(1)
	g.m[key] = c
	g.mu.Unlock()
	
    // 这里是开了个goroutine异步执行call，具体在doCall中都进行判断区分了
	go g.doCall(c, key, fn)

	return ch
}
```

#### Forget()函数

Forget()函数非常简单，就是把key给删了，要注意的就是golang的map是并发不安全的，需要加锁：

```go
// Forget tells the singleflight to forget about a key.  Future calls
// to Do for this key will call the function rather than waiting for
// an earlier call to complete.
func (g *Group) Forget(key string) {
	g.mu.Lock()
	if c, ok := g.m[key]; ok {
        // 先把call的forgotten置为true，通知doCall这个key已经被删了
		c.forgotten = true
	}
	delete(g.m, key)
	g.mu.Unlock()
}
```

# Singleflight使用的一些注意事项

#### 一个阻塞，全员等待

这个问题解决方式就是上述的结合DoChan()和select做超时控制。

#### 一个错误，全部错误

这个问题解决方案就是上述的开一个goroutine超时后执行Forget()，提高下游服务的并发，让错误时可以多试几次，提高成功率。

