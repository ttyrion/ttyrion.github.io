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

#### FROM
```javascript
FROM <image> [AS <name>]
FROM <image>[:<tag>] [AS <name>]
FROM <image>[@<digest>] [AS <name>]
```
FROM命令初始化新的构建阶段（build stage），并且**为后续的命令设置基础镜像**（Base Image）。这意味着：一个有效的Dockerfile必须以FROM命令开始。基础镜像可以是任何有效的镜像。
1. ARG是唯一的可以出现在FROM之前的命令。
2. 一个Dockerfile中，可以出现多个FROM命令：创建多个镜像，或者将某个构建阶段独立于其他阶段。需要注意的仅仅是FROM命令之前的commit输出的image ID。因为FROM命令会清空它之前的命令生成的所有状态。
3. 可以通过给FROM命令添加一个 AS name, 来给新的构建阶段一个可选的名字。这个名字可以被后续的 FROM 和 COPY --from=<name|index> 命令用来引用当前阶段构建出的镜像。
4. tag和digest值是可选的。如果你忽略了其中一个值，构建过程会使用默认tag值：latest。

##### FROM：Understand how ARG and FROM interact


#### RUN
RUN有两种形式：
```javascript
RUN <command> (shell form)
RUN ["executable", "param1", "param2"]  (exec form)
```
RUN命令将在当前镜像的顶层之上的新层中执行commands，并提交结果。这个结果镜像将被Dockerfile中下一个步骤使用。

将RUN命令**分层并提交**结果，符合Docker的核心概念：“commits are cheap”，并且可以从镜像的历史中的任何一点创建容器，很像源代码版本控制。

RUN命令的exec形式（上面第二种）使得RUN命令能够使用某个并不包含可执行的shell的基础镜像。另外，可通过SHELL命令来更改shell形式（上面第一种）默认的shell。

在shell形式中，可以用'\\'把一个RUN命令分多行书写。比如，下面的两个RUN命令是一模一样的：
```javascript
RUN /bin/bash -c 'source $HOME/.bashrc; \
echo $HOME'
```
```javascript
RUN /bin/bash -c 'source $HOME/.bashrc; echo $HOME'
```
注意：RUN命令的exec形式并不会像shell形式那样启动命令行shell。这意味着，正常的shell处理逻辑并不可用。比如，**RUN [ "echo", "$HOME" ]**不会对$HOME进行变量值替换。如果你需要类似这样的shell处理逻辑，就使用RUN命令的shell形式或者直接启动一个shell，如 **RUN [ "sh", "-c", "echo $HOME" ]**。使用shell形式或者直接启动shell时，处理环境变量的是shell，而不是docker。

RUN命令的缓存不会自动失效。如 RUN apt-get dist-upgrade -y 这样的RUN命令的缓存会被后续的构建过程复用。可通过指定--no-cache标志来使RUN命令的缓存失效，如：docker build --no-cache。

#### CMD
CMD命令有三种形式：
```javascript
CMD ["executable","param1","param2"] (exec form, this is the preferred form)
CMD ["param1","param2"] (as default parameters to ENTRYPOINT)
CMD command param1 param2 (shell form)
```
**一个Dockerfile中只能出现一个CMD命令。**如果一个Dockerfile文件中有多个CMD，只有最后那个CMD命令会生效。

CMD命令的**主要目的**是为运行的容器提供默认值。默认值可以包含可执行文件，或者也可以忽略可执行文件，这种情况下你必须同时声明一个ENTRYPOINT指令。这么看的话，上面第一种形式确实应该优先使用，它为运行的容器提供了默认的可执行文件以及默认参数值。

注意：
1. 如果CMD被用于为ENTRYPOINT命令提供默认参数，那么CMD和ENTRYPOINT两者都要被声明为JSON数组的格式。
2. exec形式（上面第一个）会被当做JSON数组解析，意味着你必须为每个单词使用双引号，而不是单引号。
3. 与RUN类似，exec形式不会提供shell处理逻辑。

当以exec形式或者shell形式使用CMD命令时，它会设置**容器运行镜像**时所执行的命令。
如果使用如下CMD命令的shell形式，\<command\>会这样执行：/bin/sh -c \<command\>:
```javascript
FROM ubuntu
CMD echo "This is a test." | wc -
```
如果你想在非shell环境中直接运行你的\<command\>，那你必须以JSON数组的形式表示，并且制定可执行文件的全路径。**这种JSON数组形式是CMD命令的第一选择。** 任何额外的（除可执行文件路径之外的）参数必须单独出现在JSON数组中：
```javascript
FROM ubuntu
CMD ["/usr/bin/wc","--help"]
```

