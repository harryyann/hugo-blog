---
title: "k8s的Informer机制"
date: 2021-05-12T23:55:43+08:00
Description: ""
draft: false
Tags: [k8s,golang]
Categories: [k8s]
DisableComments: false
---

## Client-go

client-go与其他语言版本的k8s不同的地方在于，它不仅仅是一个k8s的客户端，而且还是k8s的核心库，k8s中各个组件与api-server之间的通信都是通过client-go的iformer机制实现的快速低延迟的数据同步。kubernetes中的各个组件和api-server之间都是通过http通信，那么需要非常高的可靠性，时效性。核心就是使用了client-go的informer机制。

## Informer机制

下图是client-go中Informer机制的架构图：

![Image](/images/informer.png)

最外层是一个Informer，可以看到所有功能都被封装到了Informer中，通过Informer与kube-apiserver通信，实现对API资源的list/watch（所谓ListAndWatch就是通过List获取到所有最新版本的API对象，然后再通过Watch机制监听这些API对象的变化），Informer也可以连接用户定义的eventHandler对捕获到的某些事件进行处理，还可以将watch到的更新，输出到外部，比如存到本地数据库等等。总体而言，所谓的 Informer，就是一个自带缓存和索引机制，可以触发 Handler 的客户端库。

### Informer的三个组成部分

#### Reflector

如上图，Reflector包含一个通过clientset实现的ListerWatcher对象和一个Store对象，Reflector会通过ListerWatcher方法对api-server进行watch，通过比对resourceVersion来捕获API资源的变化，然后送入store。store是一个FIFO Queue(先入先出队列)。

FIFO Queue的实现者为DeltaFIFO，store是一个内存级的缓存，存储捕获到的变更，对Store提供了pop()方法，就实现了一个Queue。

#### Controller

ontroller将会从Reflector的FIFOQueue中逐条获取到变更事件，然后他可以根据事件的类型（增加，更新还是删除），触发onAdd、onUpdate、onDelete方法，对应的方法最终会触发用户自己定义的eventHandler；Controller也会将变更更新到Informer的Store中（注意，这个Store和Reflector中的store的区别），保持Store数据和kube-apiserver的一致。

#### Store

维护着和apiserver中相同的数据，这里面的数据不是变更数据，而是list的数据，外部可以从Indexer中Get/List到数据。这样如果用户需要获取数据，就可以直接从Store中拿，而不是访问kube-apiserver。通过Store就实现了缓存级别的List/Get我们想要的API对象的方式。

## Informer的使用方式

我们以监控pod的informer为例：

#### 获取clientset

```go
config ,err := clientcmd.BuildConfigFromFlags("","config")
if err != nil {
    panic(err)
}
// 通过kubeconfig 拿到clientset
clientset ,err := kubernetes.NewForConfig(config)
if err != nil {
    panic(err)
}
```

#### 通过clientset获取informer

```go
stopch := make(chan struct{})
defer close(stopch)
 //通过clientset获取informer的集合（参数rsync也就是同步间隔时间 如果是0那么就禁用同步功能，这里设置1m）
sharedInformers := informers.NewSharedInformerFactory(clientset,time.Minute)
```

#### 获取指定资源的Informer

```go
//通过sharedinfomers获取到pod的informer
podInfomer := sharedinformers.Core().V1().Pods().Informer()
```

#### 为podInformer添加eventHandler

```go
//为pod informer添加 controller的handlerfunc  触发回调函数之后 会通过addch 传给nextCh 管道然后调用controller的对应的handler来做处理
podinfomer.AddEventHandler(cache.ResourceEventHandlerFuncs{
	//pod资源对象创建的时候出发的回调方法,新增pod时回调
    AddFunc: func(obj interface{}) {
        obja := obj.(v1.Object)
        fmt.Println(obja)
    },
    //更新回调
    UpdateFunc: func(oldObj, newObj interface{}) {
        ...
    },
    //删除回调
    DeleteFunc: func(obj interface{}) {
        ...
    },
})
```

#### 最后启动Informer

```go
// 定义一个stop channel用于停止Informer
stopch := make(chan struct{})
defer close(stopch)
podInfomer.Run(stopch)
```

这样当informer启动后，将会监控集群内的pod事件，当出现add、update、delete事件时就可以通过我们编写的eventHandler进行处理了。



最后，不仅仅k8s原生的API对象可以使用Informer，我们自己创建的CRD(Custom Resource Defined)也可以使用Informer，通过CRD我们就可以创建我们自己的API对象，然后通过Informer机制来监听集群中CRD对象的变化，进而实现一些控制，这就是自定义控制器。而通过自定义控制器去管理有状态应用，就是k8s的**Operator**模式。