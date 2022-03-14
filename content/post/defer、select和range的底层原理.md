---
title: "Golang中的defer、select和range的底层原理"
date: 2021-08-14T19:34:46+08:00
Description: ""
Tags: [golang]
Categories: [golang]
DisableComments: false
draft: false
---

# Defer

defer语句用于延迟函数的调用，每次defer都会把一个函数压入栈中，当前函数返回前，再把延迟的函数取出来并执行，所以多个defer语句是**后写的先执行**的。

## 几个题目

### 问题1：以下输出什么？

```go
func deferFuncParameter() {
    var aInt = 1

    defer fmt.Println(aInt)

    aInt = 2
    return
}
```

答：输出1，因为defer调用的时候，传进去的参数就确定是1，并且压入栈中，defer语句在调用时后续执行的参数就确定了，所以后续调用的时候参数还是1。

### 问题2：以下输出什么？

```go
func printArray(array *[3]int) {
    for i := range array {
        fmt.Println(array[i])
    }
}
func deferFuncParameter() {
    var aArray = [3]int{1, 2, 3}

    defer printArray(&aArray)

    aArray[0] = 10
    return
}

func main() {
    deferFuncParameter()
}
```

答：输出[10, 2, 3]三个值，因为defer传入的是指针，虽然指针确定了，但最后指针指向的数组却发生更改了。

### 问题3：以下返回什么？

```go
func deferFuncReturn() (result int) {
    i := 1

    defer func() {
        result++
    }()

    return i
}
```

答：defer让具名返回值result增加1，而return语句并不是原子的，实际上分为设置返回值->ret，defer语句执行在返回前，所以过程为：**设置返回值 ->执行defer -> ret**，所以在真正返回前，result已经被赋值为1了，defer语句自增后，result=2。

## Defer的规则

- defer执行的延迟函数的参数在defer语句出现时就已经确定(题目1)。
- 延迟函数执行按照后进先出的顺序执行，是一个栈。
- 延迟函数可以操作主函数中的具名返回值(题目3)，因为defer发生在确定返回值和ret之间。

## 函数返回的过程

return不是一个原子操作，return只代理汇编指令ret，即将跳转程序执行。比如return i，实际上分两步：

1. 将i存入栈作为返回值
2. 然后执行跳转

而defer的执行时机正是跳转前和返回值和压栈后，所以说defer执行时还是可以操作返回值，前提是这个返回值得有名字，要不也操作不了。

总体而言，defer语句操作返回值有两种情况：

1. 主函数拥有匿名返回值，返回一个局部变量时，此时defer语句可以引用到返回值，但不会改变返回值

   ```go
   func foo() int {
       var i int
   
       defer func() {
       i++
       }()
   
       return i
   }
   ```

   上面的函数，返回一个局部变量，返回之前defer也会操作这个变量，对于匿名的返回值来说，可以假定仍有一个变量保存返回值，所以以上代码可以拆分成以下过程：

   ```go
   annoy = i  // 固定返回值，发生在defer之前
   i ++       // defer语句发生在固定返回值和最终返回之间，defe修改的也只是i，而不能修改annoy这个变量
   return annoy
   ```

2. 当主函数拥有具名返回值时，就可以被defer修改了

   ```go
   func foo() (ret int) {
       defer func() {
           ret++
       }()
   
       return 0
   }
   ```

   以上代码可以拆解为：

   ```go
   ret = 0  // 固定返回值
   ret ++   // 因为返回值有名字，所以最终返回前可以操作这个变量
   return ret
   ```

## Defer的实现原理

### defer的数据结构

```
type _defer struct {
    sp uintptr //函数栈指针
    pc uintptr //程序计数器
    fn *funcval //函数地址
    link *_defer //指向自身结构的指针，用于链接多个defer
}
```

defer的数据结构和一般函数类似，也有栈地址，程序计数器，函数地址等。与函数不同的是它含有一个指针，可以指向另一个defer。每次声明一个defer时，就将defer插入到单链表表头，执行时则按顺序执行。

源码包`runtime/panic.go`定义了两个方法分别用于创建defer和执行defer：

- deferproc()：在声明defer处调用，其将defer函数存入goroutine的链表中。
- deferreturn()：在return指令，准确讲是在ret指令前调用，将defer从goroutine链表中取出并执行。



# Select

select是golang语言层面提供的IO多路复用的机制，可以同时检测多个channel是否ready。



## 几个题目

### 题目1：以下程序输出什么

```go
chan1 := make(chan int)
chan2 := make(chan int)

go func(){
    chan1 <- 1
}

go func(){
    chan2 <- 1
}

select {
case <-chan1:
    fmt.Println("chan1")
case <- chan2:
    fmt.Println("chan2")
default:
    fmt.Println("default")
}
```

答：三种都有可能。因为select中各个case的执行顺序是随机的。

### 题目2：以下程序输出什么

```go
chan1 := make(chan int)
chan2 := make(chan int)

go func(){
    close(chan1)
}

go func(){
    close(chan2)
}

select {
case <-chan1:
    fmt.Println("chan1")
case <- chan2:
    fmt.Println("chan2")
default:
    fmt.Println("default")
}
```

答：close()不会导致chan不可读，所以select依然会检测各个case，输出什么依然是随机的。

### 题目3：以下程序会发生什么？

```go
func main(){
    select{
    }
}
```

答：空select会阻塞，准确说是当前协程被阻塞，同时golang自带死锁检测机制，当发现协程阻塞且再也没有机会唤醒时，会panic。

## select的数据结构

