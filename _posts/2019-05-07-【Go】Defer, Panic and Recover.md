---
layout:         page
title:         【Go】Defer, Panic and Recover
subtitle:       
date:           2019-05-07
author:         翼
header-img: image/bg.jpg
catalog: true
tags:
---

我们可以在Go里面使用 **Defer**/**Recover**，来达到C++/Java的**try**/**catch**的目的。先分别看看三者的作用。

## Defer
defer语句把一条函数调用放到一个队列，并且遵从LIFO。保存在队列里的函数调用在外围函数返回后执行。defer语句通常用于简化清理操作。

先看如下一段代码：
```javascript

func CopyFile(destFile string, srcFile string) (written int64, err error) {
    src, err := os.Open(srcFile)
    if err != nil {
        return
    }

    dst, err := os.Create(destFile)
    if err != nil {
        return
    }

    written, err = io.Copy(dst, src)
    dst.Close()
    src.Close()
    return
}

```
上面这段代码很简单，却有一个bug：如果os.Create失败，src就不会被关闭。我们可以很简单地加入一行src.Close()解决问题。但是假如是在一个复杂函数内部，有很多失败条件判断，那就需要在那些地方都加上src.Close()，并且当函数代码很多时，如果漏掉一个src.Close()，也可能不会被发现。

换作使用defer的代码：
```javascript

func CopyFile(destFile string, srcFile string) (written int64, err error) {
    src, err := os.Open(srcFile)
    if err != nil {
        return
    }

    defer src.Close()

    dst, err := os.Create(destFile)
    if err != nil {
        return
    }
    
    defer dst.Close()

    written, err = io.Copy(dst, src)
    dst.Close()
    src.Close()
    return
}

```
这里使用defer来保证不论CopyFile怎么返回（甚至是io.Copy发生异常），src和dst都会被关闭。

### Defer的三个简单规则
defer语句的行为是可预测的，有三个简单的规则可应用于defer语句。
**No.1 一个deferred函数的参数的值是在对defer语句求值时计算的**
例如：
```javascript

func Test() int {
	i := 0;
	defer func(n int) {
		fmt.Println("in defer: i=", n)
	}(i)

	i += 1
	fmt.Println("in Test: i=", i)
	return i
}

func main() {
	fmt.Println("in main: i=", Test())
}

```
输出如下:
```javascript
in Test: i= 1
in defer: i= 0
in main: i= 1
```
可见，在执行defer语句的过程中，deferred函数的参数i的值已经计算出来了，并不是等到该函数被执行时才去计算。这是合理的，拿上面CopyFile为例，假如deferred函数的参数是在它被执行时才计算，那如果src被修改，原先被打开的src文件就肯定不会被关闭。显然，这是不严谨的！

这里还有另外一种变体：
```javascript

func Test() int {
	i := 0;
	defer func() {
		fmt.Println("in defer: i=", i)
	}()

	i += 1
	fmt.Println("in Test: i=", i)
	return i
}

func main() {
	fmt.Println("in main: i=", Test())
}

```
输出如下:
```javascript
in Test: i= 1
in defer: i= 1
in main: i= 1
```
这里的区别是，deferred函数（内嵌的函数）直接访问了外围函数的局部变量，并没有通过参数传递，这就是“**闭包**”。类似JavaScript，Go也支持闭包。也就是说，defer语句中定义的内嵌函数，不论何时执行，都能访问到外围函数的局部变量。并且与JavaScript类似，Go中闭包也是通过引用实现，所以当这个内嵌函数执行时，变量i的值已经被外围函数更新了。

**No.2 一个函数内部的defer语句保存的函数调用被执行时遵从Last In First Out原则。**

**No.3 deferred函数可以访问外围函数的命名的返回变量。**
假设有一个第三方包里面的函数process：
```javascript

func process(s string) int {
	return 10 / len(s)
}

```

我们需要用这个函数，但是这个函数可能会发生异常（Go里面称为panic）。我们想在这个函数panic的时候捕获它，并且返回一个错误给外部。可能的代码如下：
```javascript

func myProcess(s string) error {
	var err error
	defer func() {
		if r := recover(); r != nil {
			fmt.Println("panic caught in myProcess.")
			err = errors.New("process error")
		}
	}()

	process(s)
	return err
}

```

下面是验证的代码：
```javascript
func main() {
	err := myProcess("")
	if err != nil {
		fmt.Println("in main: ", err)
	}
}
```
它的输出如下：
```javascript

panic caught in myProcess.

```
也就是说，process确实panic了，但是myProcess并没有返回相应的错误。

**为什么会这样？**
咋看起来，好像是myProcess最后返回了在其内部第一行定义的err。但事实是在process **panic**时，程序根本不会执行到myProcess内部return err那一行代码。添加一行代码即可证明：
```javascript
func myProcess(s string) error {
	var err error
	defer func() {
		if r := recover(); r != nil {
			fmt.Println("panic caught in myProcess.")
			err = errors.New("process error")
		}
	}()

    process(s)
	fmt.Println("befor return in myProcess.")
	return err
}

```
执行程序后发现，其输出跟上面的示例并无不同。也就是说，当process(s)产生panic时，后面的代码根本没有被执行，包括return语句。

**如何解决我们这个问题呢？**
答案就是使用**命名的**返回变量。下面是新版的myProcess代码：
```javascript

func myProcess(s string) (err error) {
	defer func() {
		if r := recover(); r != nil {
			fmt.Println("panic caught in myProcess.")
			err = errors.New("process error")
		}
	}()

	process(s)
	fmt.Println("befor return in myProcess.")
	return err
}

```
现在程序的输出变为：
```javascript

panic caught in myProcess.
in main:  process error

```
可见，使用命名返回变量后，main函数中确实取到了panic错误。

## Panic
Go有一个内置函数名为panic，在任何Go函数内部调用panic函数，就会结束正常的程序流程。调用panic的函数会立即结束执行，并且在调用栈中引起链式反应：在这个调用栈中的所有函数会一个接一个地结束。最终这个panic到达调用栈的顶端：程序会崩溃。庆幸的是：所有的deferred函数仍然会被执行，并且它们可以阻止程序崩溃。

代码示例其实在上面已经有了，上面的代码示例中有一个除0错误。在Go里面，其“效果”和调用panic基本一样。这里就可以理解，为什么上面的代码示例的myProcess函数中，return语句并没有被执行：发生panic时，调用栈所有的函数都会结束，比如这里的myProcess。

## Recover
Go还有一个内置函数叫recover，recover函数会结束panic的链式反应，阻止panic继续沿着调用栈回溯。recover函数只能在deferred函数内部调用！原因是在panic链式反应过程中，只有deferred函数会被执行。

如果调用recover时，没有发生panic，recover会返回nil。如果发生了panic，那么recover结束这个panic（可以理解为捕获异常，或者说恢复panic）并且返回panic的参数值。
