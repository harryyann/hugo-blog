---
title: "Golang的指针和内存对齐"
date: 2021-08-20T19:34:46+08:00
Description: ""
Tags: [golang]
Categories: [golang]
DisableComments: false
draft: false
---

# 指针

## golang中的指针种类

golang的指针类型分为三种，***类型**，**unsafe.Pointer**，**uintptr**。

- *类型：普通指针类型，用于传递对象地址，不能进行指针运算。
- unsafe.Pointer：通用指针类型，用于**转换不同类型的指针**，不能进行指针运算，不能读取内存存储的值，必须要转换到某一类型的普通指针。
- uintptr：用于指针运算，GC不把uintptr当指针，uintptr无法持有对象，uintptr类型的目标会被回收。

通过以上就可以看出：

- unsafe.Pointer是桥梁，可以让任意类型的普通指针进行互相转换，也可以将任意类型的指针转换成uintptr来进行指针运算。
- unsafe.Pointer可以让你的变量在不同的指针类型转来转去，也就是表示为任意可寻址的指针类型，而uintptr常用于和unsafe.Pointer打配合，用于指针运算。

## unsafe.Pointer

四个重要特性：

- 任何类型的指针都可以转化为Pointer
- Pointer可以转化为任意类型的指针
- uintptr可以转化为Pointer
- Pointer可以转化为uintptr

注意：

1. 不可以直接通过*p来获取unsafe.Pointer指针指向的真实变量的值，因为不知道p的具体类型。
2. Pointer指针只能和nil比较判断是否为空指针

## uintptr

uintptr其实就是一个整数类型，只是用于表示指针罢了。

有一点要注意，uintptr变量即使仍然有效，依然会被GC回收，这也是将其不推荐使用unitptr类型的原因。



## unsafe包

unsafe包只有两个struct、三个函数。

struct：

- ArbitraryType：是int的别名，含义是代表一个任意的类型

  ```go
  type ArbitraryType int
  ```

- Pointer：是int指针类型的别名，表示任意指针的父类型

  ```go
  type Pointer *int
  ```

函数：

- unsafe.Sizeof()：接收任意类型值，返回其占用的字节数，和C不同的是，传入的可以是任意对象表达式，结构体，变量都可以。
- unsafe.Offsetof()：返回结构体中元素所在内存的偏移量，可以接收任意类型的变量，但是需要变量是struct类型，且只能把struct的属性当作参数。
- unsafe.Alignof返回变量的对齐字节数量，即最小对齐块的字节数。



### 常用使用方式

#### unsafe.Pointer进行普通指针类型转换

```
v := int(13)
p := unsafe.Pointer(&v) // 接收一个指针，返回一个Pointer对象
uv := (*uint)(p)  //将p强制转换为uint类型的指针
fmt.Println(reflect.TypeOf(uv))  // *uint
fmt.Println(*uv)  // 13
```

#### unsafe.Pointer用于操作结构体的私有变量

```go
package a

type V struct {
    i int32
    j int64
}

func (this V) PutI() {
    fmt.Printf("i=%d\n", this.i)
}

func (this V) PutJ() {
    fmt.Printf("j=%d\n", this.j)
}
```

包a的V这个struct只有两个方法，用于打印出两个私有成员变量的值，不能给他们赋值：

```go
packagepackage main

func main(){
    var v := a.V{}
    // 拿到v结构体的起始指针，要用第一个元素的指针类型进行转化，这个指针指向的就是v结构体的第一个元素
    var i *int32 = (*int32)(unsafe.Pointer(v))
    *i = int32(98) // 给i赋值
    v.PutI()  // 打印可以看到i已经变为98了

    // 然后获取j的内存地址，需要根据内存对齐算一下，要注意内存对齐后，i占用的字节数也是和j一样的int64的长度了
    var j *int64 = (*int64)(unsafe.Pointer(uintptr(unsafe.Pointer(&v)) + uintptr(unsafe.Sizeof(int64(0)))))
    *j = int64(763)
    v.PutJ() // 打印可以看到j已经变为763了
}
```