如果你想要你的容器每次运行同一个可执行文件，那你应该组合使用CMD命令和ENTRYPOINT命令。

如果用户给docker run指定了参数，那么这个参数会**覆盖**构建镜像时用CMD命令设置的默认值。

注意：不要混淆了RUN和CMD。RUN会实际运行一个命令并且提交结果；CMD并不在构建过程中运行任何东西，它只是给镜像设置默认的执行程序。

#### LABEL
```javascript
LABEL <key>=<value> <key>=<value> <key>=<value> ...
```
LABEL命令给镜像增加元数据（metadata）。一个LABEL就是一系列的key-value。为了在一个LABEL值中包含空白字符，需要使用双引号和反斜杠。一些有用的示例如下：
```javascript
LABEL "com.example.vendor"="ACME Incorporated"
LABEL com.example.label-with-value="foo"
LABEL version="1.0"
LABEL description="This text illustrates \
that label-values can span multiple lines."
```
一个镜像可以有多个label。Docker建议，只要可能，就把所有labels组合到一个LABEL命令中。每条LABEL命令会产生一个新的层，这意味着使用太多的LABEL命令会导致镜像不够高效。
```javascript
LABEL multi.label1="value1" multi.label2="value2" other="value3"
```
上面的命令也可以写成这样：
```javascript
LABEL multi.label1="value1" \
      multi.label2="value2" \
      other="value3"
```
Labels会累积，包括FROM指定的基础镜像中的LABEL(s)：如果Docker解析时遇到一个已经存在的label key，新的label value会覆盖之前定义的具有相同key的label value。

使用 docker inspect 命令可以查看镜像的label：
```javascript
"Labels": {
    "com.example.vendor": "ACME Incorporated"
    "com.example.label-with-value": "foo",
    "version": "1.0",
    "description": "This text illustrates that label-values can span multiple lines.",
    "multi.label1": "value1",
    "multi.label2": "value2",
    "other": "value3"
},
```

#### MAINTAINER (deprecated)
MAINTAINER命令设置构建的镜像的Author字段。LABEL命令也能达到这个目的，并且更加灵活。LABEL可以设置任何你需要的metadata，并且方便查看（docker inspect）。设置一个对应MAINTAINER字段的label，可以这样：
```javascript
LABEL maintainer="SvenDowideit@home.org.au"
```
这个信息可通过 docker inspect 与其他label一起查看到。

#### EXPOSE
```javascript
EXPOSE <port> [<port>/<protocol>...]
```
EXPOSE命令通知Docker：容器运行时会监听指定的网络端口。你可以指定监听的是TCP端口或者UDP端口，默认是TCP。

EXPOSE指令不会真正地打开端口。它的作用就像是一种在镜像的构建者和容器的运行者之间的文档：说明哪些端口计划要被占用。要真正地在运行容器时打开端口，可以在 docker run 时使用-p标志来打开以及映射一个或多个端口。也可以使用-P标志。



#### ENV
```javascript
ENV <key> <value>
ENV <key>=<value> ...
```
ENV命令把环境变量key的值设置为value。
下面两种方式的效果是一样的：
```javascript
ENV myName="John Doe" myDog=Rex\ The\ Dog \
    myCat=fluffy
```
```javascript
ENV myName John Doe
ENV myDog Rex The Dog
ENV myCat fluffy
```
上面那种方式更好，因为它只产生一个缓存的layer。

使用ENV命令设置的环境变量，在从构建得到的镜像中运行容器时依然存在。你可以使用docker inspect查看环境变量的值，还能通过docker run --env key=value来改变环境变量的值。

注意：环境变量的持续性（persistence）可能导致一些意外的效应（因为它们在容器运行时依然有效）。另外，给一条单独的命令设置环境变量，可以简单地：
RUN key=value \<command\>

#### ADD
```javascript
ADD <src>... <dest>
ADD ["<src>",... "<dest>"] (this form is required for paths containing whitespace)
```
ADD命令把新的文件、目录或者远程URLs从src**拷贝**到镜像文件系统的dest路径中。可以同时指定多个src资源，如果src是文件或者目录，那它们必须是相对构建context根目录的。

