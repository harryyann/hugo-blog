---
title: "K8s的RBAC权限控制机制"
date: 2021-06-25T23:55:43+08:00
Description: ""
draft: false
Tags: [k8s,golang]
Categories: [k8s]
DisableComments: false
---

## RBAC(Role-Based Access Control)

基于角色的权限控制，其实只有三个最基本的概念：

- Role：角色，每个角色都定义了一组对k8s API对象的操作权限，就是一系列权限的集合
- Subject：权限被作用者，即Role提供的权限要作用到的用户，人，或机器
- RoleBinding：通过RoleBinding把Role和Subject绑定到一起

总之，就是通过Role来定义权限，通过Subject来认证，通过RoleBinding将Role和Subject绑定到一起，那么Subject即可拥有Role所定义的权限了。

## Role和RoleBinding

k8s的Role分为普通的role和clusterRole，前者是namespaced的，定义了对一个命名空间中的某些API对象所拥有的权限，而后者则是集群级别的，可以定义所有API对象(包括不是命名空间的API对象，如node)的操作权限。

role需要使用roleBinding和subject绑定，而clusterRole则使用clusterRoleBinding和subject绑定。即role和roleBinding是namespaced对象，而clusterRole和clusterRoleBinding是集群级别的对象。

Role的定义方式如下：

```yaml
kind: Role
apiVersion: rbac.authorization.k8s.io/v1  # 不同k8s版本可能会有所不同
metadata:
  namespace: {{命名空间}}
  name: {{role的名字}}
rules:
- apiGroups: [""]      # api组列表，不写就是核心组，*就代表任意组
  resources: ["pods"]  # 想要操作的API对象的列表
  verbs: ["get", "watch", "list"] # 操作谓词，和HTTP请求谓词类似
```

RoleBinding的定义方式如下：

```yaml
kind: RoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: {{roleBinding的名字}}
  namespace: {{命名空间}}
subjects:
- kind: User    # 被作用者是一个用户，大部分时候我们使用k8s的内置用户就可以了，或者ServiceAccount
  name: {{用户的名字}}
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: {{想要绑定的role}}
  apiGroup: rbac.authorization.k8s.io
```

clusterRole和clusterRoleBinding的定义除了名字不同，没有namespace字段，和可操作的API对象更多(包含集群级别的对象)之外，其他基本一致。

#### 默认的clusterRole

k8s提供了系统保留的clusterRole供我们使用，名字大都以system：开头，除此之外还包含了四个预定义好的clusterRole来给用户使用：

- cluter-admin：拥有整个k8s集群中的所有权限
- admin：拥有所有namespaced对象的所有权限
- view：namespaced对象的list、get、watch权限
- edit：namespaced对象的edit权限

更多clusterRole都可以通过命令`kubectl get clusterRole`看到

## ServiceAccount

serviceAccount是k8s的内置用户，也是我们最常使用的。

serviceAccount的定义：

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  namespace: {{命名空间}}
  name: {{serviceAccount名字}}
```

然后这个serviceAccount就可以被作为Role使用了，只需要通过roleBinding或者clusterRoleBinding将serviceAccount和Role或ClusterRole绑定就可以了：

```yaml
kind: RoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: example-rolebinding
  namespace: mynamespace
subjects:
- kind: ServiceAccount # 选择ServiceAccount
  name: example-sa
  namespace: mynamespace
roleRef:
  kind: Role
  name: example-role
  apiGroup: rbac.authorization.k8s.io
```

这样example-sa这个serviceAccount就拥有了example-role的权限。

**用户组**

就是一组用户的意思，如果为k8s配置了外部认证服务的话，那么这个用户组的概念就会由外部认证服务提供而对于内置用户serviceAccount来说，也有自己的用户组，一个sa，实际上在k8s中的名字是：

```
system:serviceaccount:<Namespace名字>:<ServiceAccount名字>
```

对应的用户组就是：

```
system:serviceaccounts:<Namespace名字>
```

如果想要把某个role或者clusterRole绑定到一个用户组中，则在roleBinding或者clusterRoleBinding中就可以这样写subject字段：

```yaml
subjects:
- kind: Group
  name: system:serviceaccounts
  apiGroup: rbac.authorization.k8s.io
