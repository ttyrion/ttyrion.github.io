---
layout:         page
title:          【Go】实现类似Flask Blueprint的功能
date:           2020-05-01
author:         翼
header-img: image/bg.jpg
catalog: true
tags:
---

后端服务都有一个路由的处理，注册URL。Flask里面有一个Blueprint类，可以辅助注册URL，使得每个业务模块的URL注册都只需在本模块之内处理，不依赖全局的Flask App对象。这使得各个模块很内敛、松耦合。

这里打算在Go里面实现一个类似功能的Blueprint，每个业务模块（也即代码层面上的每个package）都通过这个Blueprint注册URL，再在最上层业务中统一的地方注册这些Blueprint。

下面是Blueprint的定义：
```go
package blueprint

import (
	"net/http"
)

type RequestHandlerMethod func(http.ResponseWriter,	*http.Request)
type RequestHandler struct {
	Method RequestHandlerMethod
	// GET POST ...
	MethodType string
}
type RouteMap map[string]RequestHandler
type Blueprint struct {
	Routes RouteMap
}

func NewBlueprint() *Blueprint {
	return &Blueprint{
		Routes : RouteMap{},
	}
}

func (bp *Blueprint) RegisterRoute(urlReg string, method RequestHandlerMethod, methodType string){
	if bp == nil {
		return
	}

	bp.Routes[urlReg] = RequestHandler{
		Method: method,
		MethodType: methodType,
	}
}

```

如上面代码所示，Blueprint类型拥有一个记录URL和对应的处理函数的Map。

接下来要做的，就类似Flask对象的register_blueprint方法，我们也需要实现一个方法来将上面Blueprint的注册信息再注册到启动服务时的Router中。

通常情况下，启动web服务时的代码如下所示(这里忽略服务超时、服务panic恢复的处理)：
```go
r := mux.NewRouter()
r.HandleFunc("/", handler.GetDefault).Methods("GET")
err := http.ListenAndServe("localhost:8080", r)
if err != nil {
	...
}

```

因为我们现在打算把URL注册放到各个业务模块（包）内部，main方法里面只需要注册各个Blueprint即可。那问题就是，怎么给上面的*mux.Router增加一个注册Blueprint的方法，在此方法内部再实际注册URL。这里就要用到上一篇讲过的：Go的继承。

这里是一个自定义的RegisterRouter类型，它包含了一个匿名的\*mux.Router字段，因此，RegisterRouter就能直接访问mux.Router的字段以及\*mux.Router的方法，就好像RegisterRouter继承自\*mux.Router。

```go
import (
	"github.com/gorilla/mux"
)

// Inheritance and subclassing in Go
type RegisterRouter struct {
	*mux.Router
}

func NewRegisterRouter() *RegisterRouter {
	router := mux.NewRouter()
	return &RegisterRouter{router}
}

func (regr *RegisterRouter)RegisterBlueprint(bp *Blueprint) {
	for pattern, requestHandler := range bp.Routes {
		regr.HandleFunc(pattern, requestHandler.Method).Methods(requestHandler.MethodType)
	}
}

```
如上面代码所示，自定义的RegisterRouter类型，也包含了HandleFunc方法，并且，它还能作为http.Handler被传给ListenAndServe。

下面再看新的注册URL的方式：
```go
// main.go
package main

import (
	"fmt"
	"net/http"
	"tm.com/concurrent/blueprint"
	"tm.com/concurrent/user"
	"tm.com/concurrent/entity"
)

func main() {
	router := blueprint.NewRegisterRouter()
	router.RegisterBlueprint(user.BP)
	router.RegisterBlueprint(entity.BP)

	fmt.Println("service starting...")
	http.ListenAndServe("localhost:8080", router)
}

```

如上面代码所示，这里使用了自定义的一个RegisterRouter类型，再通过一个RegisterRouter对象注册了各个业务模块的Blueprint。

很明显，这带来的好处是各个业务模块更加内敛，松耦合，它们根本不需要关心实际的URL注册问题。

