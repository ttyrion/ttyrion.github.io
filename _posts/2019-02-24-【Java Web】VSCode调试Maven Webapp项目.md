---
layout:         page
title:         【Java Web】VSCode调试Maven Webapp项目
subtitle:       
date:           2019-02-24
author:         翼
header-img: image/bg.jpg
catalog: true
tags:
---

> 摸索着使用VSCode来开发Java Web项目，花了大半天终于可以调试war包了，记录一下过程。

#### Maven Web Archetype
Maven有个Web项目原型，可以据此生成我们的Web项目结构。创建Web项目结构的方法是：
![maven_project](https://raw.githubusercontent.com/ttyrion/Java/master/doc/img/web/maven_project.png) 

安装了maven插件之后，就可以借助maven生成各种项目原型。如上图所示，点击+号后，maven插件会列出所有可用的archetype，输入“web”过滤出我们需要的选项：
![maven_webapp](https://raw.githubusercontent.com/ttyrion/Java/master/doc/img/web/maven_webapp.png)

选项“maven-archetype-webapp”正是我们需要的web项目原型。选择一个archetype之后，maven会要求我们输入项目的maven坐标元素，如groupId、artifactId等等：

![maven_coordinate](https://raw.githubusercontent.com/ttyrion/Java/master/doc/img/web/maven_coordinates.png)

注意，这里我们要把package设置为“war”（Web Archive），因为web项目最终会被打包为war格式，而不是默认的jar。

创建完web项目之后，VSCode默认不会自动打开项目目录。手动选择File->Open Folder打开我们刚才创建的项目的根目录，就能看到maven创建的web项目结构如下：

![maven_project_structure](https://raw.githubusercontent.com/ttyrion/Java/master/doc/img/web/maven_project_structure.png)

在src/main目录下，包含了webapp目录，其内部又包含了WEB-INF目录。WEB-INF目录是一个很重要的目录，它最终会被maven打包到war包中，其内部也包含了web.xml（**最终打包后WEB-INF还会包含classes目录，classes目录包含java Servlet类**）。

这里还要提一下，项目结构下方的“**JAVA DEPENDENCIES**”项下面会列出当前项目“helloweb”的依赖项。如果我们在pom.xml中配置了某些依赖项，maven会从中央仓库中下载对应依赖项到本地仓库中（如果本地仓库没有的话），而这些依赖项就会被显示在“**JAVA DEPENDENCIES**”下的对应项目名称之下。如果没有显示，可以点击右侧的刷新按钮。

此时，还能看见此前添加maven web 项目的“**MAVEN 项目**”一栏下，已经多了一个项目，正是helloweb。**在项目名称“helloweb”上右键，可以执行项目的clean、compile、deploy等等maven生命周期。**

#### Maven Web 项目的手动部署
我们先给我们的web项目添加一个servlet。首先需要在pom.xml中声明我们的项目对javax.servlet的依赖：

```javascript
<dependency>
     <groupId>javax.servlet</groupId>
     <artifactId>javax.servlet-api</artifactId>
     <version>4.0.1</version>
     <scope>provided</scope>
</dependency>
```

其中，dependency的scope配置为provided，即表示： 此依赖项在编译和测试时有效 , 应行时不需要，因为Tomcat容器会提供。

然后，在main目录下新建一个java目录，用于存放我们的java代码（如果新建的目录没有显示，就刷新一下项目）。然后在java目录中新增一个RegisterServlet.java:

```java
import java.io.*;
import javax.servlet.*;
import javax.servlet.http.*;

public class RegisterServlet extends HttpServlet {
    @Override
    public void doGet(HttpServletRequest request, HttpServletResponse response)
         throws IOException, ServletException {
 
      // Set the response MIME type of the response message
      response.setContentType("text/html");
      // Allocate a output writer to write the response message into the network socket
      PrintWriter out = response.getWriter();
 
      // Write the response message, in an HTML page
      try {
         out.println("<html>");
         out.println("<head><title>Hello, World</title></head>");
         out.println("<body>");
         out.println("<h1>Hello, world!</h1>");  // says Hello
         // Echo client's request information
         out.println("<p>Request URI: " + request.getRequestURI() + "</p>");
         out.println("<p>Protocol: " + request.getProtocol() + "</p>");
         out.println("<p>PathInfo: " + request.getPathInfo() + "</p>");
         out.println("<p>Remote Address: " + request.getRemoteAddr() + "</p>");
         // Generate a random number upon each request
         out.println("<p>A Random Number: <strong>" + Math.random() + "</strong></p>");
         out.println("</body></html>");
      } finally {
         out.close();  // Always close the output writer
      }
   }
}
```

servlet创建完毕之后，就是配置servlet。在web.xml的“web-app”内，增加如下配置：

```javascript
  <servlet>
    <servlet-name>Register</servlet-name>
    <servlet-class>RegisterServlet</servlet-class>
  </servlet>

  <servlet-mapping>
    <servlet-name>Register</servlet-name>
    <url-pattern>/register</url-pattern>
  </servlet-mapping>
```
上面先配置了一个name为Register的servlet，servlet类为RegisterServlet。后又配置了由name为Register的servlet来处理对路径“/register”的访问。

至此，我们可以部署一下这个简单的web项目了。右键点击“**MAVEN 项目**”下面的“helloweb”项目，然后选择“package”子菜单，我们开始打包项目。打包结束后，我们可以看到该web项目被打包为target/helloweb.war。 **很明显，web项目中没有一个Java类有main方法，也就是说web项目本身是不可执行的。我们需要把项目的war包放到tomcat的webapps目录下，然后执行startup.bat启动Tomcat(容器服务器)。** 可以看到tomcat把helloweb.war包解压成webapps目录下的一个helloweb目录。而helloweb目录的结构正是一个web应用的结构，比如WEB-INF目录包含了一个classes目录，classes目录包含java类，如这里的注册servlet RegisterServlet.class。可见，是maven打包的过程中处理了这些，我们并没有手动给web项目添加classes等目录。

浏览器访问[http://localhost:8080/helloweb/register](http://localhost:8080/helloweb/register) 可以看到输出：

![maven_tomcat_server](https://raw.githubusercontent.com/ttyrion/Java/master/doc/img/web/maven_tomcat_server.png)

#### Web 项目的调试
调试web项目就需要借助“**Tomcat for Java**”插件。安装了此插件之后，VSCode EXPLORER下面就会多出一个“**TOMCAT SERVERS**”栏。右侧的“+”按钮点击后选择本地的Tomcat Home目录，即可添加一个Tomcat Server项：

![tomcat_plugin](https://raw.githubusercontent.com/ttyrion/Java/master/doc/img/web/tomcat_plugin.png)

右键点击某一Tomcat Server项，可以看到Tomcat插件可用的功能，包括“**Debug War Package**”，即调试Web项目包：

![tomcat_debug_war](https://raw.githubusercontent.com/ttyrion/Java/master/doc/img/web/tomcat_debug_war.png)

调试起war包之后，在浏览器中访问相应的地址，就可以调试Web项目，如Servlet等。Tomcat插件也会在Tomcat Server项下面记录我们调试的本地war包，此后我们调试该项目可直接右键点击该war包并选择“**Open In Browser**”即可，操作更便利。

