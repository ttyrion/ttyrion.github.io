---
layout:         page
title:         【Go】标准的ServeMux和"github.com/gorilla/mux"
subtitle:       
date:           2019-05-21
author:         翼
header-img: image/bg.jpg
catalog: true
tags:
---

**来自golang.org的对 ServeMux 的介绍：** 标准库net/http包的ServeMux，是一个HTTP请求多路复用器（HTTP request multiplexer，也是mux一词的由来）。ServeMux将请求的URL与一个已注册的模式进行匹配，并且调用与该URL最佳匹配的模式对应的处理器函数。

**来自godoc.org的对 package mux 的介绍：** 第三方的mux包（"github.com/gorilla/mux"）实现了请求的路由和分发。

不论是从上面的介绍，还是从名字来看（mux是HTTP request multiplexer的简称），都可以看出这两者的作用基本一样。

### ServeMux
#### 模式（Patterns）
模式描述了**固定的根路径**，比如"/images"；或者**带根的子树**，比如"/images/"。**"/images"和"/images/"是两个不同的模式，** 后面分别简称为路径模式和子树模式。

1. 长的模式的匹配优先级更高。比如说，我们注册了"/images/" 和 "/images/thumbnails/"两个模式，那么当请求的URL以 "/images/thumbnails/" 开头时，多路复用器会调用后者对应的处理器函数。
1. 模式结尾的"/"字符，表示这个模式描述的是带根的子树，也就是说这个模式会匹配所以包含这个根子树的URL路径。所以，模式"/"，匹配所有没匹配上其他模式的路径，而不只是路径为"/"的URL。
1. 如果注册了一个子树模式，并且接收到一个不带子树的结尾"/"字符的请求（即路径模式的请求），ServeMux会把这个请求重定向到子树（添加一个"/"）。这一行为也可以被重写：只要再注册一个路径模式（不要结尾的"/"字符）。比如说，注册了"/images/"模式的话，ServeMux会把请求"/images"重定向到"/images/"。但是如果同时也注册了一个模式"/images"，那么重定向的行为就不会发生。
1. 模式也能以主机名开头，这种特定主机的模式的匹配优先级比普通模式更高。

##### 模式：固定的根路径和带根的子树
上面已经说过这两个模式是不同的，接下来上代码。
```javascript
// package handler
func HandleDefault(w http.ResponseWriter, r *http.Request) {
    w.Write([]byte("OK"))
}

// package main
mux := http.NewServeMux()
mux.HandleFunc("/images", handler.HandleDefault)
err := http.ListenAndServe("localhost:8080", mux)
if err != nil {
    fmt.Println("start service faild.", err)
    return
}

```
上面通过一个带根的子树模式注册了URL。请求 http://localhost:8080/images 和 http://localhost:8080/images/ 的结果如下：

