---
title: "使用Kubebuilder编写Operator"
date: 2021-09-20T17:39:32+08:00
Description: ""
Tags: [k8s, operator]
Categories: [k8s]
DisableComments: false
draft: false
---

# CRD和自定义控制器

在[k8s的Informer机制](https://yanghairui.life/post/k8s%E6%A0%B8%E5%BF%83%E5%BA%93client-go%E7%9A%84informer%E6%9C%BA%E5%88%B6/)这篇文章中我们了解了K8s的Informer机制，以及对于k8s原生资源的Informer对象的使用方式。Informer机制归根到底就是利用了kube-apiserver的ListAndWatch机制，去监听集群中特定的API对象的变更，进而去实现一些控制。而且k8s的API对象是支持拓展的，允许用户在 Kubernetes 中添加一个跟 Pod、Node 类似的、新的 API 资源类型，然后编写一个针对自定义API对象的控制器，通过Informer机制去监听这个新的API对象在集群中的变更，然后实现一些控制。这中自定义的API对象就是**CRD(Custom Resource Definition)**。

K8s中创建一个CRD十分简单，只需要创建一个CustomResourceDefinition类型的对象即可，下面是一个例子：

```yaml
# crd.yaml
---
apiVersion: apiextensions.k8s.io/v1beta1
kind: CustomResourceDefinition
metadata: 
  name: networks.samplecrd.k8s.io
spec: 
  group: samplecrd.k8s.io 
  version: v1 
  names: 
    kind: Network 
    plural: networks 
  scope: Namespaced
```

关于CRD的创建这里不多赘述，关于细节可以查看k8s的[官方文档](https://kubernetes.io/docs/tasks/extend-kubernetes/custom-resources/custom-resource-definitions/)。

创建好CRD之后，我们就可以创建这种CRD类型的API对象了，但是这时我们创建的CRD只能说是一种存储在etcd里面的一些数据，除了能看一点用处都没有，要通过这种新API对象实现一些功能，就需要编写一个控制器。k8s集群里的原生API对象，如Pod、Deployment等都有自己的控制器，这些控制器都被集成在了kube-controller-manager这个控制面组件上。我们要编写的自定义控制器，可以以Pod的形式运行在集群上，然后在pod中和kube-apiserver进行ListAndWatch自定义的API对象，实现一些控制逻辑。

一个自定义控制器的实现方式可以如下图所示：

![](/images/operator/controller.webp)

一个自定义控制器包含三块部分：

- Informer

  用于从kube-apiserver中获取它关心的API对象，即我们通过CRD定义的新的API对象。

- WorkQueue

  用来缓存API对象，ListAndWatch这个机制的主要好处就是它不是在需要的时候才去访问kube-apiserver，而是实时的保持本地缓存和kube-apiserver中的数据一致性，这样我们需要的时候可以直接从缓存中快速的获取

- Control Loop

  上面的Informer和WorkQueue我们在[k8s的Informer机制](https://yanghairui.life/post/k8s%E6%A0%B8%E5%BF%83%E5%BA%93client-go%E7%9A%84informer%E6%9C%BA%E5%88%B6/)中已经讲过。Control Loop即控制循环，才是一个自定义控制器的逻辑核心。在这里我们会实现控制逻辑。从名字中就可以很好看出，Controll Loop是一个无限的循环，在我们通过Informer拿到一个API对象，这个API对象的spec字段中，会填写这种资源的期望状态，Contrl Loop通过在每一个循环周期中，对比期望状态和集群中的实际状态的差异，不断的将实际状态和期望状态靠拢，这个过程就叫做**调谐**（Reconcile）。

# Operator

以上我们了解了一个自定义控制器的使用。而通过CRD和自定义控制器的设计，实现一种对应用的特定编排方式，就是Operator。一个Operator中可以包含一个或多个的CRD以及它们的自定义控制器，通过相互协调进而实现一些特定的应用编排方式。

在这个网址[operatorhub.io](https://operatorhub.io/)有很多个别人或者公司写好的Operator，可以直接使用。比如可以在集群中部署一个etcd集群的operator，部署一个redis集群的operator等。结合Statefulset部署有状态应用是Operator最擅长实现的功能。

目前有很多工具可以帮我们自动生成Operator的代码，比如k8s官方的kubebuilder，redhat的Operator SDK等，可以帮助我们快速生成一个Operator的框架和yaml文件，Informer、Control Loop都帮我们实现完了，我们只需要编写Reconcile函数即可，实现自己的控制器逻辑即可。

# 通过kubebuilder编写一个Operator

## 设计Operator

这里我们设计一个CRD叫做MyPod，通过创建一个MyPod对象，Operator会帮我们在集群中创建一个pod出来（虽然有点脱裤子放屁，这里只是通过这个例子帮助理解哈╮(๑•́ ₃•̀๑)╭）。

我们的API对象大概长这样：

```yaml
apiVersion: harryyann.github.io
kind: MyPod
metadata:
  annotations:
  labels:
  name: 
  namespace:
spec:
    podAnnotations:
    podLabels:
    podSpec:
        # 这里是原封不动的podSpec
status:
    podPhase:
    podIp:
    nodeIp:
    createdTime:
```

metadata里面填的是MyPod对象的基础信息，spec里是期望状态，就是pod的annotation、label和podSpec。status保存和MyPod对象关联的pod的一些信息，如果phase、podIp、nodeIp等。

要实现这个需求，我们要做的事情有三个：

- 创建一个MyPod的CRD出来。
- 编写一个Operator监听集群中MyPod对象的创建，每当创建一个MyPod对象，我们就通过clien创建一个pod出来。
- operator还要监控创建的pod，并把这个pod的一些信息，实时更新到MyPod的status中，且在删除MyPod时，对应的pod也要删除

## 安装kubebuilder

在[这个地址](https://github.com/kubernetes-sigs/kubebuilder/releases)下载好需要的二进制文件，把kubebuilder放到PATH目录下，检查一下kubebuilder是否可用：

```bash
# kubebuilder version
Version: main.version{KubeBuilderVersion:"3.3.0", KubernetesVendor:"1.23.1", GitCommit:"47859bf2ebf96a64db69a2f7074ffdec7f15c1ec", BuildDate:"2022-01-18T17:03:29Z", GoOs:"linux", GoArch:"amd64"}
```

可以正确显示版本就说明安装成功了。

## 初始化项目

1. 首先创建一个空目录，然后进入这个目录：

   ```bash
   # mkdir mypod-operator
   # cd mypod-operator
   ```

2. 执行kubebuilder init命令初始化项目：

   ```bash
   # kubebuilder init --domain harryyann.github.io --repo github.com/harryyann/mypod-operator
   ```

   我这里指定了`--domain`和`--repo`两个参数，domain是我们项目的域名，他会成为我们创建的API对象的组名，repo则是仓库地址，他会成为`go mod init`时指定的项目名。

3. 然后我们创建API：

   ```bash
   # kubebuilder create api --namespaced --version v1 --kind MyPod --plural mypods
   ```

   这里我们会创建API对象的代码，`--namespaced`参数指定这个API对象是命名空间级的，--version指定了API版本，`--kind`指定了API对象的类型，即yaml文件的kind字段， `--plural`指定了API对象的复数形式。这里还可以指定`--group`组信息，如果不指定会使用`--domain`作为组名，指定了就会和domain进行拼接作为组名。

在执行完成以上命令后，我们当前的目录结构就是这样的：

```
├── Dockerfile
├── Makefile
├── PROJECT
├── api/
├── bin/
├── config
│   ├── crd/
│   ├── default/
│   ├── manager/
│   ├── prometheus/
│   ├── rbac/
│   └── samples/
├── controllers/
├── go.mod
├── go.sum
├── hack/
└── main.go
```

里面一些主要目录的作用如下：

- Makefile：定义了很多脚本，需要什么就执行make的具体步骤即可
- config：一些yaml文件，比如rbac目录下就是一些权限认证的yaml，manager里面就是部署这个operator的yaml，crd里面就是crd的yaml，包括webhook等
- bin：是我们生成代码要用到的二进制文件
- api：比较关键，我们的CRD资源的定义和register、deepcopy等方法也在这。
- controllers：比较关键，我们的控制器的reconcile逻辑就在这里写
- main.go：项目的入口，就是operator的主体逻辑，我们一般情况下不需要改这块



## 修改API对象的字段

生成了整个框架之后，定义一下我们的自定义资源的字段，位于api/v1/mypod_types.go中，在这里定义一下spec字段，和status字段。MyPodSpec中默认会带有一个Foo的string类型字段，把他删掉。然后添加我们真正需要的字段，如下：

```go
// MyPodSpec defines the desired state of MyPod
type MyPodSpec struct {
   // INSERT ADDITIONAL SPEC FIELDS - desired state of cluster
   // Important: Run "make" to regenerate code after modifying this file

   PodAnnotations map[string]string `json:"podAnnotations,omitempty"`
   PodLabels map[string]string `json:"podLabels,omitempty"`
   PodSpec  v1.PodSpec   `json:"podSpec"`
}


// MyPodStatus defines the observed state of MyPod
type MyPodStatus struct {
   // INSERT ADDITIONAL STATUS FIELD - define observed state of cluster
   // Important: Run "make" to regenerate code after modifying this file
   PodPhase string `json:"podPhase,omitempty"`
   PodIp string `json:"podIp,omitEmpty"`
   NodeIp string `json:"nodeIp,omitEmpty"`
   CreatedTimestamp int64 `json:"createdTimestamp,omitEmpty"`
}
```

我们还可以添加这样的注释，让我们可以在通过kubectl查看时展示更多内容：

```go
//+kubebuilder:printcolumn:JSONPath=".status.phase",name=Phase,type=string
//+kubebuilder:printcolumn:JSONPath=".status.podIp",name=PodIp,type=string

//+kubebuilder:object:root=true
//+kubebuilder:subresource:status
```

改完字段后，我们用命令`make manifests generate`，针对修改过的资源，生成需要的yaml文件

```go
# `make manifests generate`
```

然后我们会发现config/crd目录下多了一个bases目录，里面有CRD的定义，我们可以用这个文件创建出CRD。



## 在集群中运行一下operator

我们先不添加Reconcile的代码在集群中跑一下，我跑的过程遇到了一些坑，具体就不说了，总之我做了下面这些修改后，才顺利跑起来：

1. 首先要修改main.go的代码，修改kubeconfig的获取方式，因为我们的operator是在pod里跑的，所以改为InClusterConfig的方式获取kubeconfig，否则pod运行时会因为无法获取kubeconfig而error。

   ```go
   config, err := rest.InClusterConfig()
   if err != nil {
      setupLog.Error(err, "get in cluster config error")
      os.Exit(1)
   }
   mgr, err := ctrl.NewManager(config, ctrl.Options{
   ```

2. 然后还要在manager/manager.yaml中做下修改：添加这一行，要不然InClusterConfig函数会提示找不到token文件：

   ```yaml
   automountServiceAccountToken: true
   ```

3. 最后还要在config/rbac/role.yaml中加些权限：

   ```yaml
   - apiGroups:
       - "coordination.k8s.io"
     resources:
       - leases
     verbs:
       - create
       - delete
       - get
       - list
       - patch
       - update
       - watch
   - apiGroups:
       - ""
     resources:
       - pods
       - configmaps
       - events
     verbs:
       - create
       - delete
       - get
       - list
       - patch
       - update
       - watch
   ```

完成上述操作后，我们在集群中创建出CRD、role、roleBinding、serviceAccount：

```bash
# kubectl apply -f config/crd/bases/harryyann.github.io_mypods.yaml
# kubectl apply -f config/rbac/role.yaml
# kubectl apply -f config/rbac/role_binding.yaml
# kubectl apply -f config/rbac/serviceAccount.yaml
```

然后我们通过make构建一下Operator的镜像，并push到镜像中心：

```bash
# make docker-build
# make docker-push
```

镜像打好并push到远程仓库之后，我们在集群中创建namespace和operator的deployment，我这里给ns起名为test-mypod：

```bash
# kubectl apply -f config/manager/manager.yaml
```

然后就可以看到ns下有pod再跑了：

```bash
# kubectl get pod -n test-mypod
NAME                             READY   STATUS    RESTARTS   AGE
mypod-manager-56d6687f75-pqt6v   1/1     Running   0          18m
```

查看一下它的日志：

```bash
# kubectl logs -f -n test-mypod mypod-manager-56d6687f75-pqt6v
1.6475719323468134e+09  INFO    controller-runtime.metrics      Metrics server is starting to listen    {"addr": ":8080"}
1.6475719323471029e+09  INFO    setup   starting manager
1.6475719323472993e+09  INFO    Starting server {"path": "/metrics", "kind": "metrics", "addr": "[::]:8080"}
1.6475719323473573e+09  INFO    Starting server {"kind": "health probe", "addr": "[::]:8081"}
I0318 02:52:12.347376       1 leaderelection.go:248] attempting to acquire leader lease test-mypod/9ef2ee97.harryyann.github.io...
I0318 02:52:29.431037       1 leaderelection.go:258] successfully acquired lease test-mypod/9ef2ee97.harryyann.github.io
1.647571949431093e+09   DEBUG   events  Normal  {"object": {"kind":"ConfigMap","namespace":"test-mypod","name":"9ef2ee97.harryyann.github.io","uid":"65e4bba0-1dde-4dae-b592-6279d1953b9e","apiVersion":"v1","resourceVersion":"313330506"}, "reason": "LeaderElection", "message": "mypod-manager-56d6687f75-pqt6v_a7556a76-14a3-4c90-8fef-f74bb7144b5d became leader"}
1.6475719494311852e+09  DEBUG   events  Normal  {"object": {"kind":"Lease","namespace":"test-mypod","name":"9ef2ee97.harryyann.github.io","uid":"b3abbcb8-5d68-450c-b104-4c8a9d542e44","apiVersion":"coordination.k8s.io/v1","resourceVersion":"313330507"}, "reason": "LeaderElection", "message": "mypod-manager-56d6687f75-pqt6v_a7556a76-14a3-4c90-8fef-f74bb7144b5d became leader"}
1.647571949431183e+09   INFO    controller.mypod        Starting EventSource    {"reconciler group": "harryyann.github.io", "reconciler kind": "MyPod", "source": "kind source: *v1.MyPod"}
1.647571949431227e+09   INFO    controller.mypod        Starting Controller     {"reconciler group": "harryyann.github.io", "reconciler kind": "MyPod"}
1.6475719495316322e+09  INFO    controller.mypod        Starting workers        {"reconciler group": "harryyann.github.io", "reconciler kind": "MyPod", "worker count": 1}
```

看起来输出正常，我们的Operator可以运行了，接下来就是修改Reconcile的代码，让它实现该有的功能即可。



## 编写Reconcile函数

Reconcile函数位于`controllers`目录下。它的定义是这样的：

```go
func (r *MyPodReconciler) Reconcile(ctx context.Context, req ctrl.Request) (ctrl.Result, error)
```

参数req中会包含当前所变更对象的namespace和name信息，通过这两个值，就可以获取到我们创建、修改、删除的MyPod对象，进而完成一些操作。

这部分主要是写逻辑，我就不细说了，代码我放在我们[仓库](https://github.com/harryyann/mypod-operator)里了。



## 在集群上调试

改完代码后，执行`make docker-build`构建镜像，执行`make docker-push`推送镜像。

然后在集群中通过上设置deployment的`imagePullPolicy=Always`，然后删除manager的pod，他就会自动重建，并拉取最新的镜像。

观察到日志输出是正常的：

```verilog
1.6476801020824914e+09  INFO    controller.mypod        Starting EventSource    {"reconciler group": "harryyann.github.io", "reconciler kind": "MyPod", "source": "kind source: *v1.MyPod"}
1.6476801020826638e+09  INFO    controller.mypod        Starting Controller     {"reconciler group": "harryyann.github.io", "reconciler kind": "MyPod"}
1.647680102183413e+09   INFO    controller.mypod        Starting workers        {"reconciler group": "harryyann.github.io", "reconciler kind": "MyPod", "worker count": 1}
```

然后我们用如下yaml创建一个MyPod对象：

```yaml
apiVersion: harryyann.github.io/v1
kind: MyPod
metadata:
  name: mypod-sample
  namespace: test-mypod
spec:
  podAnnotations:
    app: mypod
  podLabels:
    app: mypod
  podSpec:
    containers:
    - image: gcr.io/google-containers/pause:3.1
      imagePullPolicy: IfNotPresent
      name: main
      resources:
        limits:
          cpu: 50m
          memory: 50Mi
    nodeSelector:
      project: test
```

集群中就可以看到这个MyPod对象已经存在了：

```bash
# kubectl get mypod -n test-mypod
NAME           AGE
mypod-sample   4m33s
```

然后我们观察manager的日志，发现pod创建成功了：

```verilog
1.6476801088302636e+09  INFO    controller.mypod        create pod mypod-sample success {"reconciler group": "harryyann.github.io", "reconciler kind": "MyPod", "name": "mypod-sample", "namespace": "test-mypod"}
```

在集群上查看，mypod-sample这个pod确实已经创建起来了：

```bash
# kubectl get po -n test-mypod
NAME                             READY   STATUS    RESTARTS   AGE
mypod-manager-56d6687f75-h8cwd   1/1     Running   0          6m20s
mypod-sample                     1/1     Running   0          5m54s
```

这就证明我们的Operator生效了。



















