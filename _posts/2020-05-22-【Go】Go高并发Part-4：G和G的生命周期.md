---
layout:         page
title:          【Go】Go高并发Part-3：G的生命周期
date:           2020-05-22
author:         翼
header-img: image/bg.jpg
catalog: true
tags:
---

https://toutiao.io/posts/1o7pvc/preview

#### G的结构体定义
```go
// stack结构描述的是一个Go的运行栈，栈内存空间范围是[lo, hi)
type stack struct {
	lo uintptr
	hi uintptr
}

type g struct {
	// Stack parameters.
	// stack describes the actual stack memory: [stack.lo, stack.hi).
	// stackguard0 is the stack pointer compared in the Go stack growth prologue.
	// It is stack.lo+StackGuard normally, but can be StackPreempt to trigger a preemption.
	// stackguard1 is the stack pointer compared in the C stack growth prologue.
	// It is stack.lo+StackGuard on g0 and gsignal stacks.
	// It is ~0 on other goroutine stacks, to trigger a call to morestackc (and crash).
	stack       stack   // offset known to runtime/cgo
	stackguard0 uintptr // offset known to liblink
	stackguard1 uintptr // offset known to liblink

	_panic         *_panic // innermost panic - offset known to liblink
	_defer         *_defer // innermost defer
	m              *m      // current m; offset known to arm liblink
	stackAlloc     uintptr // stack allocation is [stack.lo,stack.lo+stackAlloc)
	sched          gobuf
	syscallsp      uintptr        // if status==Gsyscall, syscallsp = sched.sp to use during gc
	syscallpc      uintptr        // if status==Gsyscall, syscallpc = sched.pc to use during gc
	stkbar         []stkbar       // stack barriers, from low to high (see top of mstkbar.go)
	stkbarPos      uintptr        // index of lowest stack barrier not hit
	stktopsp       uintptr        // expected sp at top of stack, to check in traceback
	param          unsafe.Pointer // passed parameter on wakeup
	atomicstatus   uint32
	stackLock      uint32 // sigprof/scang lock; TODO: fold in to atomicstatus
	goid           int64
	waitsince      int64  // approx time when the g become blocked
	waitreason     string // if status==Gwaiting
	schedlink      guintptr
	preempt        bool     // preemption signal, duplicates stackguard0 = stackpreempt
	paniconfault   bool     // panic (instead of crash) on unexpected fault address
	preemptscan    bool     // preempted g does scan for gc
	gcscandone     bool     // g has scanned stack; protected by _Gscan bit in status
	gcscanvalid    bool     // false at start of gc cycle, true if G has not run since last scan; transition from true to false by calling queueRescan and false to true by calling dequeueRescan
	throwsplit     bool     // must not split stack
	raceignore     int8     // ignore race detection events
	sysblocktraced bool     // StartTrace has emitted EvGoInSyscall about this goroutine
	sysexitticks   int64    // cputicks when syscall has returned (for tracing)
	traceseq       uint64   // trace event sequencer
	tracelastp     puintptr // last P emitted an event for this goroutine
	lockedm        *m
	sig            uint32
	writebuf       []byte
	sigcode0       uintptr
	sigcode1       uintptr
	sigpc          uintptr
	gopc           uintptr // pc of go statement that created this goroutine
	startpc        uintptr // pc of goroutine function
	racectx        uintptr
	waiting        *sudog    // sudog structures this g is waiting on (that have a valid elem ptr); in lock order
	cgoCtxt        []uintptr // cgo traceback context

	// Per-G GC state

	// gcRescan is this G's index in work.rescan.list. If this is
	// -1, this G is not on the rescan list.
	//
	// If gcphase != _GCoff and this G is visible to the garbage
	// collector, writes to this are protected by work.rescan.lock.
	gcRescan int32

	// gcAssistBytes is this G's GC assist credit in terms of
	// bytes allocated. If this is positive, then the G has credit
	// to allocate gcAssistBytes bytes without assisting. If this
	// is negative, then the G must correct this by performing
	// scan work. We track this in bytes to make it fast to update
	// and check for debt in the malloc hot path. The assist ratio
	// determines how this corresponds to scan work debt.
	gcAssistBytes int64
}
```

