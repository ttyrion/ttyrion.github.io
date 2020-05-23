---
layout:         page
title:          【Go】Go进程的启动
date:           2020-05-22
author:         翼
header-img: image/bg.jpg
catalog: true
tags:
---

Go进程启动的入口，并不是我们写的main.main函数，而是runtime·rt0_go，这个函数是汇编代码实现的。以AMD64平台为例，入口函数的定义在runtime/asm_amd64.s文件里面。下面是摘取出的部分代码，只保留关键流程的代码：
```go

TEXT runtime·rt0_go(SB),NOSPLIT,$0
	
	
	// create istack out of the given (operating system) stack.
	// _cgo_init may update stackguard.
	MOVQ	$runtime·g0(SB), DI
	LEAQ	(-64*1024+104)(SP), BX
	MOVQ	BX, g_stackguard0(DI)
	MOVQ	BX, g_stackguard1(DI)
	MOVQ	BX, (g_stack+stack_lo)(DI)
	MOVQ	SP, (g_stack+stack_hi)(DI)

	// find out information about the processor we're on
	MOVQ	$0, AX
	CPUID
	MOVQ	AX, SI
	CMPQ	AX, $0
	JE	nocpuinfo

	// Figure out how to serialize RDTSC.
	// On Intel processors LFENCE is enough. AMD requires MFENCE.
	// Don't know about the rest, so let's do MFENCE.
	CMPL	BX, $0x756E6547  // "Genu"
	JNE	notintel
	CMPL	DX, $0x49656E69  // "ineI"
	JNE	notintel
	CMPL	CX, $0x6C65746E  // "ntel"
	JNE	notintel
	MOVB	$1, runtime·lfenceBeforeRdtsc(SB)
notintel:

	// Load EAX=1 cpuid flags
	MOVQ	$1, AX
	CPUID
	MOVL	CX, runtime·cpuid_ecx(SB)
	MOVL	DX, runtime·cpuid_edx(SB)

	// Load EAX=7/ECX=0 cpuid flags
	CMPQ	SI, $7
	JLT	no7
	MOVL	$7, AX
	MOVL	$0, CX
	CPUID
	MOVL	BX, runtime·cpuid_ebx7(SB)
no7:
	// Detect AVX and AVX2 as per 14.7.1  Detection of AVX2 chapter of [1]
	// [1] 64-ia-32-architectures-software-developer-manual-325462.pdf
	// http://www.intel.com/content/dam/www/public/us/en/documents/manuals/64-ia-32-architectures-software-developer-manual-325462.pdf
	MOVL	runtime·cpuid_ecx(SB), CX
	ANDL    $0x18000000, CX // check for OSXSAVE and AVX bits
	CMPL    CX, $0x18000000
	JNE     noavx
	MOVL    $0, CX
	// For XGETBV, OSXSAVE bit is required and sufficient
	XGETBV
	ANDL    $6, AX
	CMPL    AX, $6 // Check for OS support of YMM registers
	JNE     noavx
	MOVB    $1, runtime·support_avx(SB)
	TESTL   $(1<<5), runtime·cpuid_ebx7(SB) // check for AVX2 bit
	JEQ     noavx2
	MOVB    $1, runtime·support_avx2(SB)
	JMP     testbmi1
noavx:
	MOVB    $0, runtime·support_avx(SB)
noavx2:
	MOVB    $0, runtime·support_avx2(SB)
testbmi1:
	// Detect BMI1 and BMI2 extensions as per
	// 5.1.16.1 Detection of VEX-encoded GPR Instructions,
	//   LZCNT and TZCNT, PREFETCHW chapter of [1]
	MOVB    $0, runtime·support_bmi1(SB)
	TESTL   $(1<<3), runtime·cpuid_ebx7(SB) // check for BMI1 bit
	JEQ     testbmi2
	MOVB    $1, runtime·support_bmi1(SB)
testbmi2:
	MOVB    $0, runtime·support_bmi2(SB)
	TESTL   $(1<<8), runtime·cpuid_ebx7(SB) // check for BMI2 bit
	JEQ     nocpuinfo
	MOVB    $1, runtime·support_bmi2(SB)
nocpuinfo:	
	
	// if there is an _cgo_init, call it.
	MOVQ	_cgo_init(SB), AX
	TESTQ	AX, AX
	JZ	needtls
	// g0 already in DI
	MOVQ	DI, CX	// Win64 uses CX for first parameter
	MOVQ	$setg_gcc<>(SB), SI
	CALL	AX

	// update stackguard after _cgo_init
	MOVQ	$runtime·g0(SB), CX
	MOVQ	(g_stack+stack_lo)(CX), AX
	ADDQ	$const__StackGuard, AX
	MOVQ	AX, g_stackguard0(CX)
	MOVQ	AX, g_stackguard1(CX)

#ifndef GOOS_windows
	JMP ok
#endif
needtls:
#ifdef GOOS_plan9
	// skip TLS setup on Plan 9
	JMP ok
#endif
#ifdef GOOS_solaris
	// skip TLS setup on Solaris
	JMP ok
#endif

	LEAQ	runtime·m0+m_tls(SB), DI
	CALL	runtime·settls(SB)

	// store through it, to make sure it works
	get_tls(BX)
	MOVQ	$0x123, g(BX)
	MOVQ	runtime·m0+m_tls(SB), AX
	CMPQ	AX, $0x123
	JEQ 2(PC)
	MOVL	AX, 0	// abort
ok:
	// set the per-goroutine and per-mach "registers"
	get_tls(BX)
	LEAQ	runtime·g0(SB), CX
	MOVQ	CX, g(BX)
	LEAQ	runtime·m0(SB), AX

	// save m->g0 = g0
	MOVQ	CX, m_g0(AX)
	// save m0 to g0->m
	MOVQ	AX, g_m(CX)

	CLD				// convention is D is always left cleared
	CALL	runtime·check(SB)

	MOVL	16(SP), AX		// copy argc
	MOVL	AX, 0(SP)
	MOVQ	24(SP), AX		// copy argv
	MOVQ	AX, 8(SP)
	CALL	runtime·args(SB)
	CALL	runtime·osinit(SB)
	CALL	runtime·schedinit(SB)

	// create a new goroutine to start program
	MOVQ	$runtime·mainPC(SB), AX		// entry
	PUSHQ	AX
	PUSHQ	$0			// arg size
	CALL	runtime·newproc(SB)
	POPQ	AX
	POPQ	AX

	// start this M
	CALL	runtime·mstart(SB)

	MOVL	$0xf1, 0xf1  // crash
	RET


```