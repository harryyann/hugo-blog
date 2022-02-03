---
title: "学习编写Makefile"
date: 2021-03-25T20:45:07+08:00
Description: ""
Tags: [make]
Categories: [工具使用]
DisableComments: false
---

### make工具介绍

项目构建往往要经过一系列的过程，尤其是C项目，可能需要先编译这个，再编译那个，再添加个链接，完了再打个docker镜像，会有很多的文件需要创建和更改。这些动作，我们既可以通过手工的方式，一步步完成，不免显得繁琐。或者也可以写个shell脚本自动完成。但是make是专门做这件事的。

#### make的环境准备

对于linux或者mac系统，只要有gcc，一般都会有make，通过命令`make -v`可以检查你当前的环境有没有make，而在windows上想使用make还是比较复杂的，尽管网上有很多教程，但我都没有尝试成功，推荐还是尽量在linux或者mac环境下使用make构建你的项目

#### make的使用

make的使用非常简单，笔者不是make大神，使用make都是通过一条 `make` 命令和编写一个Makefile文件即可。

### Makefile的编写

Makefile中需要指明要进行哪些操作，这些操作需要有哪些前置操作，或者需要哪些前置文件，最终又生成了什么文件。

Makefile的通用格式为：

```shell
<target> : <prerequisites>
[tab]  <commands>
```

- target是这个操作要实现的目标，可以只是一个标签，也可以是一个文件名
- prerequisites指的是这个操作的需要的前置条件，这个没有可以不写，前置条件既可以是一个文件名，也可以是一个标签
- tab是一个制表符，注意是制表符，不是空格
- commands就是我们需要执行的shell命令了，一个操作可以有一个或多个命令，多个命令可以用`\`进行换行

举个例子

```makefile
NAME = test
VERSION = `date +%Y%m%d`  # 今天的日期
NCR = ncr.nie.com/yanghairui/

# docker-push是target,docker-build是这个操作的前置操作
docker-push: docker-build
   docker push $(NCR)$(NAME):$(VERSION)

docker-build: go-build
   docker build . -t $(NCR)$(NAME):$(VERSION)

go-build:
   GO111MODULE=on \
   GOOS="linux" \
   GOARCH="amd64" \
   go build -o test 
```

在Makefile所在的目录输入`make`，然后按回车，会按以下顺序执行操作：

- 上面这个例子执行会先执行go-build标签的内容，编译一个go项目
- 然后执行docker-build，打一个docker镜像
- 最后将这个docker镜像push到远程仓库、
- `#` 号后是注释
- 变量在定义完用`$(变量名)`引用
- 还有一些内置变量，但是并不常用，如：`$(CC)`指当前使用的编译器，`$(MAKE)`指当前使用的make工具