#### G的创建
一条go语句会创建一个G。编译器最终把这样一条go语句转为调用runtime.newproc来创建G。代码如下：
```go
// Create a new g running fn with siz bytes of arguments.
// Put it on the queue of g's waiting to run.
//go:nosplit
func newproc(siz int32, fn *funcval) {
    argp := add(unsafe.Pointer(&fn), sys.PtrSize)
    pc := getcallerpc(unsafe.Pointer(&siz))
    systemstack(func() {
        newproc1(fn, (*uint8)(argp), siz, 0, pc)
    })
}

```
newproc获取了调用者的PC（the program counter），也就是那条go语句的地址，并通过调用systemstack，在系统栈中调用newproc1。G的创建工作都是由newproc1完成的。

```go
// systemstack runs fn on a system stack.
// If systemstack is called from the per-OS-thread (g0) stack, or
// if systemstack is called from the signal handling (gsignal) stack,
// systemstack calls fn directly and returns.
// Otherwise, systemstack is being called from the limited stack of an ordinary goroutine.
// In this case, systemstack switches to the per-OS-thread stack, calls fn, and switches back.

//go:noescape
func systemstack(fn func())

```
每个M有一个关联的系统栈（M的g0的栈），以及一个信号栈（gsignal）。系统栈和信号栈空间不能动态增长，但是足够执行runtime代码。每个普通的G有自己的用户栈，用户栈初始空间很小（比如可能是2K），其空间大小可以动态变化。

systemstack的作用就是在系统栈中执行其参数对应的函数。它与mcall的不同点是，在系统栈中执行完对应函数后，systemstack会切回正在执行的代码，而mcall不会返回。systemstack和mcall通过切回CPU的SP（当前线程使用的栈的栈顶地址）和PC（下一条要执行的指令地址）寄存器的值来达到**在不同的上下文环境执行goroutine代码**的目的。

为什么要在系统栈中执行代码呢？运行在系统栈上的代码是隐式不可抢占的，也不会被GC检测。因此runtime里面很多地方都会通过systemstack, mcall等等来切换到系统栈**执行一些不被抢占的任务**。

