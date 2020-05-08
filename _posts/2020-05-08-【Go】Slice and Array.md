---
layout:         page
title:          【Go】Slice and Array
date:           2020-05-08
author:         翼
header-img: image/bg.jpg
catalog: true
tags:
---

切片在Go开发过程中，是非常常用的。但与切片相关的，也有一些容易搞混的概念，如：切片长度，切片容量。这些概念与切片的实现有关。

### slice []T
可以认为切片slice的定义，应该是如下这样子的：
```go

type slice struct {
	array *Array
	len   int
	cap   int
}

一个切片包含长度、容量、一个底层数组的指针。这里有一点就需要注意：尽管slice本身是一个struct，作为参数传递时，是以值的方式。但是底层数组并没有以值的方式被传递（Copy），也就是说函数的实参和函数内的变量，共享了这个底层数组。

```
### Array [n]T
了解了slice的定义后，再来看看Go的数组。

Go的数组的定义：
```go
var a [3]int
a[0] = 12
a[1] = 78
a[2] = 50

// 或者
b := [3]int{12, 78, 50}

```
可见，数组与slice的定义，区别在于：数组需要指定一个长度。当然，数组的长度也可以指定编译器自己计算：
```go
func main() {
	a := []int{1,1,1}
	b := [...]int{1,1,1}
	fmt.Printf("type of a:%T\n", a)
	fmt.Printf("type of b:%T\n", b)
}

// 输出
type of a:[]int
type of b:[3]int

```
可以看出来，**数组的长度就是数组的类型的一部分**。也就是说，[5]int 和 [25]int 是不同的类型。下面这个就是证明：
```go
func IterAdd(a [4]int) {
	for i:=0; i < len(a); i++ {
		a[i] += 1
	}
}

func main() {
	b := [...]int{1,1,1}
	IterAdd(b)
}


```
build时，go会提示：
```go

cannot use b (type [3]int) as type [4]int in argument to IterAdd

```
所以，再次证明：**Go数组的大小也是数组类型的一部分**，这与C是不同的，C里的数组可以当指针使用。

#### Array is Value
比如这个例子：
```go
func IterAdd(a [4]int) [4]int {
	for i:=0; i < len(a); i++ {
		a[i] += 1
	}
	return a
}

func main() {
	a := [...]int{1,1,1,1}
	b := IterAdd(a)
	fmt.Println("a:", a)
	fmt.Println("b:", b)
}

// 输出
a: [1 1 1 1]
b: [2 2 2 2]

```
可见，Go数组传递时，是以值（Copy）的方式，而不是像C那样是通过指针的方式。

### slice：len, cap
slice的定义与array类似，只是没有指定长度。因为slice基于array实现的，因此还有这么一种定义slice的方式：
```go

a := [5]int{76, 77, 78, 79, 80}
var b []int = a[1:4] //creates a slice from a[1] to a[3]

```

#### slice: modifying a slice
从上面给的slice的类型定义里面能看到，slice本身并不包含任何数据，它只是保存了一个底层数据的引用。我们对slice的任何修改，都会反映在底层数组中。

#### len, cap
**slice的长度**：slice里面的元素个数。
**slice的容量**：slice的底层数组里面，从定义slice起始元素的那个数组元素开始，直到数组末尾的元素个数。
```go

func main() {
	a := [5]int{76, 77, 78, 79, 80}
	var b []int = a[1:4]
	var c []int = a[4:]
	fmt.Println(cap(b))
	fmt.Println(cap(c))

// 输出
4
1
}

```

#### 内置的make创建slice
利用make创建slice时，最多可以传三个参数：即slice的类型，长度，和容量。
```go
// The capacity parameter cap is optional and defaults to the length.
// The make function creates an array and returns a slice reference to it.
func make([]T, len, cap) []T

```

#### slice 怎么动态增长？
问题来了：既然slice的实现基于Array，而Array的size是固定的。为什么slice的长度还可变呢？

底层实现：当一个slice已满（容量不够了，底层数组被填满了），再追加元素时，go会创建一个新的数组，原数组的元素会被copy到新数组，并且返回一个新的slice（引用新的数组）。而新的slice的容量，是旧slice的两倍！
```go
func main() {
	a := [5]int{76, 77, 78, 79, 80}
	var b []int = a[1:4]
	var c []int = a[4:]
	fmt.Println("cap of b:", cap(b))
	fmt.Println("cap of c:", cap(c))
	
	c = append(c, 90)
	fmt.Println("cap of b:", cap(b))
	fmt.Println("cap of c:", cap(c))
	
	c = append(c, 91)
	fmt.Println("cap of b:", cap(b))
	fmt.Println("cap of c:", cap(c))
}

// 输出
cap of b: 4
cap of c: 1
cap of b: 4
cap of c: 2
cap of b: 4
cap of c: 4

```
从上面可以看出：对slice c进行了两次append操作，但是slice b的底层数组并没有改变，slice b的容量也没有改变。不过，slice c的容量却因为两次append操作，翻了四倍。
