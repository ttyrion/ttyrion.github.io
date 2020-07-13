---
layout:         page
title:          【Go】Go高并发Part-4：P和P的生命周期
date:           2020-05-22
author:         翼
header-img: image/bg.jpg
catalog: true
tags:
---

#### P的结构体定义
```go

type p struct {
	lock mutex

	id          int32
	status      uint32 // one of pidle/prunning/...
	link        puintptr
	schedtick   uint32   // incremented on every scheduler call
	syscalltick uint32   // incremented on every system call
	m           muintptr // back-link to associated m (nil if idle)
	mcache      *mcache
	racectx     uintptr

	deferpool    [5][]*_defer // pool of available defer structs of different sizes (see panic.go)
	deferpoolbuf [5][32]*_defer

	// Cache of goroutine ids, amortizes accesses to runtime·sched.goidgen.
	goidcache    uint64
	goidcacheend uint64

	// Queue of runnable goroutines. Accessed without lock.
	runqhead uint32
	runqtail uint32
	runq     [256]guintptr
	// runnext, if non-nil, is a runnable G that was ready'd by
	// the current G and should be run next instead of what's in
	// runq if there's time remaining in the running G's time
	// slice. It will inherit the time left in the current time
	// slice. If a set of goroutines is locked in a
	// communicate-and-wait pattern, this schedules that set as a
	// unit and eliminates the (potentially large) scheduling
	// latency that otherwise arises from adding the ready'd
	// goroutines to the end of the run queue.
	runnext guintptr

	// Available G's (status == Gdead)
	gfree    *g
	gfreecnt int32

	sudogcache []*sudog
	sudogbuf   [128]*sudog

	tracebuf traceBufPtr

	palloc persistentAlloc // per-P to avoid mutex

	// Per-P GC state
	gcAssistTime     int64 // Nanoseconds in assistAlloc
	gcBgMarkWorker   guintptr
	gcMarkWorkerMode gcMarkWorkerMode

	// gcw is this P's GC work buffer cache. The work buffer is
	// filled by write barriers, drained by mutator assists, and
	// disposed on certain GC state transitions.
	gcw gcWork

	runSafePointFn uint32 // if 1, run sched.safePointFn at next safe point

	pad [sys.CacheLineSize]byte
}

```

#### P的状态
源码定义了P的状态有如下几种：
```go
const (
	// P status
	_Pidle    = iota
	_Prunning // Only this P is allowed to change from _Prunning.
	_Psyscall
	_Pgcstop
	_Pdead
)

```