```

上面这个绑定，就意味着这个role将作用于所有的sa。



## 在Pod内访问kube-apiserver

做k8s开发在pod内访问kube-apiserver是不可避免的，如果我们通过Client-go或者python的client访问kube-apiserver就需要拿到一个kubeconfig文件，这个文件中会定义kube-apiserver的LB地址，以及认证相关的token等等。

我们可以在创建pod时指定serviceAccount，就可以根据这个serviceAccount的token挂载到pod内的`/var/run/secrets/kubernetes.io/serviceaccount`目录下，那我们只需要加载目录就可以使用client访问kube-apiserver了。

比如我们可以进入一个pod中查看这个目录下：

```bash
$ kubectl exec -it sa-token-test -n mynamespace -- /bin/bash
root@sa-token-test:/# ls /var/run/secrets/kubernetes.io/serviceaccount
ca.crt namespace  token
```

namespace文件中会写入这个pod所在的namespace，另外两个就是访问kube-apiserver时的认证文件了。

如果一个pod在创建时没有指定serviceAccount，则k8s会在pod所在的namespace下为这个pod创建一个名为default的默认serviceAccount，然后分配给pod。



## 在pod外访问kube-apiserver

Pod外有很多种访问方式，你可以通过直接访问kube-apiserver的HTTP接口，也可以使用client-go的官方客户端访问，或者我们平时使用kubectl访问也是一种外部访问的方式。

### 通过HTTP访问

直接访问HTTP接口需要填入token，只需要添加一个Authorization的header，然后填入token即可，格式大概如下，我们以获取所有pod的接口为例：

```bash
$ curl --location --request GET 'https://10.10.10.10:8080/api/v1/pods' \
--header 'Authorization: Bearer {{token}}'
```

注意token前面要加上`Bearer `的前缀。

#### token的获取

每个serviceAccount都会有一个secret对象，而secret中就保存着token，我们在创建role和serviceAccount之后，通过创建roleBinding就可以让serviceAccount获得权限，然后我们就可以拿到serviceAccount的token了。比如：

```bash
# kubectl describe sa default
Name:                default
Namespace:           default
Labels:              <none>
Annotations:         <none>
Image pull secrets:  <none>
Mountable secrets:   default-token-sdchv
Tokens:              default-token-sdchv
Events:              <none>
```

可以看到default这个serviceAccount的secret和token名叫default-token-sdchv。

然后我们就可以describe一下就拿到了token，或者`-o yaml`也能拿到：

```bash
# kubectl describe secret default-token-sdchv
Name:         default-token-sdchv
Namespace:    default
Labels:       <none>
Annotations:  kubernetes.io/service-account.name: default
              kubernetes.io/service-account.uid: 33490e00-9424-45eb-ab20-4e87e1da80b4

Type:  kubernetes.io/service-account-token

Data
====
ca.crt:     1371 bytes
namespace:  7 bytes
token:      eyJhbGciOiJSUzI1NiIsImtpZCI6ImoteF9FN1pjZVllNTRvcEVnOUd3NzA1UzJPS2JNUFdxMFdwSXQ3a09DZncifQ.eyJpc3MiOiJrdWJlcm5ldGVzL3NlcnZpY2VhY2NvdW50Iiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9uYW1lc3BhY2UiOiJkZWZhdWx0Iiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9zZWNyZXQubmFtZSI6ImRlZmF1bHQtdG9rZW4tc2RjaHYiLCJrdWJlcm5ldGVzLmlvL3NlcnZpY2VhY2NvdW50L3NlcnZpY2UtYWNjb3VudC5uYW1lIjoiZGVmYXVsdCIsImt1YmVybmV0ZXMuaW8vc2VydmljZWFjY291bnQvc2VydmljZS1hY2NvdW50LnVpZCI6IjMzNDkwZTAwLTk0MjQtNDVlYi1hYjIwLTRlODdlMWRhODBiNCIsInN1YiI6InN5c3RlbTpzZXJ2aWNlYWNjb3VudDpkZWZhdWx0OmRlZmF1bHQifQ.ivItYHIZgT4rm62TfYZ22M-OG3ezSx6pxlIBwTwLPOByZ4Kap3DJZMgdiRB4q2H4DVASp7TyYB58KBdRpVDLflf-UHCUj1OwNsAP6aJo9bGUa0MHKBty2Of7DevAm-5HBzoZ5q3b_sede_gI-7Hf0xaN6uIqaYhns-uMDsNQ6XftRGzcjIXw5r9UyllTfRdgRnG21JjUQtHkcv_ChHXCqNZQ9jL32mTB9KcUGmf98pXklEuRPRPDdwzSI1ZgYp2PQANagQrnJg5Y1XnWsOc_xJ4Udi8NLNH2dZHbRL12grOmwCvgYhOFe0lo4Qc7bL7cSHDEMddXVFUz_ZQj1ym3YA
```

拿到token后，把token填进去然后访问就可以了。

### 通过client-go或者kubectl访问

当使用kubectl想要访问一个集群时，则需要使用kubectonfig，kubeconfig是一个包括了集群的信息，还有认证信息的文件，它的格式大概长这样：

```yaml
apiVersion: v1 
kind: Config 
users: 
- name: svcs-acct-dply 
  user: 
  token: <replace this with token info> 
clusters: 
- cluster: 
    certificate-authority-data: <replace this with certificate-authority-data info> 
    server: <replace this with server info> 
  name: self-hosted-cluster 
contexts: 
- context: 
    cluster: self-hosted-cluster 
    user: svcs-acct-dply 
  name: svcs-acct-context 
current-context: svcs-acct-context
```

看得出来，这就是一个Config对象的yaml文件，其中里面有三部分比较重要，分别是：

- certificate-authority-data：包含集群的认证信息
- server：集群apiserver的地址
- token：访问apiserver使用的token

通过在master节点上执行以下命令可以获得一个admin级别的kubeconfig

```bash
# kubectl config view --flatten --minify > kubeconfig.conf
```

用上述方法获取sa的token（保证sa已经绑定好了role或clusterRole）将kubeconfig.conf改造成上面的kubeconfig的格式，填入token即可使用。

有了kubeconfig我们通过kubectl访问kube-apiserver时只需要通过flag`--kubeconfig`指定kubeconfig文件即可，client-go等客户端也可以加载这个文件，然后就可以访问了，这里不多赘述。

## Reference

https://hex108.gitbook.io/kubernetes-notes/sheng-chan-li-xiao-gong-ju/generate-kubeconfig

