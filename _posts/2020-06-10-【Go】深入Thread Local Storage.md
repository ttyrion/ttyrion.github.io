---
layout:         page
title:          【Go】深入Thread Local Storage
date:           2020-06-10
author:         翼
header-img: image/bg.jpg
catalog: true
tags:
---

>本篇内容主要翻译自[《A Deep dive into (implicit) Thread Local Storage》](https://chao-tic.github.io/blog/2018/12/25/tls)，并作了些整理。

Thread Local Storage (**TLS**) 乍一看显得很简单，但一个高效的TLS实现，需要编译器、链接器、动态链接器、系统内核、编程语言运行时的通力协作。[《ELF Handling For Thread-Local Storage》](https://uclibc.org/docs/tls.pdf)对Linux上的TLS作了非常好的介绍。这里我主要关注x86-64平台下的Linux ELF。

### Introduction
常规的显式TLS（通过调用pthread_key_create, pthread_setspecific, and pthread_getspecific三个函数来实现的）很容易理解，可以被当作一种二维的hash map：tls_map[tid][key]。其中，tid是线程ID，key是特定的线程局部变量的hash key。

然而，隐式的TLS（如下面代码所示），看起来更有魔力：它使用起来太简单了，使人不禁猜想，是否需要花点力气才能做到这一点。
```C++
// main.c

#include <stdio.h>

__thread int main_tls_var;

int main() {
    return main_tls_var;
}

// __thread是GCC内置的线程局部存储设施。
// __thread变量不会在线程之间共享，各个线程的值互不干扰。
// __thread可以用来修饰那些带有全局性且值可能变，但是又不值得用全局变量保护的变量。
```

### The Assembly
对于C语言中的任何事物，想看透它们的“magic”，最简单的方式就是反汇编。
```C++
[root@localhost c]# gcc -O0 main.c
[root@localhost c]# objdump -d a.out > main.asm

```
查看main.asm中main的代码：
```C++
000000000040050d <main>:
  40050d:	55                   	push   %rbp
  40050e:	48 89 e5             	mov    %rsp,%rbp
  400511:	64 8b 04 25 fc ff ff 	mov    %fs:0xfffffffffffffffc,%eax
  400518:	ff 
  400519:	5d                   	pop    %rbp
  40051a:	c3                   	retq   
  40051b:	0f 1f 44 00 00       	nopl   0x0(%rax,%rax,1)

```
这里最令人迷惑的一行是 mov %fs:0xfffffffffffffffc,%eax。对于不熟悉汇编的人来说，这一行只是存储在内存地址%fs:0xfffffffffffffffc处的一个32-bit的值复制到寄存器eax中。似乎反汇编给我们带来了更多疑惑：为什么会操作fs寄存器，这个big常量0xfffffffffffffffc又是什么意思？

### The Thread Register
尽管一些其他的CPU体系架构使用专用寄存器来存储特定线程上下文，x86并不这么奢侈。x86仅有少数通用寄存器，这要求编程人员在规划寄存器的使用时必须非常慎重。Intel 80386引入FS和GS作为用户自定义的段寄存器，Intel并没有明确规定说FS和GS应该作什么用。然而，随着多线程编程的普及，人们发现了重新利用FS和GS的机会：将它们用作线程寄存器（thread register）。

这里我不会深入内存分段（memory segmentation）的细节，只是提一下：在x86-64平台上，非内核程序通过FS或GS段寄存器来访问**线程上下文，包括TLS**（Windows通过GS，Linux通过FS）。

了解了这个知识点（**FS很可能指向某个特定于线程的上下文**），我们可以尝试来解码这条指令：mov %fs:0xfffffffffffffffc,%eax 。mov指令使用段寻址方式，这个常量0xfffffffffffffffc其实就是 -4的补码。所以 mov    %fs:0xfffffffffffffffc,%eax 这条指令其实相当于：
```C++
// Pointer arithmetics to get the address of `main_tls_var`
int *main_tls_var_ptr = (int *) thread_context_ptr - 4;
// Dereference tls_var_ptr and put its value in register EAX
EAX = *main_tls_var_ptr;

```
thread_context_ptr是FS指向的数据的地址。这条mov指令执行完之后，eax寄存器存储了main_tls_var的值。

在x86-64平台上，用户态进程（Ring 3）可以取FS和GS的值，但是不能修改FS和GS中存储的地址值。通常是由操作系统提供一些设施（如**系统调用**）来间接操作FS和GS。内核通Model Specific Register (MSR) 来管理FS中存储的地址值，称为[MSR_FS_BASE](https://elixir.bootlin.com/linux/v4.18.20/source/arch/x86/include/asm/msr-index.h#L18)。内核提供系统调用[arch_prctl](https://elixir.bootlin.com/linux/v4.18.20/source/arch/x86/kernel/process_64.c#L627)来修改当前运行线程的FS和GS寄存器的值。

深挖进去的话，我们还能看到，当内核进行上下文切换（[context switch](https://elixir.bootlin.com/linux/v4.18.20/source/arch/x86/kernel/process_64.c#L418)）的时候，负责切换的代码会加载下一个任务的FS到CPU的MSR中，确保不论FS中的地址处的数据是什么，它都是基于每个线程的（不是在线程间共享的）。

基于以上内容，我们可以作出猜测：运行时（runtime）必须通过某种FS操作子程序的方式（如arch_prctl）来把TLS绑定到当前线程。并且，内核在进行上下文切换的时候交换正确的FS值，以此来追踪这个绑定关系。

### Thread Specific Context
上面提到了，FS指向的数据，是某种特定于线程的上下文。那么，这个上下文到底是什么呢？

不同的平台对这个上下文有着不同的称呼。在Linux和glibc下，它被称为Thread Control Block (**TCB**)；在Windows中，它被称为Thread Information Block (**TIB**) or Thread Environment Block (**TEB**)。

这里将专注于TCB。关于TIB/TEB的更多信息，可以参考Ken Johnson写的[Thread Local Storage](http://www.nynaeve.net/?p=180)。

不论我们如何称呼特定线程的上下文（thread specific context），叫**TCB，或者TIB/TEB**，这个上下文其实就是一个包含了方便我们管理线程以及线程的局部存储（也就是**TLS**）的信息以及元数据。

对于“x86-64 Linux + glibc”，TCB的数据结构其实就是struct pthread，有时也称为线程描述符；它是glibc内部的一种数据结构，与POSIX线程相关，但是不完全一样。

至此，我们已经知道了：FS寄存器指向的是一个TCB数据结构。但是还有一些问题未解，例如：TCB是谁分配以及设置的？我们编写代码的时候，显然没有在源码里面处理TCB相关的事情。

### The Initialisation of TCB or TLS
对于静态链接或动态链接的可执行程序，TCB或者TLS的设置过程是有些不同的。至于原因，后面会讲清楚。目前，我们先专注于动态链接的可执行程序的TCB设置的过程，因为人们发布软件的最常用的方式就是动态链接。更复杂的是，主线程与随后创建的其他线程初始化TLS的过程也不相同。显然，主线程是内核启动的，而其他线程是应用程序运行之后启动的。

对于动态链接的ELF程序，只要它们被加载、映射进内存，内核就会撒手不管，并将指挥棒交给动态链接器（Linux系统中是ld.so，macOS中是dyld）。ld.so是glibc的一部分，所以接下来我们要剖开glibc的内部，看看能否找到一些有用的东西。

我们此前做了一个假设：runtime将调用arch_prctl系统调用来修改FS寄存器。我们可以试试在glibc源码中搜索arch_prctl来看看。排除掉几处不太可能的代码之后，我们看到一个宏定义[TLS_INIT_TP](https://sourceware.org/git/?p=glibc.git;a=blob;f=sysdeps/x86_64/nptl/tls.h;h=e88561c93412489e532af8e388a8eeb1f879b771;hb=437faa9675dd916ac7b239d4584b932a11fbb984#l154) 这个宏使用内联汇编的方式触发arch_prctl系统调用并且更新FS寄存器的值为TCB的地址。

知道了TLS_INIT_TP，通过搜索TLS_INIT_TP可以发现，主线程的TLS是通过函数[init_tls](https://sourceware.org/git/?p=glibc.git;a=blob;f=elf/rtld.c;h=1b0c74739f967093d26f5867b9b9d552d8b1ad00;hb=437faa9675dd916ac7b239d4584b932a11fbb984#l681)设置的。init_tls调用_dl_allocate_tls_storage来分配TCB（即struct pthread），最后调用宏TLS_INIT_TP来将这个TCB绑定至主线程。

这一发现验证了此前的假设：动态链接器分配、设置了TCB（即struct pthread）并且调用系统调用arch_prctl来把TLS绑定至主线程。

稍后继续看看其他线程的TLS是怎么设置的，但在此之前，还有一个问题：上面访问线程局部存储的变量时，那个很奇怪的“-4”是怎么来的？

### The Internals of ELF TLS
不同的平台和体系结构下，TLS/TCB的结构也有所不同。对于x86-64，其TLS结构如图：
![tls](https://raw.githubusercontent.com/ttyrion/ttyrion.github.io/master/image/tls/tls-structure.png)

这张图包含了很多信息：
1. $tp_t$是线程寄存器（即线程t的线程指针，thread pointer，也是FS存储的数据）。
1. $dtv_t$是动态线程向量（Dynamic Thread Vector），可以把它当作一个二维数组，可通过一个模块ID（module id）以及TLS变量偏移（variable offset）来寻址任何TLS变量。TLS变量偏移是局部的（模块内部），并且是在编译期间与TLS变量绑定。这里所谓的模块，可以指代glibc中的可执行程序，或者动态共享库。因此，一个模块ID实际就是一个进程中的已加载的ELF对象的索引。注意，对于一个给定的进程来说，启动程序（main executable）的模块ID[一定是1](https://sourceware.org/git/?p=glibc.git;a=blob;f=elf/rtld.c;h=1b0c74739f967093d26f5867b9b9d552d8b1ad00;hb=437faa9675dd916ac7b239d4584b932a11fbb984#l1232)，而其他的共享对象则直到被加载且被连接器[赋值](https://sourceware.org/git/?p=glibc.git;a=blob;f=elf/dl-load.c;h=c51e4b3718ef68915d6d80356fa5caade7ef3019;hb=437faa9675dd916ac7b239d4584b932a11fbb984#l1133)一个模块ID。
1. $dtv_{t,1}$、$dtv_{t,2}$、$dtv_{t,3}$...它们每一个都包含了一个被加载的共享ELF对象（模块ID为1,2,3...）的TLS信息，并且它们指向一个TLS块，块内包含特定动态对象内部的TLS变量。只要知道某个TLS变量的偏移，就能从这个TLS块中取到这个TLS变量。

你可能已经注意到了，DTV数组中开头的几个元素指向了TCB前面的几个白色区域，而剩下的两个则指向标记为“Dynamically-Loaded Modules”的TLS块。这里的“Dynamically-Loaded Modules”并不是表示动态共享对象，它指的是通过显示调用dlopen被加载的共享对象。

TCB阴影区域前面的白色区域，可以被划分为多个TLS块，每个TLS块对应着一个被加载的模块（上图中将它们描述为$tlsoffset_1$，$tlsoffset_2$，$tlsoffset_3$...），这些TLS块包含了所有来自可执行对象以及共享对象（它们在main函数执行之前被加载）的TLS变量。在glibc术语里面，这些白色区域被称为静态TLS（static TLS ），“static”一词，是为了将这些TLS块与通过dlopen加载的模块的TLS块区分开。

这些static模块的TLS变量可以通过线程寄存器（也就是线程指针）的一个负的常量偏移（在整个进程的生命周期中都是常量）来访问。例如，指令 mov %fs:0xfffffffffffffffc 就属于这种情况。

由于可执行文件（模块）自己的模块ID永远都是1，$dtv_{t,1}$ （即dtv[1]）总是指向可执行文件的TLS块。另外，从上图中还能看到，链接器也总是把可执行文件的TLS块放在紧挨着TCB的地址处。这大概就是为了把可执行文件的TLS变量存在可预知的地址处，并且让这些TLS变量在缓存里面保持一种热的状态。

$dtv_t$是tcbhead_t的一个成员变量（定义在[tcbhead_t](https://sourceware.org/git/?p=glibc.git;a=blob;f=sysdeps/x86_64/nptl/tls.h;h=e88561c93412489e532af8e388a8eeb1f879b771;hb=437faa9675dd916ac7b239d4584b932a11fbb984#l46)），而tcbhead_t又是TCB（pthread）的第一个变量（定义在[pthread](https://sourceware.org/git/?p=glibc.git;a=blob;f=nptl/descr.h;h=9c01e1b9863b178c174508b78c7772e71ffdc5ba;hb=437faa9675dd916ac7b239d4584b932a11fbb984#l127)）。

DTV的数据结构“貌似”简单（定义在[dtv](https://sourceware.org/git/?p=glibc.git;a=blob;f=sysdeps/generic/dl-dtv.h;h=b2346b2b65b272a99ee3b3c1aac36e9af363cc18;hb=437faa9675dd916ac7b239d4584b932a11fbb984#l29)），但是它也是许多C技巧的不幸受害者（例如负指针运算，类型别名，命名不正确的成员变量等等）。

如上图所示，TCB的dtv数组的第一个元素是一个代数计数器（generation counter，非数学上的代数）$gen_t$，$gen_t$充当着版本号的作用。当DTV需要扩容或者重建的时候，$gen_t$有助于通知运行时。程序中每次调用dlopen或者dlfree时，全局的代数计数器会[自增](https://sourceware.org/git/?p=glibc.git;a=blob;f=elf/dl-open.c;h=f6c8ef1043b9a6cf90419db0f536445389764b7d;hb=437faa9675dd916ac7b239d4584b932a11fbb984#l468)。只要运行时检测到DTV被使用并且它的$gen_t$与全局的代数号码[不匹配](https://sourceware.org/git/?p=glibc.git;a=blob;f=elf/dl-tls.c;h=c87caf13d6a97ba4981c5ddaba2ecec21f4753b0;hb=437faa9675dd916ac7b239d4584b932a11fbb984#l828)，它就会再次更新DTV，并将其$gen_t$[设置](https://sourceware.org/git/?p=glibc.git;a=blob;f=elf/dl-tls.c;h=c87caf13d6a97ba4981c5ddaba2ecec21f4753b0;hb=437faa9675dd916ac7b239d4584b932a11fbb984#l726)为当前的全局代数号码:
```C
 while ((listp = listp->next) != NULL);
 /* This will be the new maximum generation counter.  */
 dtv[0].counter = new_gen;
```

正如我们刚才提到的，dtv的第二个元素dtv[1]指向可执行程序（the main executable）的TLS块，其地址（在 static TLS 中）紧挨着TCB。如下面是dtv数组得快速浏览：
```C
dtv[-1].counter; /* Pro tip: The length of this dtv array */
dtv[0].counter;  /* Generation counter for the DTV in this thread */
dtv[1].pointer;  /* Pointer to the main executable TLS block in this thread */

/* Pointer to a TLS variable defined in a module id `ti_module` */
main_tls_var = *(dtv[tls_index.ti_module].pointer + tls_index.ti_offset);

```

静态TLS和DTV配置的一个特性就是静态TLS中的某个线程TLS内的任何一个变量，都可以通过dtv[ti_module].pointer + ti_offset，或者TCB的负偏移来访问（如mov %fs:0xffffffffffxxxxxx %eax，其中0xffffffffffxxxxxx就是负偏移）。

我们**必须注意**，dtv[ti_module].pointer + ti_offset 是最通用的访问TLS变量的方式，无论这个TLS变量是在staic TLS中，还是dynamic TLS中。

### The Initialisation of DTV and static TLS
截止目前，我们已经了解了从汇编、内核，到动态链接器等等背景知识。但此前的问题依然存在：静态TLS是怎么建立的，前面的 -4 或 0xfffffffffffffffc 是怎么来的？

动态链接器维护着一个链接链表[link_map](https://sourceware.org/git/?p=glibc.git;a=blob;f=include/link.h;h=5924594548e7ef820595217e7714f7994c533ec1;hb=437faa9675dd916ac7b239d4584b932a11fbb984#l83)，这个链表记录着所有被加载的模块以及它们的元数据。
```C
/* Structure describing a loaded shared object.  The `l_next' and `l_prev'
  members form a chain of all the shared objects loaded at startup.
  
  These data structures exist in space used by the run-time dynamic linker;
  modifying them may have disastrous results.
  
  This data structure might change in future, if necessary.  User-level
  programs must avoid defining objects of this type.  */
  
struct link_map
```
链接器将这些被加载的模块映射到进程的地址空间之后，将调用[init_tls](https://sourceware.org/git/?p=glibc.git;a=blob;f=elf/rtld.c;h=1b0c74739f967093d26f5867b9b9d552d8b1ad00;hb=437faa9675dd916ac7b239d4584b932a11fbb984#l681)，init_tls根据link_map来填充dl_tls_dtv_slotinfo_list，另一个全局的链接链表。dl_tls_dtv_slotinfo_list包含了使用了TLS的那些模块的元数据。然后init_tls会调用_dl_determine_tlsoffset，告诉各个被加载的模块，它们应该把它们的TLS块放在静态TLS的哪个偏移处。接着，init_tls调用_dl_allocate_tls_storage来分配静态TLS和DTV。

最终，动态链接器设置过程中会调用_dl_allocate_tls_init来最终初始化静态TLS以及主线程的DTV：
```C
void *
_dl_allocate_tls_init (void *result)
{
  if (result == NULL)
    /* The memory allocation failed.  */
    return NULL;

  dtv_t *dtv = GET_DTV (result);
  struct dtv_slotinfo_list *listp;
  size_t total = 0;
  size_t maxgen = 0;

  /* Check if the current dtv is big enough.   */
  if (dtv[-1].counter < GL(dl_tls_max_dtv_idx))
    {
      /* Resize the dtv.  */
      dtv = _dl_resize_dtv (dtv);

      /* Install this new dtv in the thread data structures.  */
      INSTALL_DTV (result, &dtv[-1]);
    }

  /* We have to prepare the dtv for all currently loaded modules using
     TLS.  For those which are dynamically loaded we add the values
     indicating deferred allocation.  */
  listp = GL(dl_tls_dtv_slotinfo_list);
  while (1)
    {
      size_t cnt;

      for (cnt = total == 0 ? 1 : 0; cnt < listp->len; ++cnt)
        {
          struct link_map *map;
          void *dest;

          /* Check for the total number of used slots.  */
          if (total + cnt > GL(dl_tls_max_dtv_idx))
            break;

          map = listp->slotinfo[cnt].map;

          /* snip */

          dest = (char *) result + map->l_tls_offset;

          /* Set up the DTV entry.  The simplified __tls_get_addr that
             some platforms use in static programs requires it.  */
          dtv[map->l_tls_modid].pointer.val = dest;

          /* Copy the initialization image and clear the BSS part.  */
          memset (__mempcpy (dest, map->l_tls_initimage,
                             map->l_tls_initimage_size), '\0',
                  map->l_tls_blocksize - map->l_tls_initimage_size);
        }

      total += cnt;
      if (total >= GL(dl_tls_max_dtv_idx))
        break;

      listp = listp->next;
    }

  /* The DTV version is up-to-date now.  */
  dtv[0].counter = maxgen;

  return result;
}

```
这个函数主要分两部分，第一部分检查DTV数组的大小，而第二部分则给DTV里的每个模块的TLS块设置一个合适的静态TLS的偏移(l_tls_offset)，并且将这些模块的TLS数据以及BSS（tdata and tbss）复制到静态TLS块中。

这就是TLS和DTV的设置过程。但是是谁来确定 -4 或者 0xfffffffffffffffc 就是那个可以正确获取前面的TLS变量main_tls_var的偏移呢?

答案在上面提到的那个规则中:主入口可执行程序的TLS块总是会被放在紧贴着TCB的地址处。实质上，是编译器以及动态链接器密谋了这一切：因为可执行文件的TLS块以及TLS偏移是提前知道的。编译器和链接器的合作基于以下事实：
1. FS寄存器指向TCB。
2. 编译器设置的TLS变量偏移在运行时不会被改变。
3. 主入口可执行程序的TLS块正好在TCB的前面。
编译器在构建期间就知道上述事实，而动态链接器在运行时确保这些事实是正确的。

### TLS access in executables and shared objects
你或许已经意识到了目前为止，我们只是讨论了可执行文件里面访问可执行文件自己定义的TLS变量的情况，其他情况下访问TLS变量时又会是怎么样呢？

显然答案与上面会有所不同，因为上面的讨论基于一个事实：编译器以及动态链接器都对主可执行程序的TLS做了特殊对待。依据一个TLS变量在何处定义以及在何处访问它，有以下四种情况：
1. TLS变量在可执行程序内部被定义、使用。
1. TLS变量在外部共享对象中定义，但是在可执行程序中使用。
1. TLS变量在同一个共享对象中定义、使用。
1. TLS变量在一个共享对象中定义，在另一个共享对象中使用。

下面对上述四种情况一个个来探讨。
#### Case1： TLS variable locally defined and used within an executable
这种情况上面已经讨论了很多了。提示：这种情况下访问TLS的形式是：mov fs:0xffffffffffxxxxxx %xxx。

#### Case2： TLS variable externally defined in a shared object but used in a executable
函数库里定义的TLS变量：
```C
// libfoo.c
__thread int foo_tls = 42;
```

主程序试图使用这个变量：
```C
// main.c

extern __thread int foo_tls;

int main() {
    return foo_tls;
}
```
反汇编这段main代码，结果可能是这样的：
```C
000000000000073a <main>:
 73a:   55                      push   %rbp
 73b:   48 89 e5                mov    %rsp,%rbp
 73e:   48 8b 05 93 08 20 00    mov    0x200893(%rip),%rax        # 200fd8 <foo_tls>
 745:   64 8b 00                mov    %fs:(%rax),%eax
 748:   5d                      pop    %rbp
 749:   c3                      retq
```
意料之中的是，访问TLS变量foo_tls和访问定义在主程序中的TLS变量main_tls_var的指令相似，但又不同。

依然是使用FS寄存器，然后，不再是使用一个负的常量偏移来获取变量地址，而是从地址0x200fd8处取出一个值，并将这个值作为相对TCB的偏移来访问TLS变量foo_tls。下面是伪代码：
```C
// Retrieve the offset of `foo_tls_offset`
int foo_tls_offset = *((int *) 0x200fd8);

// Pointer arithmetics to get the address of `foo_tls`
int *foo_tls_ptr = thread_context_ptr - foo_tls_offset;

// Dereference tls_var_ptr and put its value in register EAX
EAX = *foo_tls_ptr;
```
[这里](https://www.polarxiong.com/archives/x64%E4%B8%8BPIC%E7%9A%84%E6%96%B0%E5%AF%BB%E5%9D%80%E6%96%B9%E5%BC%8F-RIP%E7%9B%B8%E5%AF%B9%E5%AF%BB%E5%9D%80.html)是一篇介绍RIP寄存器的帖子。
```C
以上面那段代码为例：
rip寄存器的值是下一条指令的地址，即0x745。所以最终rax= 0x745 + 0x200893，即rax= 0x200fd8。
```

现在有个问题，为什么这里没有使用一个常量偏移0x200fd8来访问foo_tls？原因是在第一个case里面，编译器预先就知道了变量的偏移以及可执行文件的TLS块的地址。而在第二个case中，编译器只知道部分信息。
编译器知道这个TLS变量（foo_tls）将被存在static TLS的某个地址处。因为编译器知道foo_tls是在外部定义的，并且它知道foo_tls就定义在libfoo.so中，而libfoo.so是main可执行文件的**直接依赖项**。 **别忘了，static TLS包含了定义在可执行文件以及在可执行文件的main函数执行之前被加载的共享对象中的TLS变量。**

所以，编译器知道foo_tls就在static TLS中，但它不知道foo_tls在static TLS中的精确偏移（或相对TCB的偏移）。与可执行文件不同，动态链接器并不确保共享对象（library）被加载的先后顺序（也就是模块ID）。

必须有一种运行时机制来解决此问题，这也就是神秘的0x200fd8发挥作用的地方。

要深入这种运行时机制，就需要了解什么是**重定位**。[这里](https://eli.thegreenplace.net/2011/08/25/load-time-relocation-of-shared-libraries/)，[这里](https://eli.thegreenplace.net/2011/11/03/position-independent-code-pic-in-shared-libraries/)， [还有这里](https://eli.thegreenplace.net/2011/11/11/position-independent-code-pic-in-shared-libraries-on-x64)是几篇很棒的关于ELF重定位的文章。

如果你不打算耐心看完那几篇的话，这里有一个对重定位的概述：因为编译器并不知道定义在共享对象内部的变量在运行时的内存地址，因此需要重定位，并且编译器还会设置一个 Global Offset Table (GOT)。在运行时动态链接器会负责把合适的地址值填充进GOT。

回到关于foo_tls偏移的讨论：因为并不知道在外部定义的TLS变量foo_tls在静态TLS中的偏移，构建时编译器就会在GOT中创建一个条目，并且设置一个标记（[R_X86_64_TPOFF64](https://sourceware.org/git/?p=glibc.git;a=blob;f=elf/elf.h;h=7e2b072a7f75451c0faeb47013b6b93c2fc575a2;hb=437faa9675dd916ac7b239d4584b932a11fbb984#l3334)），表示这个偏移要怎么计算。

对可执行文件运行readelf来获取它的重定位dump信息：
```C
> readelf -r main

Relocation section '.rela.dyn' at offset 0x518 contains 9 entries:
  Offset          Info           Type           Sym. Value    Sym. Name + Addend
000000200dc8  000000000008 R_X86_64_RELATIVE                    730
000000200dd0  000000000008 R_X86_64_RELATIVE                    6f0
000000201008  000000000008 R_X86_64_RELATIVE                    201008
000000200fd0  000100000006 R_X86_64_GLOB_DAT 0000000000000000 _ITM_deregisterTMClone + 0
000000200fd8  000200000012 R_X86_64_TPOFF64  0000000000000000 foo_tls + 0
000000200fe0  000300000006 R_X86_64_GLOB_DAT 0000000000000000 __libc_start_main@GLIBC_2.2.5 + 0
000000200fe8  000400000006 R_X86_64_GLOB_DAT 0000000000000000 __gmon_start__ + 0
000000200ff0  000500000006 R_X86_64_GLOB_DAT 0000000000000000 _ITM_registerTMCloneTa + 0
000000200ff8  000600000006 R_X86_64_GLOB_DAT 0000000000000000 __cxa_finalize@GLIBC_2.2.5 + 0
```

注意foo_tls那条。它的偏移是000000200fd8，类型是R_X86_64_TPOFF64。

当运行时，动态链接器设置完TCB和静态TLS之后，就开始处理[重定位](https://sourceware.org/git/?p=glibc.git;a=blob;f=elf/rtld.c;h=1b0c74739f967093d26f5867b9b9d552d8b1ad00;hb=437faa9675dd916ac7b239d4584b932a11fbb984#l2197)。作为重定向中的一部分，运行时将[确定](https://sourceware.org/git/?p=glibc.git;a=blob;f=sysdeps/x86_64/dl-machine.h;h=1942ed5061d18c6802669cfd9e1c7f21133be3ff;hb=437faa9675dd916ac7b239d4584b932a11fbb984#l425)对于这种类型的重定向下这个变量的偏移值。
```C
case R_X86_64_TPOFF64:
  /* The offset is negative, forward from the thread pointer.  */
  if (sym != NULL)
    {
      CHECK_STATIC_TLS (map, sym_map);
      /* We know the offset of the object the symbol is contained in.
          It is a negative value which will be added to the
          thread pointer.  */
      value = (sym->st_value + reloc->r_addend
                - sym_map->l_tls_offset);
      *reloc_addr = value;
    }
  break;

```
我们着重看看计算偏移值的地方：
```C
value = sym->st_value          /* The offset of the value within symbol section */ 
      + reloc->addend          /* Zero, can be ignored for most cases in x86-64 */ 
      - sym_map->l_tls_offset; /* This is the module's TLS block offset within the static TLS */
```
在此之后，地址0x200fd8处的值将会是一个相对TCB（线程指针、寄存器指向的地址）的负偏移，并且被可执行文件用于访问foo_tls。

#### Case3： TLS variable locally defined in a shared object and used in the same shared object
可执行文件的讨论到此结束。现在开始看看共享库对象是如何访问TLS变量的。
```C
// libbar.c
static __thread int s_bar_tls;

int get_bar_tls() {
    return s_bar_tls;
}
```
请先确保禁用任何编译优化选项来编译上述代码文件。现在我们反编译libbar.so中的函数get_bar_tls：
```C
000000000000067a <get_bar_tls>:
 67a:   55                      push   %rbp
 67b:   48 89 e5                mov    %rsp,%rbp
 67e:   66 48 8d 3d 4a 09 20    lea 0x20094a(%rip),%rdi        # 200fd0 <.got>
 686:   66 66 48 e8 f2 fe ff    callq <__tls_get_addr@plt>
 68e:   8b 00                   mov    (%rax),%eax
 690:   5d                      pop    %rbp
 691:   c3                      retq 
```
这一次，访问s_bar_tls的代码与之前的完全不同。有一个".got"的提示，这表示GOT中的一个条目。且更重要的是，有一个[__tls_get_addr](https://sourceware.org/git/?p=glibc.git;a=blob;f=elf/dl-tls.c;h=c87caf13d6a97ba4981c5ddaba2ecec21f4753b0;hb=437faa9675dd916ac7b239d4584b932a11fbb984#l823) 的函数调用。

lea 0x20094a(%rip),%rdi 字面上看，这只是给寄存器%rdi设置了一个值0x200fd0（0x20094a + 0x686，GOT项的地址）。根据x86-64调用协议，%rdi被用于传递函数调用的第一个参数。

展开__tls_get_addr 函数体的部分宏后：
```C
void *
__tls_get_addr (tls_index *ti)
{
  dtv_t *dtv = THREAD_DTV ();

  if (__glibc_unlikely (dtv[0].counter != GL(dl_tls_generation)))
    return update_get_addr (ti);

  void *p = dtv[ti->ti_module].pointer.val;

  if (__glibc_unlikely (p == TLS_DTV_UNALLOCATED))
    return tls_get_addr_tail (ti->ti_offset, dtv, NULL);

  return (char *) p + ti->ti_offset;
}
```
可能你已经注意到了，上面的函数跟之前我们讨论过的利用DTV dtv[ti_module].pointer + ti_offset 来访问TLS变量的方式相似。本质上，__tls_get_addr做了同样的事情，除了有些额外的检查、更新代数目（generation）的代码，以及TLS块的懒惰分配。

我们知道get_bar_tls的汇编代码用指向GOT的一个项的指针作为参数调用了__tls_get_addr函数。那么0x200fd0地址处的那个GOT项里面存的是什么数据呢？

从glibc源码里面可以看到,__tls_get_addr的第一个参数是[struct tis_index](https://sourceware.org/git/?p=glibc.git;a=blob;f=sysdeps/x86_64/dl-tls.h;h=bc18e70b23f03fb9ce59b647b76f0c36e7e27afa;hb=437faa9675dd916ac7b239d4584b932a11fbb984), 所以自然而然地,那个地址处的数据必须是一个此结构体的实例?

为了确认这个,我们再次请出readelf:
```C
> readelf -r libbar.so 

Relocation section '.rela.dyn' at offset 0x478 contains 8 entries:
  Offset          Info           Type           Sym. Value    Sym. Name + Addend
000000200e00  000000000008 R_X86_64_RELATIVE                    670
000000200e08  000000000008 R_X86_64_RELATIVE                    630
000000201020  000000000008 R_X86_64_RELATIVE                    201020
000000200fd0  000000000010 R_X86_64_DTPMOD64                    0
000000200fe0  000100000006 R_X86_64_GLOB_DAT 0000000000000000 __cxa_finalize + 0
000000200fe8  000200000006 R_X86_64_GLOB_DAT 0000000000000000 _ITM_registerTMCloneTa + 0
000000200ff0  000300000006 R_X86_64_GLOB_DAT 0000000000000000 _ITM_deregisterTMClone + 0
000000200ff8  000500000006 R_X86_64_GLOB_DAT 0000000000000000 __gmon_start__ + 0

```
偏移000000200fd0处那一项是我们要追踪的数据。与foo_tls不同，这里没有显示出符号名，因为s_bar_tls是局部（locally/statically）定义的，它在符号表中的名字可以忽略。另一处与foo_tls不同的是它的类型：R_X86_64_DTPMOD64，而foo_tls的类型是R_X86_64_TPOFF64。

重定位时，类型为R_X86_64_DTPMOD64的项告诉动态链接器，它需要在运行时找出libbar.so的模块ID（l_tls_modid），并且把对应的模块ID值写入地址0x200fd0：
```C
case R_X86_64_DTPMOD64:
    /* Get the information from the link map returned by the
       resolve function.  */
    if (sym_map != NULL)
      *reloc_addr = sym_map->l_tls_modid;
    break;
```
眼尖的读者可能已经看到了，模块ID仅仅是struct tis_index的第一个成员：
```C
/* Type used for the representation of TLS information in the GOT.  */
typedef struct dl_tls_index
{
  uint64_t ti_module;
  uint64_t ti_offset;
} tls_index;
```
所以，ti_offset是什么时候，在哪里被初始化的？

请记着，bar_tls是静态的（statically defined），所以编译器可以知道它的TLS偏移，唯一不知道的是运行时这个共享对象的模块ID。所以实际上足够聪明的编译器在构建时期就已经把这个常量偏移直接写入GOT中了，紧挨着这个常量偏移ti_offset的是ti_module。所以，运行时期，只有模块ID需要被重定向。

为了核实这一点，我们可以引入多个局部定义的TLS变量：
```C
// libbar2.c

static __thread int s_bar_tls1;
static __thread int s_bar_tls2;
static __thread int s_bar_tls3;

int get_bar_tls() {
    return s_bar_tls1 + s_bar_tls2 + s_bar_tls3;
}
```
它的重定位信息里面包含三个R_X86_64_DTPMOD64类型的项，正如我们所预期的那样：
```C
> readelf -r libbar2.so 

Relocation section '.rela.dyn' at offset 0x478 contains 10 entries:
  Offset          Info           Type           Sym. Value    Sym. Name + Addend
000000200de0  000000000008 R_X86_64_RELATIVE                    6a0
000000200de8  000000000008 R_X86_64_RELATIVE                    660
000000201020  000000000008 R_X86_64_RELATIVE                    201020
000000200fb0  000000000010 R_X86_64_DTPMOD64                    0
000000200fc0  000000000010 R_X86_64_DTPMOD64                    0
000000200fd0  000000000010 R_X86_64_DTPMOD64                    0
000000200fe0  000100000006 R_X86_64_GLOB_DAT 0000000000000000 __cxa_finalize + 0
000000200fe8  000200000006 R_X86_64_GLOB_DAT 0000000000000000 _ITM_registerTMCloneTa + 0
000000200ff0  000300000006 R_X86_64_GLOB_DAT 0000000000000000 _ITM_deregisterTMClone + 0
000000200ff8  000500000006 R_X86_64_GLOB_DAT 0000000000000000 __gmon_start__ + 0

Relocation section '.rela.plt' at offset 0x568 contains 1 entry:
  Offset          Info           Type           Sym. Value    Sym. Name + Addend
000000201018  000400000007 R_X86_64_JUMP_SLO 0000000000000000 __tls_get_addr@GLIBC_2.3 + 0
```
然后检查其GOT区：
```C
> readelf -x .got libbar2.so 

Hex dump of section '.got':
  0x00200fb0 00000000 00000000 00000000 00000000 ................
  0x00200fc0 00000000 00000000 04000000 00000000 ................
  0x00200fd0 00000000 00000000 08000000 00000000 ................
  0x00200fe0 00000000 00000000 00000000 00000000 ................
  0x00200ff0 00000000 00000000 00000000 00000000 ................
```
注意前三行的第4、5两列：00000000 00000000, 04000000 00000000, 08000000 00000000。它们是小端对齐的64位的整数0,4,8，对应这三个TLS变量的三个偏移值。

#### Case 4: TLS variable externally defined in a shared object and used in an arbitrary shared object
这个case与case3相似，区别只是TLS变量的定义和使用在不同的地方。
```C
// libxyz.c

// xyz_tls is used in another shared object
__thread int xyz_tls;
```

```C
// libuvw.c

extern __thread int xyz_tls;

int get_xyz_tls() {
    return xyz_tls;
}

```
使用-O0选项编译这个文件，接下来，反汇编：
```C
000000000000066a <get_xyz_tls>:
 66a:   55                      push   %rbp
 66b:   48 89 e5                mov    %rsp,%rbp
 66e:   66 48 8d 3d 5a 09 20    lea 0x20095a(%rip),%rdi        # 200fd0 <xyz_tls>
 676:   66 66 48 e8 f2 fe ff    callq <__tls_get_addr@plt>
 67e:   8b 00                   mov    (%rax),%eax
 680:   5d                      pop    %rbp
 681:   c3                      retq   
```
有趣的是，除地址之外，这段代码看起来与get_bar_tls一样。

那readelf又会怎么样呢？
```C
> readelf -r libuvw.so 

Relocation section '.rela.dyn' at offset 0x458 contains 9 entries:
  Offset          Info           Type           Sym. Value    Sym. Name + Addend
000000200e00  000000000008 R_X86_64_RELATIVE                    660
000000200e08  000000000008 R_X86_64_RELATIVE                    620
000000201020  000000000008 R_X86_64_RELATIVE                    201020
000000200fd0  000100000010 R_X86_64_DTPMOD64 0000000000000000 xyz_tls + 0
000000200fd8  000100000011 R_X86_64_DTPOFF64 0000000000000000 xyz_tls + 0
000000200fe0  000200000006 R_X86_64_GLOB_DAT 0000000000000000 __cxa_finalize + 0
000000200fe8  000300000006 R_X86_64_GLOB_DAT 0000000000000000 _ITM_registerTMCloneTa + 0
000000200ff0  000400000006 R_X86_64_GLOB_DAT 0000000000000000 _ITM_deregisterTMClone + 0
000000200ff8  000600000006 R_X86_64_GLOB_DAT 0000000000000000 __gmon_start__ + 0

Relocation section '.rela.plt' at offset 0x530 contains 1 entry:
  Offset          Info           Type           Sym. Value    Sym. Name + Addend
000000201018  000500000007 R_X86_64_JUMP_SLO 0000000000000000 __tls_get_addr@GLIBC_2.3 + 0
```
这一次我们看到符号名那列显示了xyz_tls，因为它们是由外部定义的。更有趣的是，除了此前我们看到过的R_X86_64_DTPMOD64 之外，这里还出现了另一种R_X86_64_DTPOFF64类型的项。

因为xyz_tls是在另一个共享对象中定义的，编译器既不知道它的模块ID，也不知道它在模块的TLS块中的偏移。除了R_X86_64_DTPMOD64，TLS偏移值不再是编译期间可知的一个常量了，编译器需要动态链接器提供更多的帮助。

R_X86_64_DTPOFF64正是额外的帮助。这个重定位类型意味着运行时需要计算xyz_tls在它的模块的TLS块中的偏移：
```C
case R_X86_64_DTPOFF64:
    /* During relocation all TLS symbols are defined and used.
       Therefore the offset is already correct.  */
    if (sym != NULL)
      {
        value = sym->st_value + reloc->r_addend;
        *reloc_addr = value;
      }
    break;

```
所以，对于这个例子，每个TLS变量都需要两个重定位的步骤来填充足够可用的信息。

至此，四种使用TLS变量的情形都已经解析过了。到这里，你可能会问，为什么这么复杂？最简单、最常规的访问TLS变量的方式是使用__tls_get_addr，但是为了在给定条件下取得最快的执行速度，glibc设计者牺牲了简便性来获取性能。

当然，通过线程寄存器（FS）访问TLS变量，被认为比__tls_get_addr更快（单次内存访问，且对缓存更加友好）。因此，如果访问TLS的速度对你的应用程序很重要的话，请尝试在main可执行程序中定义以及使用TLS变量。如果不行，请至少尝试在main可执行程序中使用TLS变量，这种情况下，至少还能让TLS变量被放在静态TLS中。

### The Initialisation of TCB or TLS in Non-main Threads
非主线程的线程的TCB和TLS设置有些不同。当调用pthread_create来创建一个新线程时，首先要[分配一个新的栈](https://sourceware.org/git/?p=glibc.git;a=blob;f=nptl/pthread_create.c;h=fe75d04113b8aa3f8c606acb4f4a4e3675d103f1;hb=437faa9675dd916ac7b239d4584b932a11fbb984#l669)。

[allocate_stack](https://sourceware.org/git/?p=glibc.git;a=blob;f=nptl/allocatestack.c;h=04e3f08465ed9982be98535ca6d81d6784d9eb4b;hb=437faa9675dd916ac7b239d4584b932a11fbb984#l407)首先[检查](https://sourceware.org/git/?p=glibc.git;a=blob;f=nptl/allocatestack.c;h=04e3f08465ed9982be98535ca6d81d6784d9eb4b;hb=437faa9675dd916ac7b239d4584b932a11fbb984#l546)是否已有缓存好的栈，如果缓存里没有合适的栈，就分配一个新的。

有意思的是，当获取到了一个新栈之后，allocate_stack 将在栈空间中初始化 static TLS和TCB。也就是说，主线程的TCB和static TLS内存是分配在堆上；与此不同的是，非主线程的线程使用栈来存储TLS数据，减少内存分配（次数）。

设置完栈和TLS之后，pthread_create 会触发clone系统调用：
```C
const int clone_flags = (CLONE_VM | CLONE_FS | CLONE_FILES | CLONE_SYSVSEM
                           | CLONE_SIGHAND | CLONE_THREAD
                           | CLONE_SETTLS | CLONE_PARENT_SETTID
                           | CLONE_CHILD_CLEARTID
                           | 0);

TLS_DEFINE_INIT_TP (tp, pd);

if (__glibc_unlikely (ARCH_CLONE (&start_thread, STACK_VARIABLES_ARGS,
                                  clone_flags, pd, &pd->tid, tp, &pd->tid)
                      == -1))
  return errno;

```
在x86-64平台上，tp和pd是相同的:
```C
 # define TLS_DEFINE_INIT_TP(tp, pd) void *tp = (pd)
```
它们指向TCB（即struct pthread）。注意线程指针（thread pointer, tp pointing to TCB）是clone系统调用的第六个参数。

查看[clone的源码](https://elixir.bootlin.com/linux/v4.18.20/source/kernel/fork.c#L2230)，可以看到它只是调用了_do_fork，并且把第六个参数tls传给了_do_fork。追踪参数tls，可以看到通过依次调用_do_fork，[copy_process](https://elixir.bootlin.com/linux/v4.18.20/source/kernel/fork.c#L1605)，[copy_thread_tls](https://elixir.bootlin.com/linux/v4.18.20/source/arch/x86/kernel/process_64.c#L289)，最终把tls绑定到线程的FS寄存器中。