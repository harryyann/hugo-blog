---
title: "Golang的slice底层实现"
date: 2021-07-13T20:01:29+08:00
Tags: [golang]
Categories: [golang]
draft: false
---

# Slice的底层数据结构

Golang的slice切片依托于数组实现，和数组的差别就是slice可以方便的扩容，实际使用中除了在少数情况我们明确知道数组长度时，大多数时候都是用的是slice，因为不用担心容量不够的问题。但是如果不了解它内部的实现机制，就有可能会遭遇一些莫名的现象。

**slice底层结构**：

```
// 源码位于src/runtime/slice.go

type slice struct{
    array unsafe.Pointer
    len int
    cap int
}
```

array是一个unsafe.Pointer指针类型，len表示这个切片的长度，cap表示这个切片的容量。

# Slice的创建方式

### 使用make()创建

```go
slice := make([]int, 5, 10) // 创建一个len=5， cap=10的slice，后续添加元素直到10时都不用扩充
```

**make()**函数创建的slice可以指定len和cap，这样就会创建出一块内存，分配给这个slice。



![](/images/slice/make2slice.png)

### 根据一个数组或切片创建

对一个数组或切片进行切片会产生一个新切片。

```go
array := [10]int{}
slice := array[5:7]
```

注意这个**切片将会和原数组公用一部分内存**：

![](/images/slice/array2slice.png)

切片产生的这个slice对象的array指针指向的是原array的5号位置，到7号位置结束，即这个新的slice的len为2；同时**原array中被引用的部分以及剩下的部分**都将成为slice的cap，即cap=5。

根据这个原理，当对一个数组和切片执行切片操作时，也可以顺带指定cap的位置：

```go
sliceA := make([]int, 5, 10) // sliceA的len是5， cap是10
sliceB := sliceA[0:5:5] // 切0-5这部分，同时指定cap是到了原切片5号索引的位置
```

sliceB是从sliceA的头部开始切的，所以它的cap也原本是10，但我们也可以指定它的容量是到索引5的位置，所以它的cap就变成了5。再举一个例子：

```go
sliceB := sliceA[5:7:8] // 切5-7这部分，同时指定cap是到8号索引的位置
```

sliceB是从sliceA的5号索引切到7号索引，并且指定cap的位置是8，所以cap是5-10的5个，现在变成了5-8的3个。

# Slice自动扩容

当我们只用append()函数向一个slice追加元素时，当slice的cap不足时，就会触发自动扩容。

```
newSlice := append(slice, 10) // 向slice中插入一个10，返回一个新slice
```



扩容的过程实际上是重新分配一块更大的内存，然后将原slice的数据拷贝进新的slice中，并且把新append的数据追加到这个slice中。如下图：

![](/images/slice/append.png)

原来的len值已经和cap值相等了，当append一个新元素时，发现cap<len+1，则就触发了扩容。可以看到，扩容后的cap由5变成了10。

### append()的过程

1. 当slice的cap值允许len+1时，就可以将这个元素直接插进入，同时len++，由于slice的位置没有变动，所以返回新slice的array指针和原slice的array指针是一样的；
2. 如果slice的cap不允许len+1，则先将slice执行扩容，开辟一块更大的新空间将原slice的值拷贝进去；
3. 然后将新元素追加到新slice中，重复1的过程。

### 扩容容量的规则

当slice执行扩容时会根据原容量的大小，决定扩容后的容量大小，并不是单纯的翻倍：

- 如果原slice容量小于1024，则新slice容量每次扩容将扩大为原来的两倍
- 如果原slice容量大于等于1024，则新slice容量将扩大为原来的1.25倍

# Slice的拷贝

slice需要通过函数copy()完成，拷贝时会**将原切片的数据依次拷贝到新切片**中。

要注意，如果说新的切片len值和旧切片的len值不相等的时候，copy()也是可以完成的。具体分两种情况：

1. 当新切片的len值小于旧切片len值时：

   将会只从旧切片中拷贝新切片len值数量的元素到新切片，如下：

   ```go
   src := []int{1,2,3,4} // src的len=4
   dst := make([]int, 3) // dst的len=3
   copy(dst, src) //执行拷贝
   fmt.Println(dst) // 打印将会输出[1 2 3]，即只拷贝了三个元素过来
   ```

2. 当新切片的len值大于旧切片的len值时：

   将会把旧切片的元素完全拷贝过来，并且空位补充元素的0值：

   ```go
   src := []int{1,2,3,4} // src的len=4
   dst := make([]int, 3) // dst的len=5
   copy(dst, src) //执行拷贝
   fmt.Println(dst) // 打印将会输出[1 2 3 4 0]，即元素全部拷贝并且还补了0
   ```

出现这种情况归根到底就是copy()函数的工作方式，**将原切片的数据依次拷贝到新切片**。只是单纯的拷贝数据，能拷多少拷多少，对新切片是不会有任何变动的，即copy过程不会发生扩容。

# Slice使用时要注意的点

- 创建切片时可根据需要分配尽量够用的cap，尽量避免由于追加导致的扩容操作，以提高性能；
- 切片拷贝时要注意新切片的len和旧切片的len，否则可能会有元素拷不进去；
- 要小心通过数组或切片创建新切片时，因为公用同一块内存地址，可能会有读写冲突；而当不停的append导致切片扩容，从而新切片被迁移到了另一个地址，那就和旧数组或切片分开了，自然就不会有读写冲突了；
- 使用len()计算切片长度，时间复杂度为O(1)，不需要遍历数组，因为已经定义在了数组的结构体中，使用cap()计算切片容量时，时间复杂度也为O(1)，同理；
- 通过函数参数传递切片时，不会拷贝整个切片，因为切片本身只是个结构体而已，真正的底层array是个指针，传递指针进去，函数里是可以修改是会改变外部的切片的；
- 当对数组、切片、map等复杂对象去判断相等时，需要使用reflect包的DeepEqual()方法，比较复杂数据中的所有成员都相等。