每个src可以包含通配符，匹配时遵循Go的filepath.Match的规则。比如：
```javascript
ADD hom* /mydir/        # adds all files starting with "hom"
ADD hom?.txt /mydir/    # ? is replaced with any single character, e.g., "home.txt"
```

当文件或者目录包含特殊字符（如 [ 和 ] ）时，你需要根据Golang的规则来转义路径；否则它们会被当做模式的一部分。例如，添加一个文件arr[0].txt，
```javascript
ADD arr[[]0].txt /mydir/    # copy a file named "arr[0].txt" to /mydir/
```


所有的文件和目录创建时，使用0作为UID和GID（即超级用户和超级用户组）。

注意：
1. 如果你构建镜像时，是通过STDIN指定的Dockerf（ docker build - < somefile ），那么构建context是不存在的，这种情况下，Dockerfile里面只能使用基于URL的ADD命令。（这里指的是通过标注输入提供Dockerfile文件内容，而不是通过命令行指定Dockerfile文件路径）
2. 如果你的URLs需要授权，那你需要使用 RUN wget, RUN curl等等，ADD命令不支持鉴权。
3. 当src的内容发生改变时，Dockerfile中的第一条ADD命令会使接下来的所有命令（包括RUN）的缓存都失效。

**ADD命令遵循以下规则：**
1. src必须在build context内：不能 ADD ../something /something，因为 docker build 第一步是把context目录（以及子目录，递归的）发送到docker daemon。（真正的构建过程是在 docker daemon 中执行的，显然，../something是没意义的）
1. 如果src是一个URL并且dest不以'/'结尾，那么URL被当作文件下载并拷贝到dest。
1. 如果src是一个URL并且dest以'/'结尾，那么docker将从URL中解析出文件名filename，并且下载URL对应的文件到dest/filename。比如， http://example.com/foobar / 这条ADD命令将创建文件 /foobar。这种情况下，URL路径必须能被解析出正确的文件名（如http://example.com这样的URL就是无效的）。
1. 如果src是一个目录，那么此目录中的所有数据都会被拷贝，包括文件系统元数据。**注意：src目录本身不会被拷贝，只是拷贝其内部的数据。**
1. 如果src是一个压缩格式可被识别（gzip, bzip2 or xz）的本地tar文档，那么src将被解压为一个目录。但是URLs指定的远程资源不会被解压。当拷贝或者解压出一个目录时，与 tar -x 有同样的行为（具体看原文）。
1. 如果src是任何其他类型文件，src文件本身以及其元数据都会被拷贝。这种情况下，如果dest以'/'结尾，docker会将dest视为目录并且把src的内容拷贝到dest/base(src)。
1. 如果通过一个目录或者使用通配符作为src来指定多重资源，那么dest必须是目录，并且必须以'/'结尾。
1. 如果dest不以'/'结尾，那么docker将dest视为常规文件，并且把src的内容写入dest。
1. 如果dest不存在，则docker会创建dest，包括其路径中的各级目录。

#### COPY
```javascript
COPY <src>... <dest>
COPY ["<src>",... "<dest>"] (this form is required for paths containing whitespace)
```
COPY命令将src中的文件或目录拷贝至容器的文件系统的dest路径中。

可以在src中同时指定多个资源，但是这些资源必须相对构建context的根目录。 每个src都可以包含通配符，docker根据Go的filepath.Match的规则进行匹配。例如：
```javascript
COPY hom* /mydir/        # adds all files starting with "hom"
COPY hom?.txt /mydir/    # ? is replaced with any single character, e.g., "home.txt"
```

dest是一个绝对路径，或者相对于WORKDIR的路径。src的内容将被拷贝到目标容器的这个路径内：
```javascript
COPY test relativeDir/   # adds "test" to `WORKDIR`/relativeDir/
COPY test /absoluteDir/  # adds "test" to /absoluteDir/
```

当拷贝的src文件或目录的路径中包含特殊字符（如 [ 或者 ] ），处理方式与ADD命令相同。

同样，所有的文件和目录创建时都使用0作为UID和GID（超级用户和超级用户组）。

COPY命令接收可选的 --from=\<name|index\> 标志，该标志用于设置资源路径为此前的build阶段（build stage，由 FROM .. AS \<name\> 创建），用户指定的构建context则被忽略。

**COPY遵循与ADD类似的规则。**但是，COPY不能作用于URL，也不能解压tar文档。详情参看原文。