golang实现select时，定义了一个数据结构表示每个case语句(包括default)，select执行过程可以类比为一个函数，函数输入case数组，输出选中的case，然后程序流转到这个case块。

```go
type scase struct{
    c *hchan 
    kind uint16
    elem unsafe.Pointer
}
```

- c是当前case语句的channel指针，一个case只能操作一个channel。
- kind表示该case的类型，分为读channel、写channel和default，又三个常量定义：
  - caseRecv：case语句中尝试读取scase.c中的数据
  - caseSend：case语句尝试向scase.c中写数据
  - caseDefault：default语句
- elem表示缓冲区地址，根据scase.kind不同，有不同的用途：
  - scase.kind == caseRecv，elem表示要读出channel的数据存放地址。
  - scase.kind == caseSend，elem表示要写入channel的数据存放地址。

## select的实现逻辑

源码包`runtime/select.go`中定义了select选择case的函数：

```
func selectgo(cas0 *scase, order0 *uint16, ncases int) (int, bool)
```

参数说明：

- cas0为scase数组的首地址，selectgo()就是从这些scase中找出一个返回
- order0为一个两倍cas0数组长度的buffer，保存scase随机序列pollorder和scase中channel地址序列lockorder：
  - pollorder：每次selectgo执行都会把scase序列打乱，以实现随机检测case的目的。
  - lockorder：所有case语句中channel序列，以达到去重防止对channel加锁时重复加锁的目的。
- ncases表示scase数组的长度



返回值说明：

- int：选中case的编号，这个case编号和代码一致。
- bool：是否成功从chennel中读取了数据，如果选中的case是从channel中读数据，则该返回值表示是否读取成功。

## 一个select的执行过程

1. 锁定scase语句中所有的channel
2. 按照随机顺序检测scase中的channel是否ready
   1. 如果case可读，则读取channel中数据，解锁所有的channel，然后返回(case index, true)
   2. 如果case可写，则将数据写入channel，解锁所有的channel，然后返回(case index, false)
   3. 所有case都未ready，则解锁所有的channel，然后返回（default index, false）
3. 所有case都未ready，且没有default语句
   1. 将当前协程加入到所有channel的等待队列
   2. 当将协程转入阻塞，等待被唤醒
4. 唤醒后返回channel对应的case index
   1. 唤醒后返回channel对应的case index
   2. 如果是写操作，解锁所有的channel，然后返回(case index, false)

## 总结

- select语句中除了default外，每个case操作一个channel，要么读要么写。
- select语句除了default，各case执行顺序是随机的。
- select语句中如果没有default语句，则会阻塞等待任意一个case。
- 对于select语句中读操作，要判断是否成功读取，因为关闭的channel也可以读取，此时ok=false，可能会读到错误的信息。


# Range

range是一种迭代遍历手段，可操作的类型有数组，切片，Map，channel等。

## 几个题目

### 题目1：切片遍历，请问以下程序性能上有没有优化的空间？

```go
func RangeSlice(slice []int) {
    for index, value := range slice {
     _, _ = index, value
    }
}
```

答：遍历过程中，每次的遍历都会给index和value赋值，而实际上给value赋值的步骤是多余的，因为可以直接通过slice[index]直接访问到value。所以可以用_忽略value的值，用slice[index]来访问元素。

### 题目2：map遍历打印key和value，有没有优化的空间？

```go
func RangeMap(myMap map[int]string){
    for key, _ := range myMap{
        _, _ = key, myMap[key]
    }
}
```

答：遍历时只获取到了key，忽略了value，虽然少了一次赋值，但实际上多了一步通过key查找value的步骤，而查找的性能消耗可能高于赋值的性能消耗(map的查找和slice的查找不同的，后者直接就能找到，前者则需要运算)，所以这里能否优化需要取决于map存储数据结构特征。

### 题目3：动态遍历，以下程序能否正常结束？

```go
func main(){
    v := []int{1, 2, 3}
    for i := range v{
        v = append(v, i)
    }
}
```

答：可以在遍历三次后结束。因为循环次数在循环开始时就确定了，**后续切片变长后并不会改变循环次数**。

## range实现原理

range是个C风格的循环结构，range支持数组，数组指针，切片，map和channel类型，对于不同类型的实现有细节差异。

### 遍历切片的过程

1. 先获取到切片的长度作为循环次数（正因如此，循环中如果切片长度发生改变，新添加的元素是没办法遍历到的。但是修改的后面元素却是可以遍历到的）。
2. 循环中，每次循环会先获取元素的索引和值作为一个匿名的变量。
3. 如果range语句中对两个值都有接收的话，则会对index和value进行一次**赋值**。

### 遍历map过程

1. 遍历map时没有指定循环次数。循环体与slice类似。
2. 由于map底层使用hash表，如果遍历时插入数据，则插入数据的位置是随机的。
3. 如果遍历时新插入的数据在当前遍历位置的后面，则可以被后续遍历到，如果在前面则遍历不到。无法保证新键值对会被遍历到还是不会被遍历到。

### 遍历channel过程

1. channel在被遍历时是不知道里面有多少个元素的。
2. 循环时会按顺序从channel中取数据出来。
3. 如果channel中没有元素，则会阻塞等待。
4. 如果channel被关闭，则会解除阻塞并退出循环。



## Range使用的一些技巧

- 使用range遍历切片时，可以适当的放弃接收index和value，减少数据的拷贝可以提高一定性能。
- 遍历channel的表现很不同，需要特别注意。
- 尽量不要在遍历时修改原数据，因为可能会产生不好确定的效果。需要这样的场景，最好重新开辟一块内存暂存需要修改的数据。



