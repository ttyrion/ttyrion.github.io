---
layout:         page
title:          【Go】Go进程的启动
date:           2020-05-22
author:         翼
header-img: image/bg.jpg
catalog: true
tags:
---

#### Go进程启动入口
Go进程启动的入口，并不是main包的main函数。下面通过查看汇编代码，找到进程入口。
```go
func main() {
    fmt.Println("Hello, world.")
}

```
go build生成可执行文件后，利用 [objdump -f](https://linux.die.net/man/1/objdump) 查看目标文件头的摘要信息：
```go
[sxh@localhost concurrent]$ objdump -f concurrent

concurrent:     file format elf64-x86-64
architecture: i386:x86-64, flags 0x00000112:
EXEC_P, HAS_SYMS, D_PAGED
start address 0x0000000000454b70

[sxh@localhost concurrent]$ 

```
从上面最后一行可以看出，入口的地址是**0x0000000000454b70**。那么再通过 objdump -d 反汇编来查看这个地址是哪个函数。反汇编结果中部分内容如下：
```go
0000000000454b70 <_rt0_amd64_linux>:
  454b70:    e9 2b c6 ff ff           jmpq   4511a0 <_rt0_amd64>
  454b75:    cc                       int3   
  454b76:    cc                       int3   
  454b77:    cc                       int3   
  454b78:    cc                       int3   
  454b79:    cc                       int3   

```
根据反汇编的结果可以看到，在我这台开发机上，Go启动的入口是**_rt0_amd64_linux**。

已知进程启动入口是_rt0_amd64_linux，查看源码可以看到该函数代码：
```go
// rt0_linux_amd64.s

TEXT _rt0_amd64_linux(SB),NOSPLIT,$-8
    LEAQ    8(SP), SI // argv
    MOVQ    0(SP), DI // argc
    MOVQ    $main(SB), AX
    JMP    AX

```
_rt0_amd64_linux函数很简单，它把参数argv和argc分别存入SI，DI寄存器后，即调用**main**函数。查看不同系统和体系结构的Go汇编代码可以发现，所有的入口函数都是这个逻辑：保存参数、调用main函数。我们也就可以认为，**一个Go进程的入口即是由汇编实现的这个runtime.main函数**。

#### Go进程启动流程
既然不同的系统和体系结构下的Go进程启动都会调用runtime.main函数，那么查看函数代码：
```go
TEXT main(SB),NOSPLIT,$0
    JMP    runtime·rt0_go(SB)

```
可见，这个main函数也很简单， 它只是调用了runtime·rt0_go函数。这个函数是汇编代码实现的，以AMD64平台为例，入口函数的定义在runtime/asm_amd64.s文件里面。下面是摘取出的部分代码，只保留关键流程的代码：
```go

TEXT runtime·rt0_go(SB),NOSPLIT,$0
    // create istack out of the given (operating system) stack.
    // _cgo_init may update stackguard.
    MOVQ    $runtime·g0(SB), DI
    LEAQ    (-64*1024+104)(SP), BX
    MOVQ    BX, g_stackguard0(DI)
    MOVQ    BX, g_stackguard1(DI)
    MOVQ    BX, (g_stack+stack_lo)(DI)
    MOVQ    SP, (g_stack+stack_hi)(DI)
```
这里用到了Go的4个伪寄存器之一的SB寄存器。
```go
// Go的4个伪寄存器
FP(stack frame pointer) 栈帧指针。指向栈帧的底布。
SP(stack pointer) 栈顶指针。始终指向栈空间的顶部。

    High Addr  +++++++++++++++++++
 Frame 0 +-->  |   Param n       |
               |     ...         |
               |   Param 1       |
               |   Return Addr   |
               +++++++++++++++++++
 Frame 1 +-->  |   Previous FP   |  <---+ FP  
               |   Local 1       |
               |     ...         |
               |   Local n       |
               |   Param n       |
               |     ...         |
               |   Param 1       |
               |   Return Addr   |  <---+ SP
    Low Addr   +++++++++++++++++++


PC(program counter) 程序计数器，负责跳转和分支。
SB(static base pointer) 静态全局符号(symbol)基指针。

// SP寄存器的说明，有些硬件体系里面是有物理SP寄存器的
On architectures with a hardware register named SP, the name prefix distinguishes 
references to the virtual stack pointer from references to the architectural 
SP register. That is, x-8(SP) and -8(SP) are different memory locations:
the first refers to the virtual stack pointer pseudo-register, while 
the second refers to the hardware’s SP register.
```

回到上面那段代码。那段代码就是把全局变量runtime.g0的地址存入DI寄存器(runtime.g0是进程第一个goroutine)，接着就设置了g0的栈的边界。


    // find out information about the processor we're on
    MOVQ    $0, AX
    CPUID
    MOVQ    AX, SI
    CMPQ    AX, $0
    JE    nocpuinfo

    // Figure out how to serialize RDTSC.
    // On Intel processors LFENCE is enough. AMD requires MFENCE.
    // Don't know about the rest, so let's do MFENCE.
    CMPL    BX, $0x756E6547  // "Genu"
    JNE    notintel
    CMPL    DX, $0x49656E69  // "ineI"
    JNE    notintel
    CMPL    CX, $0x6C65746E  // "ntel"
    JNE    notintel
    MOVB    $1, runtime·lfenceBeforeRdtsc(SB)
notintel:

    // Load EAX=1 cpuid flags
    MOVQ    $1, AX
    CPUID
    MOVL    CX, runtime·cpuid_ecx(SB)
    MOVL    DX, runtime·cpuid_edx(SB)

    // Load EAX=7/ECX=0 cpuid flags
    CMPQ    SI, $7
    JLT    no7
    MOVL    $7, AX
    MOVL    $0, CX
    CPUID
    MOVL    BX, runtime·cpuid_ebx7(SB)
no7:
    // Detect AVX and AVX2 as per 14.7.1  Detection of AVX2 chapter of [1]
    // [1] 64-ia-32-architectures-software-developer-manual-325462.pdf
    // http://www.intel.com/content/dam/www/public/us/en/documents/manuals/64-ia-32-architectures-software-developer-manual-325462.pdf
    MOVL    runtime·cpuid_ecx(SB), CX
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
    MOVQ    _cgo_init(SB), AX
    TESTQ    AX, AX
    JZ    needtls
    // g0 already in DI
    MOVQ    DI, CX    // Win64 uses CX for first parameter
    MOVQ    $setg_gcc<>(SB), SI
    CALL    AX

    // update stackguard after _cgo_init
    MOVQ    $runtime·g0(SB), CX
    MOVQ    (g_stack+stack_lo)(CX), AX
    ADDQ    $const__StackGuard, AX
    MOVQ    AX, g_stackguard0(CX)
    MOVQ    AX, g_stackguard1(CX)

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

    LEAQ    runtime·m0+m_tls(SB), DI
    CALL    runtime·settls(SB)

    // store through it, to make sure it works
    get_tls(BX)
    MOVQ    $0x123, g(BX)
    MOVQ    runtime·m0+m_tls(SB), AX
    CMPQ    AX, $0x123
    JEQ 2(PC)
    MOVL    AX, 0    // abort
ok:
    // set the per-goroutine and per-mach "registers"
    get_tls(BX)
    LEAQ    runtime·g0(SB), CX
    MOVQ    CX, g(BX)
    LEAQ    runtime·m0(SB), AX

    // save m->g0 = g0
    MOVQ    CX, m_g0(AX)
    // save m0 to g0->m
    MOVQ    AX, g_m(CX)

    CLD                // convention is D is always left cleared
    CALL    runtime·check(SB)

    MOVL    16(SP), AX        // copy argc
    MOVL    AX, 0(SP)
    MOVQ    24(SP), AX        // copy argv
    MOVQ    AX, 8(SP)
    CALL    runtime·args(SB)
    CALL    runtime·osinit(SB)
    CALL    runtime·schedinit(SB)

    // create a new goroutine to start program
    MOVQ    $runtime·mainPC(SB), AX        // entry
    PUSHQ    AX
    PUSHQ    $0            // arg size
    CALL    runtime·newproc(SB)
    POPQ    AX
    POPQ    AX

    // start this M
    CALL    runtime·mstart(SB)

    MOVL    $0xf1, 0xf1  // crash
    RET


```