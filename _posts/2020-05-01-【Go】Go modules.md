---
layout:         page
title:          【Go】Go modules
date:           2020-05-01
author:         翼
header-img: image/bg.jpg
catalog: true
tags:
---


# Go module
Go在1.11版本中引入了名为Go Modules的内置包管理工具，它是GOPATH的替代品，集成了版本控制和软件包分发支持。

Go 1.12版本默认启用Go Modules，GOPATH将在1.13版本中弃用。也就是说，在此之后，我们不必再设置GOPATH，然后在GOPATH中创建项目以解析自定义包路径。

#### 1. module & GOPATH
在**Go 1.11**中, 如果当前目录或任一父级目录存在一个go.mod文件，并且这些目录不在 **\$GOPATH/src** 中，go命令即启用module模式。为了保持兼容性，在 **\$GOPATH/src** 中，go命令仍然启用老的 **GOPATH** 模式。

从**Go 1.13**开始, go命令默认启用module模式。

### 2. module定义 & 定义module
官方的定义：A module is a collection of Go packages stored in a file tree with a go.mod file at its root.

将一个代码目录作为go module的操作很简单，只需要在此目录中执行：go mod init MODULE_PATH，go会在此目录生成一个go.mod文件。
```go
$ cd concurrent
$ go mod init sample.com/concurrent
$ cat go.mod

module sample.com/concurrent

go 1.14
```

**go.mod**文件主要定义了两个内容：
1. 该module的**模块路径**。这也是模块根目录的导入路径。
2. 该module的**依赖项**，即它依赖的其他modules。每条依赖项包含了一个**模块路径**和一个特定语义的**版本号**。

### 3. package（inside module）解析
go在解析导入的包时，会判断go.mod文件中列出的依赖模块（module）的版本号。

如果go.mod中没有任何一个module提供该导入的package，go就自动查找包含此package的module的**最新版本**。
**最新版本**的定义：
1. the latest tagged stable (non-prerelease) version
1. 其次，the latest tagged prerelease version
1. 其次，the latest **untagged** version，例如：v0.0.0-20170915032832-14c0d48ead0c

当然，我们可以指定项目依赖于特定版本（tagged）的第三方module。

### 4. module's semantic version
在Go modules中，版本号是通过一种语义版本标记（semantic version tags）表示的。

一个版本号包含三个部分：**major**, **minor**, 以及 **patch**。例如版本号 v0.1.2，major版本是0，minor版本是1，patch版本是2。需要注意的是，major版本和minor版本的**更新规则**是不一样的。

##### 4.1 minor版本更新
可以先执行命令：go list -m all， 来查看当前依赖的module以及对应版本号的列表。如：
```go
$ go list -m all
tm.com/concurrent
github.com/gorilla/mux v1.7.4
golang.org/x/text v0.0.0-20170915032832-14c0d48ead0c
rsc.io/sampler v1.3.0

```
而对应的go.mod文件内容如下：
```go
module tm.com/concurrent

go 1.13

require (
	github.com/gorilla/mux v1.7.4
	rsc.io/sampler v1.3.0 // indirect
)

```
**indirect**注释表示当前module并没有直接依赖此依赖项，而是有其他依赖模块在依赖此模块，或者，根本没有模块依赖此模块。比如这里的rsc.io/sampler模块，其实是我执行go get rsc.io/sampler@v1.3.0 后被go添加进go.mod文件的，项目其实根本没有依赖此module。

尽管假设模块rsc.io/sampler是真的被项目依赖，先来看看，如何升级此模块的minor版本。执行命令：go get rsc.io/sampler，再查看go.mod文件内容：
```go
module tm.com/concurrent

go 1.13

require (
	github.com/gorilla/mux v1.7.4
	rsc.io/sampler v1.99.99 // indirect
)

```
可见，默认情况下，go get命令会查找或者更新模块到**最新的minor版本**。这里最新版本的定义，已经在上面提过了。