接下来看看newproc1，它真正负责创建G。这个函数代码比较多，源码请看[newproc1](https://golang.org/src/runtime/proc.go#L3388)，这里只保留部分关键流程的代码。
```go

// Create a new g running fn with narg bytes of arguments starting at argp and returning nret bytes of results.
// callerpc is the address of the go statement that created this. 
// The new g is put on the queue of g's waiting to run.
func newproc1(fn *funcval, argp *uint8, narg int32, nret int32, callerpc uintptr) *g {
	// getg() != getg().m.curg
	// getg()返回的可能是M的g0或者gsignal，因为用户g可以被切换到系统栈或信号栈执行goroutine代码
	_g_ := getg()

	...

	_p_ := _g_.m.p.ptr()
	// 从当前P的free G list，或者全局的free G list获取一个G，而不是新创建一个G
	newg := gfget(_p_)
	if newg == nil {
		// 新创建一个G，初始状态为_Gidle，栈大小为_StackMin(2K)
		newg = malg(_StackMin)
		// 将G的状态改为_Gdead
		casgstatus(newg, _Gidle, _Gdead)
        newg.gcRescan = -1
        // 新创建的G被加入全局的allgs切片
		allgadd(newg) // publishes with a g->status of Gdead so GC scanner doesn't look at uninitialized stack.
	}

	...
	
	// 设置栈顶指针=栈顶-参数占用的空间
	sp := newg.stack.hi - totalSize
	// 初始化G的gobuf，保存sp，pc，任务函数等
	...

	// 设置新创建的G状态为_Grunnable，让它可以被调度执行（不是立即执行）
	casgstatus(newg, _Gdead, _Grunnable)
	...

	// 将新创建的G，放入当前P的runnable queue 或者（当前P的队列已满）全局的runnable queue.
	runqput(_p_, newg, true)

	// 如果有空闲的p 且没有M处于自旋状态 且 main goroutine已经启动，那么唤醒或创建新的M来执行任务
	if atomic.Load(&sched.npidle) != 0 && atomic.Load(&sched.nmspinning) == 0 && runtimeInitTime != 0 {
		wakep()
	}
	...

	return newg
}

```
newproc1主要的逻辑如下：
1. 通过当前G获取到当前M关联的P，从当前P的free G 链表（如果没有，从全局的free G链表）获取一个G来复用。如果没有获取到可复用的G，则调用malg创建一个新的G，并设置好G的栈大小，SP，PC等等。新创建的G还会被加入全局的allgs切片中，也就是说，一个G只要被创建出来，就不会被回收。不过G引用的其他对象会被释放。
1. 设置G的状态为_Grunnable并且将它加入当前P的runnable G 队列（如果已满，加入全局的 Runnable G 队列）中。
1. 如果还有空闲的P而且没有处于自旋状态的M，则唤醒或者创建新的M来执行任务。

#### G的调度执行


#### G的阻塞

#### G的退出


#### G的状态
一个Goroutine（后面称为G）从创建到执行到退出，会涉及到各种状态转换。源码定义了G的状态有如下几种：
```go
const (
    // G status
    //
    // Beyond indicating the general state of a G, the G status
    // acts like a lock on the goroutine's stack (and hence its
    // ability to execute user code).

    // _Gidle means this goroutine was just allocated and has not yet been initialized.
    _Gidle = iota // 0

    // _Grunnable means this goroutine is on a run queue.
    // It is not currently executing user code. The stack is not owned.
    _Grunnable // 1

    // _Grunning means this goroutine may execute user code. The stack is owned by this goroutine.
    // It is not on a run queue.
    // It is assigned an M and a P.
    _Grunning // 2

    // _Gsyscall means this goroutine is executing a system call.
    // It is not executing user code. The stack is owned by this goroutine.
    // It is not on a run queue. It is assigned an M.
    _Gsyscall // 3

    // _Gwaiting means this goroutine is blocked in the runtime.
    // It is not executing user code. It is not on a run queue, but should be recorded
    // somewhere (e.g., a channel wait queue) so it can be ready()d when necessary.
    // The stack is not owned *except* that a channel operation may read or write 
    // parts of the stack under the appropriate channel lock. 
    // Otherwise, it is not safe to access the stack after a goroutine enters _Gwaiting (e.g., it may get moved).
    _Gwaiting // 4

    // _Gmoribund_unused is currently unused 
    _Gmoribund_unused // 5

    // _Gdead means this goroutine is currently unused. 
    // It may be just exited, on a free list, or just being initialized.
    // It is not executing user code. It may or may not have a stack allocated. 
    // The G and its stack (if any) are owned by the M that is exiting the G or that obtained the G from the free list.
    _Gdead // 6

    // _Genqueue_unused is currently unused.
    _Genqueue_unused // 7

    // _Gcopystack means this goroutine's stack is being moved.
    // It is not executing user code and is not on a run queue. 
    // The stack is owned by the goroutine that put it in _Gcopystack.
    _Gcopystack // 8

    // _Gscan combined with one of the above states other than _Grunning indicates
    // that GC is scanning the stack. 
    // The goroutine is not executing user code and the stack is owned by the goroutine that set the _Gscan bit.
    //
    // _Gscanrunning is different: it is used to briefly block state transitions 
    // while GC signals the G to scan its own stack. This is otherwise like _Grunning.
    //
    // atomicstatus&~Gscan gives the state the goroutine will
    // return to when the scan completes.
    _Gscan         = 0x1000
    _Gscanrunnable = _Gscan + _Grunnable // 0x1001
    _Gscanrunning  = _Gscan + _Grunning  // 0x1002
    _Gscansyscall  = _Gscan + _Gsyscall  // 0x1003
    _Gscanwaiting  = _Gscan + _Gwaiting  // 0x1004
)

```