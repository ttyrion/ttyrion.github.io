---
layout:         page
title:          【Spring Boot】Spring Boot Multi-modules Project
date:           2020-07-31
author:         翼
header-img: image/bg.jpg
catalog: true
tags:
---

## Maven Multi-Modules Project
一个多模块的项目，其实就是由一个parent pom管理着一组子模块（每个子模块有自己的pom）。parent pom位于项目的根目录，并且：parent pom 的 packaging（Maven坐标之外的几个基础要素之一）必须是“pom”。而子模块是常规的Maven项目，其 packaging 类型的值可以是jar, war, ear等。我们可以选择基于单个子模块的pom文件或者parent pom 文件来执行 Maven build任务。如果是在parent pom上运行Maven build，那所有的子模块都会被构建。

### Create a Spring Boot Multi-Module Maven Project
#### 1. Create a parent project
由于parent项目只是负责通过pom管理各个子模块，其本身并不包含什么代码，所以parent项目其实只需包含一个pom.xml文件即可。我们依然通过Spring Initializr来创建一个Spring Boot 项目，只不过，项目类型选择“Maven POM”。此时，创建的新项目就只会包含一个pom.xml，没有src代码目录、mvnw命令等等。如图：

![maven_pom](https://github.com/ttyrion/Java/blob/master/doc/img/spring_boot/maven_pom.png)

注意，接下来必须设置parent pom的packaging类型为pom：
```javascript
<packaging>pom</packaging>

```

#### 2. Create child modules
接下来是为parent project创建各个子模块。在parent项目上右键选择New，来新建各个Module即可。值得注意的是，子模块的项目类型都是“Maven Project”，而不是“Maven POM”。创建完成的项目组织结构如图：

![maven_parent_project](https://github.com/ttyrion/Java/blob/master/doc/img/spring_boot/maven_parent_project.png)

创建完各个子模块之后，接下来就需要编辑相关的pom了。

#### 3. 修改pom配置

##### 3.1 给 parent pom 添加各个子module
如：
```javascript
<modules>
	<module>mm-application</module>
	<module>mm-controller</module>
	<module>mm-domain</module>
</modules>

```

##### 3.2 修改 child pom 的 parent pom
新创建的子模块pom中的parent应该还是Spring Boot，我们必须将各个子模块的 parent 设置为parent（root）项目。如：
```javascript
<parent>
	<groupId>com.example</groupId>
	<artifactId>demo-multi-modules</artifactId>
	<version>0.0.1-SNAPSHOT</version>
	<relativePath/> <!-- lookup parent from repository -->
</parent>
```

##### 3.3 修改各个子模块的依赖
每个子模块除了公共的依赖（置于parent pom 中），也会有自己独立的依赖项。例如这里的 mm-application 模块会依赖 mm-controller，而 mm-controller 会依赖 mm-domain。
如下面是mm-application的pom，这里添加了一个对mm-controller的依赖项：
```javascript
// mm-application/pom.xml
<dependencies>
	<dependency>
		<groupId>org.springframework.boot</groupId>
		<artifactId>spring-boot-starter-webflux</artifactId>
	</dependency>

	<dependency>
		<groupId>io.projectreactor</groupId>
		<artifactId>reactor-test</artifactId>
		<scope>test</scope>
	</dependency>

	<dependency>
		<groupId>com.example</groupId>
		<artifactId>mm-controller</artifactId>
		<version>0.0.1-SNAPSHOT</version>
	</dependency>
</dependencies>

```

下面这个则是 mm-controller 的pom：
```javascript
// mm-controller/pom.xml
<dependencies>
	<dependency>
		<groupId>com.example</groupId>
		<artifactId>mm-domain</artifactId>
		<version>0.0.1-SNAPSHOT</version>
	</dependency>
</dependencies>

```

##### 3.4 删除冗余pom配置
parent pom的一些配置，如properties、dependencies是会被子模块继承过去的，因此我们可以把子模块重复的pom配置删掉。还有各个子module的**parent.relativePath**(it's the relative path from the module's pom.xml to the parent's pom.xml)配置，它的默认值是../pom.xml，我们这里的工程结构刚好就是默认的那种目录组织结构，因此不需要配置这个值，直接删掉子模块的parent.relativePath。

下面是一段来自maven官网的对[relativePath](http://maven.apache.org/ref/3.3.9/maven-model/maven.html#parent)的介绍：
```javascript
The relative path of the parent pom.xml file within the check out. If not specified, 
it defaults to ../pom.xml. 

Maven looks for the parent POM first in this location on the filesystem, then the local repository, 
and lastly in the remote repo. relativePath allows you to select a different location, 
for example when your structure is flat, or deeper without an intermediate parent POM. 
However, the group ID, artifact ID and version are still required, 
and must match the file in the location given or it will revert to the repository for the POM. 

This feature is only for enhancing the development in a local checkout of that project. 
Set the value to an empty string in case you want to disable the feature and always resolve 
the parent POM from the repositories.
Default value is: ../pom.xml.

```
从上面的介绍可知，parent.relativePath提供了一种在开发环境下，让maven从项目目录中找到parent pom的方式。如果不删掉parent.relativePath，而是保留默认生成的值（即空值\<relativePath/\>），则maven只会从仓库中找这个parent pom（先本地仓库，再远程仓库）。

另外还需要删除的是默认生成的build配置：
```javascript
<build>
	<plugins>
		<plugin>
			<groupId>org.springframework.boot</groupId>
			<artifactId>spring-boot-maven-plugin</artifactId>
		</plugin>
	</plugins>
</build>

```
因为默认情况下，这个spring-boot-maven-plugin插件的**spring-boot:repackage**目标（goal）会被执行。下面是spring-boot官方文档对[repackage](https://docs.spring.io/spring-boot/docs/2.3.0.RELEASE/maven-plugin/reference/html/#goals)的介绍：
```javascript
Repackage existing JAR and WAR archives so that they can be executed from the command line 
using java -jar. 

With layout=NONE can also be used simply to package a JAR with nested dependencies (and no main class, so not executable).
```
从上面的描述可知：如果build里面配置了插件spring-boot-maven-plugin，那么它的repackage目标就会把maven build生成的JAR包或WAR包重新打包一次，生成可直接运行的JAR包（java -jar）。

而我们这里的mm-domain和mm-controller这两个子模块，只是想充当mm-application主程序的依赖包，而不是直接作为可执行JAR包存在于项目中。因此需要禁止mm-domain和mm-controller这两个包被spring-boot-maven-plugin插件**repackage**。另外，又因为pom配置可被继承，因此只是删除子模块的build配置不够，还要删掉parent pom的build，因为这里也配置了spring-boot-maven-plugin插件。