核心思想就是：结构体的成员在内存中的分配是连续的内存，采用内存对齐的原则，这样就可以根据成员类型推算出各个成员所在的内存地址了。

一定要注意的是：不要把uintptr类型的数据作为一个临时变量，要把涉及到了uintptr的表达式都写在一行里，因为uintptr类型变量会被GC回收！

```go
// 错误写法!!!
tmp := uintptr(unsafe.Pointer(&v) + uintptr(unsafe.Sizeof(int64(0)))
j := (*int64)(unsafe.Pointer(tmp)
```

# 内存对齐

## 内存对齐的原因

- 并不是所有硬件平台都能访问任意地址上的任意数据。
- 性能，访问未对齐的内存，处理器需要做两次内存访问，而对齐的内存只需要访问一次。

## 内存对齐的方式

```go
fmt.Println(unsafe.Sizeof(int64(0))) // "8"
type SizeOfA struct {
    A int
}
unsafe.Sizeof(SizeOfA{0}) // 8
type SizeOfC struct {
    A byte  // 1字节
    C int32 // 4字节
}
unsafe.Sizeof(SizeOfC{0, 0})    // 8
unsafe.Alignof(SizeOfC{0, 0})   // 4
```

以上的结构体SizeOfC，实际只需要占用5个字节，但是实际上占了8字节。因为这个结构体的对齐字节倍数Alignof(SizeOfC)=4，即结构体占用的实际大小必须是4的倍数。

Align返回的对齐数是结构体中单位基本类型所占的内存字节数，不超过8，如果元素是数组则取数据元素所占内存值而不是整个数组的内存值：

```go
type SizeOfD struct {
    A byte
    B [3]int32
}
unsafe.Sizeof(SizeOfD{})   // 16
unsafe.Alignof(SizeOfD{})  // 4，因为结构体最大元素为int32，占四个字节
```

因为结构体元素在内存中的布局是按顺序放的，先放了A，然后发现后面的7个字节放不下B，则会再后分配8个字节给B，那么A自己就独占了8个字节。以上结构体在内存中的布局：

![](/images/pointer/struct1.png)

再举个例子：

```go
type SizeOfE struct {
    A byte  // 1
    B int64 // 8
    C byte  // 1
}
unsafe.Sizeof(SizeOfE{})    // 24
unsafe.Alignof(SizeOfE{})   // 8
```

以上结构体在内存中的布局：

![](/images/pointer/struct2.png)

而如果把C写在B之前，那么按顺序放C的时候发现C和A占用的内存刚好一样，就可以把C直接放在A后面了，然后发现下一个元素B占8个字节，那么为了内存对齐，。这样这个结构体占内存就比上面小了：

```go
type SizeOfF struct {
    A byte  // 1
    C byte  // 1
    B int64 // 8   
}
unsafe.Sizeof(SizeOfF{})    // 16
unsafe.Alignof(SizeOfF{})   // 8
```

以上结构体在内存中的布局：

![](/images/pointer/struct3.png)

后面的元素在放时会首先根据前一个元素所占的内存大小来决定这个元素要怎么和前一个元素拥有相同的内存数量，反正最大不超过8字节：

```go
type SizeOfH struct {
    A byte
    C int16
    B int64
    D int32
}
unsafe.Offsetof(SizeOfH{}.A) // 0
unsafe.Offsetof(SizeOfH{}.C) // 2
unsafe.Offsetof(SizeOfH{}.B) // 8
unsafe.Offsetof(SizeOfH{}.D) // 16
```

以上结构体在内存中的布局：

![](/images/pointer/struct4.png)

## golang中常用类型的内存大小

- bool：1字节
- int、uint、uintptr、*T、map、func、chan：1机器字节（根据机器是32位还是64位，分别占4个和8个字节）
- string：2机器字节
- interface：2机器字节
- []T：3机器字节
- intN、uintN、floatN、complexN：N/8个字节