![/images](https://raw.githubusercontent.com/ttyrion/ttyrion.github.io/master/image/go/http/images1.png)

![/images/](https://raw.githubusercontent.com/ttyrion/ttyrion.github.io/master/image/go/http/images2.png)

上面的请求结果表明："/images/" 和 "/images" 是两个不同的模式。"/images"称为固定根路径的模式，即这个模式匹配的是路径。而"/images/"称为带根的子树模式，即这个模式会匹配这个子树包含的所有路径。

因此，如果把上面代码中的模式"/images"改为"/images/"，那么ServeMux就会把 http://localhost:8080/images/ 和 http://localhost:8080/images 这两个请求都交给handler.HandleDefault处理。甚至，请求http://localhost:8080/images/a/b/c/d 也会交给handler.HandleDefault处理，因为它们都属于模式子树"/images/"内部的路径，即都能匹配成功。

**这就是子树和路径两种模式的区别。**

### package mux
"mux"是"HTTP request multiplexer"的简写。mux.Router的作用与标准的http.ServeMux类似，但是它有自己的特性。
#### mux.Router的特性
##### 1. 灵活的请求URL路径匹配方式
###### 1.1 普通的路径模式注册URL
package mux也支持标准的http.ServeMux所支持的注册模式，但有些不同。
```javascript

func HandleProduct(w http.ResponseWriter, r *http.Request) {
    w.Write([]byte("OK"))
}

func main() {
    router := mux.NewRouter()
    router.HandleFunc("/product/", HandleProduct)
    server := "localhost:8080"
    fmt.Println("starting service on", server)
    err := http.ListenAndServe(server, router)
    if err != nil {
        fmt.Println("start service faild.", err)
        return
    }
}

```
上面通过一个带根的子树模式注册了URL。请求 http://localhost:8080/images 和 http://localhost:8080/images/ 的结果如下：

![/images](https://raw.githubusercontent.com/ttyrion/ttyrion.github.io/master/image/go/http/images3.png)

![/images/](https://raw.githubusercontent.com/ttyrion/ttyrion.github.io/master/image/go/http/images4.png)

上面的结果表明了package mux 和标准的http.ServeMux的不同。即，gorilla mux 注册URL时，不论是带根的路径模式，还是带根的子树模式，都会进行完全匹配。比如，如果通过mux.Router注册了一个请求URL，模式串是"/product/"。我们可以通过 http://localhost:8080/product/ 正常请求资源，但是通过 http://localhost:8080/product 或者 http://localhost:8080/product/a 都不行，服务都会返回“404 page not found”。

**这是gorilla mux 与 http.ServeMux 在使用基本模式注册URL时的不同之处。**

mux还能设置其他的匹配规则，比如路径前缀， HTTP方法， URL schemes等：
r.PathPrefix("/products/")
r.Methods("GET", "POST")
r.Schemes("https")
r.Headers("X-Requested-With", "XMLHttpRequest")

##### 2. 请求URL的host, path, query values 可以包含变量以及正则表达式
```javascript

func HandleProduct(w http.ResponseWriter, r *http.Request) {
    vars := mux.Vars(r)
    w.Write([]byte("OK " + vars["name"] + " " + vars["size"]))
}

func HandleNews(w http.ResponseWriter, r *http.Request) {
    vars := mux.Vars(r)
    w.Write([]byte("OK " + vars["title"]))
}

func main() {
    router := mux.NewRouter()
    router.HandleFunc("/product/{name}", HandleProduct)
    router.HandleFunc("/product/{name}/{size:[0-9]+}", HandleProduct)
    router.HandleFunc("/news/{title}", HandleNews).Host("{subdomain:[a-z]+}.test.domain:8080")
    server := "localhost:8080"
    fmt.Println("starting service on", server)
    err := http.ListenAndServe(server, router)
    if err != nil {
        fmt.Println("start service faild.", err)
        return
    }
}

```
上面的代码示例，在路径和host中加了变量或者正则表达式。需要注意的有以下几点：
1. 注册URL为"/product/{name}"时，依然会有“完全匹配”的规则。即，我们可以访问到 http://localhost:8080/product/chairs 并且变量name的值为chairs。但是，访问 http://localhost:8080/product/chairs/ 会返回 404 page not found
2. 域名限制需要包含端口号（如果端口号不是80）。

##### 3. 可以构建注册的URL ？
这个貌似不常用，以后用到再说。

##### 4. 子Routes
```javascript

func main() {
    router := mux.NewRouter()
    sub := router.PathPrefix("/product").Subrouter()
    sub.HandleFunc("/{name}", HandleProduct)
    sub.HandleFunc("/{name}/{size:[0-9]+}", HandleProduct)
    server := "localhost:8080"
    fmt.Println("starting service on", server)
    err := http.ListenAndServe(server, router)
    if err != nil {
        fmt.Println("start service faild.", err)
        return
    }
}

```
这里注册的两个URL "/{name}" 和 "/{name}/{size:[0-9]+}"，只有在满足PathPrefix("/product")的前提下才会被测试是否匹配。

##### 5. 实现了 http.Handler 接口
上面的示例都是直接把router传给http.ListenAndServe，因为 mux.Router也是一个http.Handler接口。这使得gorilla mux用起来非常方便。