##### 补充：ADD 与 COPY
COPY与ADD的区别是：ADD支持获取远端URL资源，并且解压tar资源。通常从构建context中复制资源（文件、目录）推荐使用COPY（个人认为这个命令名称更符合语义）。ADD命令擅长获取URL资源以及解压tar文档。


#### ENTRYPOINT
```javascript
ENTRYPOINT ["executable", "param1", "param2"] (exec form, preferred)
ENTRYPOINT command param1 param2 (shell form)
```

ENTRYPOINT 命令允许你配置容器以便作为可执行文件运行。例如，如下的命令将nginx作为默认入口启动，并监听80端口：
```javascript
docker run -i -t --rm -p 80:80 nginx
```
docker run \<image\> 的命令行参数会被追加在ENTRYPOINT命令的exec形式指定的元素之后，并且覆盖CMD命令指定的所有元素。这允许你传递参数给容器的入口程序。如 docker run \<image\> -d 会把 -d 参数传递给入口程序（entry point）。你可以通过 docker run --entrypoint 标志重写ENTRYPOINT命令。

shell形式的ENTRYPOINT命令会屏蔽所有CMD命令和 docker run 指定的参数。并且这种形式还有一个劣势：你的ENTRYPOINT将被作为shell的子程序启动（/bin/sh -c），而这种启动方式不会传递信号。这意味着你指定的可执行程序不会被作为容器的 PID 1 进程，并且不会接收Unix信号：你的可执行程序不会接收docker stop \<container\> 发送的 SIGTERM。

**因此推荐使用ENTRYPOINT的exec形式。**

Dockerfile文件中，只有最后一个ENTRYPOINT命令是有效的。

##### ENTRYPOINT：Exec form ENTRYPOINT example
你可以使用ENTRYPOINT命令的exec形式来设置相当稳定的默认程序和参数。还能再用CMD命令的任一形式来设置额外的更可能变化的默认值。
```javascript
FROM ubuntu
ENTRYPOINT ["top", "-b"]
CMD ["-c"]
```
当你运行容器时，你可以看到只有一个top进程：
```javascript
$ docker run -it --rm --name test  top -H
top - 08:25:00 up  7:27,  0 users,  load average: 0.00, 0.01, 0.05
Threads:   1 total,   1 running,   0 sleeping,   0 stopped,   0 zombie
%Cpu(s):  0.1 us,  0.1 sy,  0.0 ni, 99.7 id,  0.0 wa,  0.0 hi,  0.0 si,  0.0 st
KiB Mem:   2056668 total,  1616832 used,   439836 free,    99352 buffers
KiB Swap:  1441840 total,        0 used,  1441840 free.  1324440 cached Mem

  PID USER      PR  NI    VIRT    RES    SHR S %CPU %MEM     TIME+ COMMAND
    1 root      20   0   19744   2336   2080 R  0.0  0.1   0:00.04 top
```

你还可以用 docker exec（在运行的容器中执行命令）来更深入检查结果：
```javascript
$ docker exec -it test ps aux
USER       PID %CPU %MEM    VSZ   RSS TTY      STAT START   TIME COMMAND
root         1  2.6  0.1  19752  2352 ?        Ss+  08:24   0:00 top -b -H
root         7  0.0  0.1  15572  2164 ?        R+   08:25   0:00 ps aux
```
你还能使用 docker stop test 来优雅地停止top程序。

下面的Dockerfile显示了如何使用ENTRYPOINT命令来在前台运行（即作为PID 1 ）Apache：
```javascript
FROM debian:stable
RUN apt-get update && apt-get install -y --force-yes apache2
EXPOSE 80 443
VOLUME ["/var/www", "/var/log/apache2", "/etc/apache2"]
ENTRYPOINT ["/usr/sbin/apache2ctl", "-D", "FOREGROUND"]
```

如果你要为一个可执行程序编写启动脚本，你可以使用 exec 以及 gosu 命令来确保最终的可执行程序能接收到Unix信号：
```javascript
#!/usr/bin/env bash
set -e

if [ "$1" = 'postgres' ]; then
    chown -R postgres "$PGDATA"

    if [ -z "$(ls -A "$PGDATA")" ]; then
        gosu postgres initdb
    fi

    exec gosu postgres "$@"
fi

exec "$@"
```

