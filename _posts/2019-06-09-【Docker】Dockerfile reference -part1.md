---
layout:         page
title:         【Docker】Dockerfile reference
subtitle:       
date:           2019-06-09
author:         翼
header-img: image/bg.jpg
catalog: true
tags:
---

> 这篇文章翻译自Docker官网。原文链接在[这里](https://docs.docker.com/v17.09/engine/reference/builder/)

Docker可以自动读取一个Dockerfile中的指令来构建镜像（build images）。一个Dockerfile就是一个文本文件，里面包含着用户通过命令行集成镜像时可以调用的一切命令。用户可以执行 docker build 来创建一个连续地执行多条命令行指令的自动构建过程。

这篇文章将描述一个Dockerfile文本内部可用的命令。

#### 作用（Usage）
**docker build命令 根据一个Dockerfile和一个context来构建镜像。** 构建过程的context就是指定的PATH或者URL包含的诸多文件的集合：其中，PATH是本地系统的一个目录，而URL是一个Git仓库地址。

context会被递归地处理。也就是说：指定的PATH包含了PATH以及其子目录，而URL包含对应的Git仓库以及其submodules。如下示例展示了使用当前目录作为context的build命令：
```javascript

$ docker build .
Sending build context to Docker daemon  6.51 MB
...

```

**真正的构建过程（build）是由 Docker daemon 执行的，而不是Docker CLI。** 因此build过程中第一件事就是把整个context（递归地）发送到daemon。大部分情况下，最好从一个空目录开始，将它作为build的context，此目录内只添加Dockerfile以及其它构建镜像所必要的文件。
```javascript
作为一个补充提示：千万不要把根目录（/）作为context的PATH，因为这将导致docker把整个磁盘的内容发送给Docker daemon。

```

Dockerfile要在命令（如COPY）中才能引用context中的文件。可以在context目录中添加一个.dockerignore文件来去除一些文件和目录，以便提高build性能。

习惯上，Dockerfile位于context的根目录中，这也是docker默认的Dockerfile路径。但是你可以通过-f标志指定一个Dockerfile：
```javascript

$ docker build -f /path/to/a/Dockerfile .

```
注意上面命令的末尾指定了"."作为context，通过-f指定了Dockerfile，这就是 docker build 构建镜像所需的全部**两个要素**。

你还可以指定一个repository并且给它打上tag来保存新构建的镜像：
```javascript

$ docker build -t shykes/myapp .

```

可以给build命令添加多个-t参数，以便在构建成功后给镜像打上多个repositories的tag：
```javascript

$ docker build -t shykes/myapp:1.0.2 -t shykes/myapp:latest .

```

Docker daemon执行Dockerfile中的命令之前，会对Dockerfile进行初步校验，并且在语法不正确时返回错误：
```javascript

$ docker build -t test/myapp .
Sending build context to Docker daemon 2.048 kB
Error response from daemon: Unknown instruction: RUNCMD

```

Docker daemon 一个接一个地执行Dockerfile中的命令，并且在必要的时候把每条命令的结果提交到新镜像中，最后输出新镜像的ID。**Docker daemon 会自动清理 Docker CLI 发送的context。**

注意：每条命令的执行都是独立的，并且会创建新的镜像。只要有可能，Docker就会复用临时镜像（cache），以便加速处理 docker build。这一点，可以从命令行输出的日志中的“Using cache”看出来：
```javascript

$ docker build -t svendowideit/ambassador .
Sending build context to Docker daemon 15.36 kB
Step 1/4 : FROM alpine:3.2
 ---> 31f630c65071
Step 2/4 : MAINTAINER SvenDowideit@home.org.au
 ---> Using cache
 ---> 2a1c91448f5f
Step 3/4 : RUN apk update &&      apk add socat &&        rm -r /var/cache/
 ---> Using cache
 ---> 21ed6e7fbb73
Step 4/4 : CMD env | grep _TCP= | (sed 's/.*_PORT_\([0-9]*\)_TCP=tcp:\/\/\(.*\):\(.*\)/socat -t 100000000 TCP4-LISTEN:\1,fork,reuseaddr TCP4:\2:\3 \&/' && echo wait) | sh
 ---> Using cache
 ---> 7ea8aef582cc
Successfully built 7ea8aef582cc

```
只有拥有父链的镜像才能被作为构建缓存使用：这类镜像是先前的build过程创建的，或者这个镜像链是由 docker load 命令加载的。你可以通过--cache-from指定某个镜像作为构建缓存使用，并且，--cache-from指定的镜像不需要拥有父链，甚至可以是从其他registries拉取的镜像。

构建完成后，就可以“Pushing a repository to its registry”。

#### Dockerfile的格式
Dockerfile的格式如下：
```javascript

# Comment
INSTRUCTION arguments

```
docker命令并不区分大小写。不过，习惯上使用大写的命令以便与参数区分开。

一个Dockerfile文件的第一条命令必须是**FROM**。FROM命令指定了你要构建的镜像的**基础镜像**（Base Image）。

另外，Docker把以#开头的行当做注释行，除非这一行是个有效的解析预处理器(parser directive)。任何其他位置的#也会被当做命令的参数，例如：
```javascript

# Comment
RUN echo 'we are running some # of cool things'

```
##### Parser directives
解析预处理器是可选的，并且它会影响Dockerfile文件中后续的行被处理的方式。预处理器并不会在构建镜像的过程中增加层，也不会显示为构建的一个步骤。预处理器以一种特殊的注释的形式书写：
```javascript

# directive=value

```

一个预处理器只能使用一次。因此，下面的方式是无效的：
```javascript

# directive=value1
# directive=value2

FROM ImageName

```

只要Docker处理过一条注释、空行、build命令，就不会再处理预处理器。任何预处理器格式的行都会被当做注释。**因此，所有的解析预处理器都必须放在Dockerfile的最顶部。**比如，下面的预处理器会被当做注释：
```javascript

# About my dockerfile
# directive=value
FROM ImageName

```


预处理器也不区分大小写，但是惯例是使用小写（相对于命令使用大写）。惯例上，也会在预处理器后面包含一个空行。另外，行连接符对预处理器行无效。

有以下可用的预处理器：

###### Parser directives：escape
escape预处理器设置作为转义字符的字符，默认的转义字符是'\\'。
```javascript

# escape=\ (backslash)

```
或者：
```javascript

# escape=` (backtick)

```
默认的转义字符'\\'既可以用于转义其它字符（escape other characters），也可以用与行连接（转义换行， escape a newline），这允许Dockerfile中的一条命令横跨多行。注意：不管Dockerfile中是否使用了escape预处理器，RUN命令中都不会发生转义，除非转义出现在这一行的末尾。

考虑下面这个Windows系统中的Dockerfile：
```javascript

FROM microsoft/nanoserver
COPY testfile.txt c:\\
RUN dir c:\

```
第二行末尾的'\\'字符会被当做转义新行的转义字符（连接多行），而不是第一个'\\'字符的转义目标。第三行的'\\'也类似。最终，这个Dockerfile中的第二行和第三行会被作为一条命令执行：
```javascript

PS C:\John> docker build -t cmd .
Sending build context to Docker daemon 3.072 kB
Step 1/2 : FROM microsoft/nanoserver
 ---> 22738ff49c6d
Step 2/2 : COPY testfile.txt c:\RUN dir c:
GetFileAttributesEx c:RUN: The system cannot find the file specified.
PS C:\John>

```
注意看日志中的"Step 2/2"。

可以在上面的Dockerfile中添加一个escape解析预处理器：
```javascript

# escape=`

FROM microsoft/nanoserver
COPY testfile.txt c:\
RUN dir c:\

```
这个Dockerfile里面，文件路径可以使用Windows系统的自然语义。

#### Environment replacement
环境变量（由ENV语句声明的）可以作为由Dockerfile解释的变量，在某些命令中使用。还会处理转义，以便在语句字面量中包含类似变量的语法。

Dockerfile中的环境变量，可以用\$variable\_name或者\${variable\_name}表示，二者等价。通常使用花括号的形式来解决不带空格字符的变量名的问题，如 like ${foo}_bar。

${variable_name}语法还支持某些标准的bash修饰符。比如：
```javascript

1. ${variable:-word} 
It indicates that if variable is set then the result will be that value. 
If variable is not set then word will be the result.

2. ${variable:+word} 
It indicates that if variable is set then word will be the result, otherwise the result is the empty string.

```
不论哪种形式，word可以是任何字符串，甚至是其他环境变量。

Dockerfile中的以下命令都支持环境变量：
```javascript

ADD
COPY
ENV
EXPOSE
FROM
LABEL
STOPSIGNAL
USER
VOLUME
WORKDIR

```
当与上述命令组合使用时，ONBUILD也支持环境变量。

#### .dockerignore file
在将context发送给docker daemon之前，docker CLI 会先查看context根目录下的.dockerignore文件。如果存在这样一个文件，CLI 会修改context：排除相应的文件和目录。这防止把不必要的或敏感的内容发送到docker daemon，这还可能导致这些不必要的内容被 ADD 或者 COPY 命令添加到镜像中。

.dockerignore文件的每一行都会被CLI视为一个模式。为了匹配模式，context的根目录会被同时当作工作目录和根目录。举例来说，模式/foo/bar和foo/bar都会排除PATH或者URL指定的git仓库的根目录的foo子目录中的名为bar的文件或者目录。

.dockerignore中以#作为第一个字符开头的行，会被当作注释。注释行在CLI解释.dockerignore之前就会被忽略。

比如下面这个.dockerignore文件：
```javascript

# comment
*/temp*
*/*/temp*
temp?

```
此文件引起的build行为如下：
```javascript

1. # comment
忽略。

2. */temp*
排除root目录的直接子目录内的名字以temp开头的文件和目录。
比如文件/somedir/temporary.txt会被排除，目录/somedir/temp也会被排除。

3. */*/temp*
排除root目录的二级子目录内的名字以temp开头的文件和目录。
比如文件/somedir/subdir/temporary.txt会被排除。

4. temp?
排除root目录中名字以temp开头，结尾是任意单个字符的文件或者目录。
比如文件/tempa和目录/tempb会被排除。
```
模式匹配是根据Go的filepath.Match的规则来处理的。还有一个预处理的步骤，它根据filepath.Clean的规则，将开头和结尾的空白字符去掉，并且清除 . 和 .. 。

除了Go的filepath.Match规则，Docker还支持一个特殊的通配符"**"，可以匹配任意数量的目录。比如，模式 \*\*/*.go 将排除所有目录（包括build context的根目录）中的所有名字以.go结尾的文件。

以'!'开头的行，可以用作排除项的例外项。比如：
```javascript

    *.md
    !README.md

```
所有的markdown文件，除了README.md，都会从context中排除。

