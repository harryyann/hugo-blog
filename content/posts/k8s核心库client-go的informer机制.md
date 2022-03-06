---
title: "k8s的Informer机制"
date: 2021-03-12T23:55:43+08:00
Description: ""
draft: false
Tags: [k8s,golang]
Categories: [k8s]
DisableComments: false
---

### Client-go

client-go与其他语言版本的k8s不同的地方在于，它不仅仅是一个k8s的客户端，而且还是一个k8s的核心库，k8s中各个组件与api-server之间的通信都是通过client-go的iformer机制实现的快速无延迟的数据同步。

### Informer机制

![Image](/images/informer.png)