最后，如果你想在（容器）关闭时做一些清理工作（或者与其他容器交互），或者协作多个可执行程序，你可能需要确保ENTRYPOINT脚本能接收到Unix信号、继续传递信号进而做些其他工作：
```javascript
#!/bin/sh
# Note: I've written this using sh so it works in the busybox container too

# USE the trap if you need to also do manual cleanup after the service is stopped,
#     or need to start multiple services in the one container
trap "echo TRAPed signal" HUP INT QUIT TERM

# start service in background here
/usr/sbin/apachectl start

echo "[hit enter key to exit] or run 'docker stop <container>'"
read

# stop service and clean up here
echo "stopping apache"
/usr/sbin/apachectl stop

echo "exited $0"
```

如果你通过 docker run -it --rm -p 80:80 --name test apache 来运行这个镜像，就能通过 docker exec 或者 docker top 来检查容器的进程列表，还能让脚本来停止Apache：
```javascript
$ docker exec -it test ps aux
USER       PID %CPU %MEM    VSZ   RSS TTY      STAT START   TIME COMMAND
root         1  0.1  0.0   4448   692 ?        Ss+  00:42   0:00 /bin/sh /run.sh 123 cmd cmd2
root        19  0.0  0.2  71304  4440 ?        Ss   00:42   0:00 /usr/sbin/apache2 -k start
www-data    20  0.2  0.2 360468  6004 ?        Sl   00:42   0:00 /usr/sbin/apache2 -k start
www-data    21  0.2  0.2 360468  6000 ?        Sl   00:42   0:00 /usr/sbin/apache2 -k start
root        81  0.0  0.1  15572  2140 ?        R+   00:44   0:00 ps aux
$ docker top test
PID                 USER                COMMAND
10035               root                {run.sh} /bin/sh /run.sh 123 cmd cmd2
10054               root                /usr/sbin/apache2 -k start
10055               33                  /usr/sbin/apache2 -k start
10056               33                  /usr/sbin/apache2 -k start
$ /usr/bin/time docker stop test
test
real	0m 0.27s
user	0m 0.03s
sys	0m 0.03s
```

注意：
1. 你可以通过 --entrypoint 来重写 ENTRYPOINT 设置，但这只能设置exec形式的二进制程序（没有 sh -c）。
1. exec形式也会被当作JSON数组解析，因此每个单词得使用双引号。
1. exec形式不会启动shell。具体参考上面RUN命令。

##### ENTRYPOINT：Shell form ENTRYPOINT example
你可以给ENTRYPOINT指定一个字符串，该字符串将被以 /bin/sh -c 的形式执行。这种形式将使用shell处理来替代shell环境变量，并且忽略CMD命令以及 docker run 的命令行参数。为了确保 docker stop 能正确地给长时间运行的ENTRYPOINT可执行程序发送信号，你得记得用exec来启动它：
```javascript
FROM ubuntu
ENTRYPOINT exec top -b
```
当你运行这个镜像时，你将看到只有一个PID为1的进程：
```javascript
$ docker run -it --rm --name test top
Mem: 1704520K used, 352148K free, 0K shrd, 0K buff, 140368121167873K cached
CPU:   5% usr   0% sys   0% nic  94% idle   0% io   0% irq   0% sirq
Load average: 0.08 0.03 0.05 2/98 6
  PID  PPID USER     STAT   VSZ %VSZ %CPU COMMAND
    1     0 root     R     3164   0%   0% top -b
```
并且，在 docker stop 时这个进程能够干净地退出：
```javascript
$ /usr/bin/time docker stop test
test
real	0m 0.20s
user	0m 0.02s
sys	0m 0.04s
```

如果你忘了在ENTRYPOINT的开头添加 exec 的话：
```javascript
FROM ubuntu
ENTRYPOINT top -b
CMD --ignored-param1
```
接下来你可以运行它：
```javascript
$ docker run -it --name test top --ignored-param2
Mem: 1704184K used, 352484K free, 0K shrd, 0K buff, 140621524238337K cached
CPU:   9% usr   2% sys   0% nic  88% idle   0% io   0% irq   0% sirq
Load average: 0.01 0.02 0.05 2/101 7
  PID  PPID USER     STAT   VSZ %VSZ %CPU COMMAND
    1     0 root     S     3168   0%   0% /bin/sh -c top -b cmd cmd2
    7     1 root     R     3164   0%   0% top -b
```
你可以看到，ENTRYPOINT指定的top进程的PID不是1（PID 1 进程是先启动的shell）。

