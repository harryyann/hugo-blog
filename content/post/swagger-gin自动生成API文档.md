---
title: "swagger-gin自动生成API文档"
date: 2021-03-16T22:22:10+08:00
Description: "使用swagger-gin对gin框架生成的后端服务自动生成API文档"
Tags: [swagger,gin]
Categories: [其他]
DisableComments: false
---

### swagger

swagger是通用的API文档生成工具，那么对于go语言的gin框架，也有专用的swagger工具swagger-gin

### 使用

swagger-gin的使用非常简单，只需要下载包、添加注释、添加路由、命令行生成swagger文档即可。

- 安装依赖包：

```bash
$ go get github.com/swaggo/gin-swagger
$ go get github.com/swaggo/swag 
$ go get github.com/swaggo/files
```

- 在你的代码中添加注释
  
  swagger-gin主要就是根据这些注释生成API文档的。
  
  首先，在你的main函数所在的.go文件中添加这些注释，这些都是一些元信息

```go
// @title cluster-inspector
// @version 1.0
// @description a tool for inspecting clusters
// @contact.name wb.yanghairui
// @contact.email wb.yanghairui@mesg.corp.netease.com
// @license.name Apache 2.0
// @license.url http://www.apache.org/licenses/LICENSE-2.ml
// @schemes http
// @host localhost:9999
// @BasePath /api/v2/health
```

​        然后在你对应的handler函数中，添加对每个接口的注释

```go
// @Tags Register
// @Summary Register a cluster's information into the database
// @Accept application/json
// @Produce application/json
// @Param body body swaginfo.ClusterRegisterRequest true "the information of a cluster"
// @Success 200 {object} swaginfo.ClusterRegisterSucceedResponse true "information of the clusters which is effected"
// @Router /cluster [post]
```

- 生成API文档
  
  在main函数所在文件的路径下，使用命令

```bash
$ swag init
```

可以在根路径下看到生成了一个docs目录，里面有三个文件，就是我们的swagger文档了

- 添加路由
  
  在你的路由中添加一行，即可

```go
router.GET("/swagger/*any", ginSwagger.WrapHandler(swaggerFiles.Handler))
```

​        编译执行你的程序，在浏览器中输入这个路由，就能看到可交互式的swagger文档了

### 注释信息详解