##### 4.2 minor版本回退
按照go的规范，同一个minor版本是要保证**向前兼容**的。也就是说，如果我们的模块依赖rsc.io/sampler模块的v1.3.0版本时能够正常运行，那么当升级到同一个minor版本的新版本v1.99.99时，也应该是能正常运行的。

但是，这种“向前兼容”，并非go给开发者施加的一种限制，而只是一种规范建议。所以，可能升级rsc.io/sampler模块到v1.99.99版本后，我们的concurrent模块并不能正常工作了。

这个时候，我们就得回退到v1.99.99之前的版本试试。那么我们可以查看一下rsc.io/sampler模块的可用的已标记版本，执行命令 go list -m -versions rsc.io/sampler：
```go
$ go list -m -versions rsc.io/sampler
rsc.io/sampler v1.0.0 v1.2.0 v1.2.1 v1.3.0 v1.3.1 v1.99.99

```
可见，在我们之前能正常使用的v1.3.1版本和目前无法正常使用的最新版本v1.99.99之间，还有一个版本v1.3.1。我们可以尝试使用这个稍新的版本试试，执行命令：go get rsc.io/sampler@v1.3.1：
```go
$ go get rsc.io/sampler@v1.3.1
go: finding rsc.io/sampler v1.3.1
go: downloading rsc.io/sampler v1.3.1
go: extracting rsc.io/sampler v1.3.1

```
再查看go.mod文件，会发现我们依赖的rsc.io/sampler模块的版本已经降低到了v1.3.1:
```go
module tm.com/concurrent

go 1.13

require (
	github.com/gorilla/mux v1.7.4
	rsc.io/sampler v1.3.1 // indirect
)

```
这个时候，我们就能再次验证一下，v1.3.1版本的模块，能否满足项目需要了。

##### 4.3 清理多余的module
随着项目不断开发，可能有些模块已经不需要了，例如上面被备注了"indirect"的模块rsc.io/sampler。对 go build 来说，在构建过程中发现当前构建的模块的依赖模块是否有缺失，或是否需要添加依赖模块，是比较容易的。但是判断一个依赖模块是否能被安全移除，是比较困难的。所以go build命令并不会清理依赖模块，这个时候，需要执行命令 go mod tidy。执行此命令之后，再查看go.mod文件内容：
```go
module tm.com/concurrent

go 1.13

require github.com/gorilla/mux v1.7.4

```
可以发现，rsc.io/sampler模块已经被移除，因为我们的concurrent模块实际并不依赖它。

##### 4.4 依赖 major版本的module
Go module的每个不同的major版本使用不同的模块路径：从v2开始，模块路径必须以major版本结尾。例如， rsc.io/quote模块的v3版本的模块路径不再是 rsc.io/quote，而是 rsc.io/quote/v3。因为major版本与minor版本不同，go的规范建议是major版本可以不与旧版本兼容，或者说，major版本应该是不与旧版兼容的，否则新版仍应该升级minor版本。

因为module实际是由module path标识的，不同的module path就使得我们的module可以依赖一个module的多个major版本。同时，这也使得我们可以在不更新（迁移）依赖模块的老版本的情况下，在新业务中使用依赖模块的新版本（major版本）。这就是所谓“**增量迁移**”。

本质上，我们可以认为，一个module的不同major版本，其实本就是不同的module。这个理解，可以使很多疑惑以及module依赖操作迎刃而解。只有一点区别，就是这些module的默认导入名称都相同。比如"rsc.io/quote/v3"和"rsc.io/quote"的默认导入名称都是quote。

### 5. go.sum
除了go.mod文件，go还维护了一个go.sum文件。这个文件里面记录的是我们依赖的module版本的内容的密码检验和（cryptographic checksum）。go命令通过这个文件来确保今天下载的某个module版本与昨天下载的一样，以保证构建过程是可重复的，以及发现意外的变更。

**因此，go.mod以及go.sum都必须提交到代码仓库中。**