然后，你可以执行 docker stop test，就会发现，容器并没有干净地退出。stop命令不得不在超时后发送一个SIGKILL：
```javascript
$ docker exec -it test ps aux
PID   USER     COMMAND
    1 root     /bin/sh -c top -b cmd cmd2
    7 root     top -b
    8 root     ps aux
$ /usr/bin/time docker stop test
test
real	0m 10.19s
user	0m 0.04s
sys	0m 0.03s
```

##### Understand how CMD and ENTRYPOINT interact
CMD和ENTRYPOINT命令都能指定在容器运行时将运行什么程序。这里是几条描述CMD和ENTRYPOINT协作的规则：
1. Dockerfile应该至少通过CMD或ENTRYPOINT声明一个可执行程序。
1. 当把容器作为可执行程序时，要声明ENTRYPOINT命令。
1. CMD应该被用来给ENTRYPOINT命令指定的可执行程序提供默认参数。
1. 当运行容器并指定参数时，CMD提供的默认参数将被覆盖。

下面的表格展示了不同的ENTRYPOINT / CMD组合时，将执行什么可执行程序：
![ENTRYPOINT-CMD.png](https://raw.githubusercontent.com/ttyrion/ttyrion.github.io/master/image/docker/ENTRYPOINT-CMD.png) 


#### VOLUME
```javascript
VOLUME ["/data"]
```
VOLUME命令以指定的name创建一个挂载点，并且标记为持有来自本地宿主机或者其他容器的外部挂载卷。其值可以是JSON数组，如VOLUME ["/var/log/"]；或者只是带多个参数的字符串形式，如 VOLUME /var/log 或者 VOLUME /var/log /var/db。更多的信息请参考[Share Directories via Volumes](https://docs.docker.com/storage/volumes/)。

**docker run会用基础镜像中指定的路径中的数据来初始化（容器中）新创建的卷。** 例如，考虑下面这一小段Dockerfile：
```javascript
FROM ubuntu
RUN mkdir /myvol
RUN echo "hello world" > /myvol/greeting
VOLUME /myvol
```
这个Dockerfile文件将生成一个镜像，这个镜像将使 docker run 在容器的/myvol处创建一个挂载点，并且把镜像文件系统中的greeting文件拷贝到这个新创建的卷中。

##### VOLUME：Notes about specifying volumes
关于Dockerfile中的卷（volumes），需要记住以下内容：
1. 基于Windows的容器...(正式生产环境不会用，略过)
1. 在声明了volume之后的构建步骤中对volume内数据所做的修改，都会被丢弃。
1. volume列表被当做JSON数组解析，所以也要使用双引号。
1. 宿主机目录（挂载点）本质上是与宿主机相关的。这是为了确保镜像的可移植性。然而，不可能保证一个宿主目录在所有的宿主机上都有效。出于这个原因，你不能在Dockerfile中挂载宿主机的目录（因为运行构建生成的镜像的宿主机，很可能就没有这个目录，甚至宿主系统都不一样）。VOLUME命令不支持指定一个“宿主目录”参数：你必须在创建或者运行容器的时候指定挂载点。

比如：
```javascript
docker run -it --name test -v /src/data:/myvol python app.py
```
上面就在运行容器时把宿主机（执行docker run的机子）的/src/data挂载到容器的/myvol卷。容器中操作/myvol就相当于操作宿主机的/src/data，这就实现了数据的持久化：就算容器停止甚至删除，数据也依然存在。


#### USER
```javascript
USER <user>[:<group>] or
USER <UID>[:<GID>]
```
USER命令设置运行镜像以及Dockerfile中所有在USER命令之后的RUN, CMD 和 ENTRYPOINT命令时使用的用户名（或用户ID）和可选的用户组（或组ID）。

注意：当指定的用户不属于用户组时，镜像（以及接下来的命令）将被以root组权限运行。

#### WORKDIR
```javascript
WORKDIR /path/to/workdir
```
WORKDIR命令给（Dockerfile中）随后的RUN, CMD, ENTRYPOINT, COPY 和 ADD 命令设置工作目录。如果WORKDIR不存在，就会被创建出来，尽管可能后面的命令并不使用它。

一个Dockerfile中可以使用多个WORKDIR命令。如果命令中使用相对路径，则这个路径将相对于上一个WORKDIR命令。