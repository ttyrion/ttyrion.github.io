---
layout:         page
title:         【Docker】Best practices for writing Dockerfiles -part2
subtitle:       
date:           2019-06-09
author:         翼
header-img: image/bg.jpg
catalog: true
tags:
---

> 这篇文章的主要内容是翻译自Docker官网，删减了部分与part1重复的基础内容。原文链接在[这里](https://docs.docker.com/v17.09/engine/userguide/eng-image/dockerfile_best-practices/)

Docker能够读取Dockerfile中的命令来自动化构建镜像。这篇文档的主题是由Docker官方以及社区推荐的构建高效镜像的最佳方法和实践。

### General guidelines and recommendations
#### 1. Containers should be ephemeral
你的Dockerfile定义的镜像生成的容器应该越短小精悍越好。“ephemeral”的意思是：容器应该能被停止、销毁，并且能用最小的设置和配置来构建和安装新容器。

#### 2. Use a .dockerignore file
**注意，以下这段文字并非翻译，个人认为官网文档有错误。** 请对比原文查看。

执行 docker build 构建镜像时，一般需要传递了两个参数：一个Dockerfile路径（ -f ...Dockerfile ）以及一个目录路径。如果Dockerfile文件就在执行 docker build 时的工作目录中，那么可以忽略Dockerfile路径这个参数，只传一个目录路径。这个目录路径就是 **“构建上下文”**（build context）。“构建上下文”是构建镜像的二要素之一，另一个要素是 **“基础镜像”**。

官方文档说“构建上下文”是执行 docker build 时的工作目录，显然是不对的。因为我可以这么执行：
```javascript
docker build ../
```
那么，构建上下文就是这个工作目录的父目录。

执行 docker build 时，Docker CLI 会把“构建上下文”中的所有文件和子目录（递归遍历上下文目录）发送至 Docker Daemon；这些文件和子目录就是Docker daemon 构建镜像时的“构建上下文”。如果无意中在“构建上下文”目录中包含了无关文件，就会导致上下文变大，并且镜像的大小也变大。这同时也会使得构建时间变长，pull或者push镜像的时间也变长；还有运行时的容器也变大。执行 docker build 时的第一行日志就能看到你的“构建上下文”的大小：
```javascript
Sending build context to Docker daemon  187.8MB
```
为了精简构建上下文，可以使用一个.dockerignore文件来忽略目录中的一些文件或者子目录；只把必要的文件和子目录当做“构建上下文”。

#### 3. Use multi-stage builds
Docker 17.05 以及以上版本，都可以使用“multi-stage builds”来极大地减小最终的镜像大小，而不需要减少中间层数或者在构建期间删除中间文件。

本人使用的版本没这么高，后续升级版本使用后再翻译。

#### 4. Avoid installing unnecessary packages
简单来说，为了降低复杂性、依赖性、文件大小、构建时间等等，你应该避免安装额外的、非必须的包。

#### 5. Each container should have only one concern
将应用程序分离到多个容器中，能够使横向扩展以及复用容器变得容易得多。比如，为了以解耦的方式来管理web应用程序、数据库、内存缓存，一个web应用程序可以由三个独立的容器组成，每个容器有自己的特定的镜像。

你可能听说过“一个容器一个进程”。尽管这句话的意图不错，但这并非必须的。现在，容器可以由一个初始进程（init process）启动，除此之外，有些程序也会自行启动额外的进程。比如，Apache可能会为每个请求创建一个进程。虽然“一个容器一个进程”通常是一个好的经验法则，单并不是硬性规定。你应该自己做出最佳判断来尽量保持容器的干净和模块化。

#### 6. Minimize the number of layers
Docker 17.05 以前的版本，甚至是 Docker 1.10 以前的版本，减少你的镜像中的层数是很重要的。下面的一些改进减轻了这个要求：
1. Docker 1.10 以及更高版本中，只有 RUN、COPY、ADD命令会创建新层。其他命令只是创建临时的中间镜像，且并不会直接增加构建结果的大小。
1. Docker 17.05 以及更高的版本支持“multi-stage builds”，这允许你只把必须的内容拷贝进最终的镜像，把工具和调试信息只放在中间构建阶段而不增加最终镜像的大小。

#### 7. Sort multi-line arguments
只要有可能，通过以字母表顺序排序多行参数来环节后续的更改。这能帮你避免包的重复，并且更新参数也更容易。例如：
```javascript
RUN apt-get update && apt-get install -y \
  bzr \
  cvs \
  git \
  mercurial \
  subversion
```

#### Build cache
在构建镜像的过程中，Docker将一步步按顺序执行你的Dockerfile中的命令。检查每条命令时，Docker会在其缓存中查找可复用的镜像，而不是创建新（重复的）镜像。如果你不想使用缓存，可以使用 docker build 的 --no-cache=true 选项。

然后，如果你准备让Docker使用缓存，那么理解Docker什么时候会查找匹配的镜像，什么时候不会。Docker遵循的基本规则如下：
1. 从缓存中已有的一个父镜像开始，接下来的一条命令被拿来和此基础镜像（父镜像）的所有子镜像对比，来判断其中是否有某个子镜像是由相同的命令构建出来的。如果没有，则缓存无效。
1. 大部分情况下，将Dockerfile中的命令和子镜像进行对比即可。然后，某些命令需要做更多检查。
1. 对于ADD、COPY命令，镜像内的文件的**内容**也会被检查，会对每个文件计算出一个检验码。文件的最后修改时间和最后访问时间不影响检验码的计算。在查找缓存的过程中，这个检验码会被拿来与已有镜像中的检验码进行对比。如果文件的内容或者元数据等信息有任何改变，缓存无效。
1. 除了ADD、COPY之外，缓存检查不会查看容器中的文件来觉得换成是否匹配。比如，当处理 RUN apt-get -y update 时，容器中更新的文件并不会被用于判断缓存是否可用。这种情况下，Docker只是根据命令字符串本身来对比，以便找到匹配的缓存。

只要缓存无效，Dockerfile中后续的命令都会生成新镜像而不使用缓存。