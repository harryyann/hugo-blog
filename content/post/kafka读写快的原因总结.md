---
title: "Kafka读写快的4点原因总结"
date: 2022-03-12T00:19:51+08:00
Description: ""
Tags: [kafka, 零拷贝]
Categories: [消息中间件]
DisableComments: false
draft: true
---

# Partiton

## 并行写

Kafka的每个消息都会发到某个topic下的一个partition，而不同的partition的leader副本往往是位于不同的broker机器上的，所以消息如果不是发给同一个partition，实际上都是可以并行处理的，发挥多个机器并行处理的优势。

## 顺序写

对于同一个partiton来说，写入操作都是顺序写，磁盘的顺序IO要比随机IO快的多，甚至可以超过内存的随机IO速度。

# Page cache

Kafka使用page cache来批量的进行刷盘操作。Page cache是操作系统内核态的buffer。使用内核态buffer的好处就是，如果Kafka的broker重启了，page cache的数据依然不会丢，除非机器宕机，这是一种保证数据完整和效率之间的权衡。

Page cache会包含多个buffer cache，用于文件系统的数据缓存，写入都是先写入buffer，然后再批量刷盘：

- IO Scheduler会将连续的小块写组装成一个大块写来提高性能；
- IO Scheduler还会尝试将一些写操作重排，从而减少磁盘的磁头移动。

对于读操作，也是会先读page cache，如果生产和消费速度相当，甚至直接在cache中就可以交换数据，落盘反而成了一个异步操作。

# Zero-copy

Kafka的数据由生产者发送到落盘，和磁盘文件通过网络发送给消费者的两个过程，都使用了零拷贝技术。



# 批处理和数据压缩

Kafka的客户端在通过网络发送数据给broker时，会在一个批处理中累计多条记录（包括读和写），发送记录的批处理分摊了网络往返的开销，使得带宽利用率得到提升。

生产者还可以将数据压缩后再发送给broker，从而减少网络数据的传输量，支持的算法有Snappy、GZip、LZ4等。数据压缩一般都是和批处理配套使用进行优化的。