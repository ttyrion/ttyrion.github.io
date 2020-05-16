---
layout:         page
title:          【Go】Go并发模式：Context
date:           2020-05-09
author:         翼
header-img: image/bg.jpg
catalog: true
tags:
---

首先，在Go服务中，每个到来的请求都是在一个单独的goroutine中处理的。请求处理方法中，经常需要启动额外的goroutines来完成一些工作，比如：并发去请求第三方的HTTP服务、请求数据库等等。所有这些goroutines都在处理同一个请求。那么，很有可能它们都需要访问共同的与请求相关的一些数据，比如用户id，设备id，以及请求的deadline。当一个请求被取消或者超时，所有这些处理该请求的goroutines，都应该迅速推出，以便Go系统能拿回它们占用的资源。这就涉及到Go的Context。

### Context
Context定义在context包中。
```go

// A Context carries a deadline, a cancellation signal(actually an empty struct passed by channel), 
//	and other values across API boundaries.
// Context's methods may be called by multiple goroutines simultaneously.
type Context interface {
	// Deadline returns the time when work (which done on behalf of this context) should be canceled.
	// Deadline returns ok==false when no deadline is set. 
	Deadline() (deadline time.Time, ok bool)

	// Done returns a channel that's closed when work (which done on behalf of this context) should be canceled. 
	// Done may return nil if this context can never be canceled(ie, Done would block forever). 

	// WithCancel() arranges for Done to be closed when cancel is called;
	// WithDeadline() arranges for Done to be closed when the deadline expires; 
	// WithTimeout() arranges for Done to be closed when the timeout elapses.

	// 可见Done()返回的类型被定义为一个只读通道，并且只作为“信号”使用（struct{}）
	Done() <-chan struct{}

	// If Done is not yet closed, Err returns nil.
	// If Done is closed, Err returns a non-nil error explaining why:
	// Canceled if the context was canceled
	// or DeadlineExceeded if the context's deadline passed.
	Err() error

	// A key can be any type that supports equality;
	// packages should define keys as an unexported type to avoid
	// collisions.
	//
	// Packages that define a Context key should provide type-safe accessors
	// for the values stored using that key:
	//

	Value(key interface{}) interface{}
}

```

Go源码里面的注释很清楚地描述了各个方法的作用。

1. Done()方法返回一个只读chan struct{}, 很明显，该通道是作为**信号**（取消信号）使用的。当Done()返回的通道被close时（所有从此通道读取的goroutine都会唤醒，并读取到nil），相应的Goroutine里面的函数应该释放资源、停止执行并返回。
2. Err()方法返回一个error，表示Context被取消的原因（通道被close），或者返回nil。
3. Context并不包含一个Cancel方法，原因与Done()返回的是一个只读通道一样：通常收到取消信号（读取Done()通道）的goroutine并不是发送取消信号的goroutine。而且，当某个操作启动了几个goroutines去完成一些子操作时，这些子操作不能取消父操作。
4. Context可以被多个goroutines安全地并行访问。一个Context可以被传递给多个goroutines，并且取消这个Context来通知所有那些goroutines。
5. Deadline方法使得那些函数能够判断它们该不该开始工作。如果剩余的时间过少，那启动工作也就没有意义了。代码里面也可以用deadline来设置IO超时。
6. Value允许Context搭载请求作用域（**request-scoped**）内的数据。当然，这些数据类型必须是能安全地被多个goroutine并行访问的。

#### Context的派生
context包提供了一些函数来从现有的Contexts中派生出新的Context。这些Context组成树的结构：当一个Context被取消时，所有从它派生出来的Context都会被取消。

**Background**是所有**Context Tree**的根Context，它永远不会被取消。
```go
// Background returns an empty Context. It is never canceled, has no deadline,
// and has no values. Background is typically used in main, init, and tests,
// and as the top-level Context for incoming requests.
func Background() Context

```

**WithCancel**和**WithTimeout**这两个方法负责返回新的派生出来的Context，这些Context会在父Context被取消之前被取消。**与一个请求关联的Context(可通过r.Context()获取)通常在请求处理方法返回时被取消**。**WithValue**负责把请求作用域内的数据与Context关联起来。

```go

// A CancelFunc cancels a Context.
type CancelFunc func()

// WithCancel returns a copy of parent whose Done channel is closed as soon as
// parent.Done is closed or cancel is called.
func WithCancel(parent Context) (ctx Context, cancel CancelFunc)

// WithTimeout returns a copy of parent whose Done channel is closed as soon as
// parent.Done is closed, cancel is called, or timeout elapses. The new
// Context's Deadline is the sooner of now+timeout and the parent's deadline, if
// any. If the timer is still running, the cancel function releases its
// resources.
func WithTimeout(parent Context, timeout time.Duration) (Context, CancelFunc)

// WithValue returns a copy of parent whose Value method returns val for key.
func WithValue(parent Context, key interface{}, val interface{}) Context

```



#### Context的应用
#### 1. 通知子操作取消执行
这里定义了一个测试接口：
```go
var BP *blueprint.Blueprint

func init() {
	BP = blueprint.NewBlueprint()
	BP.RegisterRoute("/context", HandleContext)
}

func HandleContext(w http.ResponseWriter, r *http.Request) {
	ctx := r.Context()
	ctx = context.WithValue(ctx, 1, "value1")
	for i := 0; i < 3; i++ {
		go getData(ctx)
	}
	select {
		case <-time.After(3 * time.Second):
			fmt.Println("context request processed")
	}
	
	w.Write([]byte(""))
}

func getData(ctx context.Context) string  {
	var data string
	var ok bool
	if data,ok = ctx.Value(1).(string); ok {
		fmt.Println("getData: get data=", data)
	} 
	select {
	case <-time.After(10 * time.Second):
		fmt.Println("getData: process takes 10 seconds")
	case <-ctx.Done():
		fmt.Println("getData: process cancelled")
	}
	return data	
}

```
HandleContext处理对/context的请求，并且通过time.After来模拟耗时的任务。HandleContext的业务在3s内会处理完，而它启动的goroutine都需要10s才能处理完工作。在浏览器请求/context接口，命令行输出如下：
```go
getData: get data= value1
getData: get data= value1
getData: get data= value1
// 这里等待3s
context request processed
getData: process cancelled
getData: process cancelled
getData: process cancelled

```
这里发生的情况是这样的：请求处理方法HandleContext处理完请求（耗时约3s）就返回了，与该请求关联的Context，即r.Context()，会被取消。而HandleContext启动的几个goroutine，执行getData函数，传递的Context都是从r.Context()派生的，当r.Context()被取消时，会先取消这些派生来的Context。

这里改动一下，再试试另外一种情况。
```go
func HandleContext(w http.ResponseWriter, r *http.Request) {
	ctx := r.Context()
	ctx = context.WithValue(ctx, 1, "value1")
	for i := 0; i < 3; i++ {
		go getData(ctx)
	}
	select {
		case <-time.After(3 * time.Second):
			fmt.Println("context request processed")
		case <-ctx.Done():
			fmt.Println("context request cancelled")
	}
	
	w.Write([]byte(""))
}

```
在HandleContext里面也加上读取Done()通道的代码，监听子Context被取消的情况。这次用浏览器请求/context后立即关闭浏览器。可以看到命令行输出如下：
```go
getData: get data= value1
getData: get data= value1
getData: get data= value1
context request cancelled
getData: process cancelled
getData: process cancelled
getData: process cancelled

```
如上，可发现HandleContext也收到了子Context的取消信号。这里的情况与上面类似，不同的地方是，请求关联的Context（r.Context()）不是因为请求处理方法返回而被取消，而是浏览器关闭（客户端断开连接）而取消。

另外，调用WithCancel或者WithTimeout方法返回的CancelFunc，也可以**主动发送取消信号**。


#### 2. 超时设置
下面利用Context来设置对外请求的超时处理。
```go

func HTTPGet(url string, timeoutMs int) ([]byte, error, int) {
	// 生产环境不能这么用，详情参考另外一篇《Go并发模式：http.Client和TIME_WAIT连接的故事》
	client := &http.Client{}

	req, err := http.NewRequest(http.MethodGet, url, nil)
	if err != nil {
		return nil, err, http.StatusInternalServerError
	}

	ctx,cancel := context.WithTimeout(context.Background(), time.Duration(timeoutMs) * time.Millisecond)
	defer cancel()
	req = req.WithContext(ctx)
	
	req.Header.Set("Accept-Encoding", "gzip")
	res, err := client.Do(req)
	if err != nil {
		return nil, err, http.StatusInternalServerError
	}

	// It is the caller's responsibility to close Body.
	defer res.Body.Close()
	if res.StatusCode != http.StatusOK {
		return nil, errors.New("wrong http status code"), res.StatusCode
	}

	var body []byte
	encoding := res.Header.Get("Content-Encoding")
	if encoding == "gzip" {
		var reader *gzip.Reader
		reader, err = gzip.NewReader(res.Body)
		// It is the caller's responsibility to call Close on the Reader when done.
		defer reader.Close()
		body, err = ioutil.ReadAll(reader)
	} else {
		body, err = ioutil.ReadAll(res.Body)
	}

	if err != nil {
		return nil, err, http.StatusInternalServerError
	}

	return body, nil, http.StatusOK
}

```
如上面的代码，从background Context派生了一个Context，并设置了超时时间，并将这个Context与请求req关联起来。如下是测试这个HTTPGet函数的代码：
```go
result,err,code := utility.HTTPGet("https://blog.golang.org/context/google/google.go", 100)
fmt.Println("HandleGet:", err, code)
w.Write([]byte(result))

```
命令行输出是这样的：
```go
HandleGet: Get https://blog.golang.org/context/google/google.go: context deadline exceeded 0

```
显然，请求https://blog.golang.org/context/google/google.go时，超过100ms未返回数据。再把超时时间设置为1000:
```go
result,err,code := utility.HTTPGet("https://blog.golang.org/context/google/google.go", 1000)

```
命令行输出是这样的：
```go
HandleGet: <nil> 200

```
说明请求https://blog.golang.org/context/google/google.go，在1000ms内处理完成了。
