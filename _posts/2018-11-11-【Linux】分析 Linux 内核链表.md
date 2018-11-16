---
layout:         page
title:         【Linux】分析 Linux 内核双向链表（内核版本v4.0）
subtitle:       从内核源码看Linux怎么实现双向链表
date:           2018-11-11
author:         翼
header-img: image/bg.jpg
catalog: true
tags:
---

### Linux 内核双向链表数据结构
通常我们实现双向链表时，会定义如下的结构：
```C
struct list_node {
    struct list_node *prev, *next;
    elemment e;
};

```
这是数据结构教材上定义双向链表的经典方式：数据被包含在链表节点结构中。然后每一个数据类型的链表都要定义一个类似的 node 结构。当然，C++可以用模板实现链表，抽象出类型无关的链表操作接口，正如STL中的std::\<list\>。

Linux内核却不是这么定义的。Linux内核定义的双向链表数据结构如下：
```C

/* /include/linux/types.h */

struct list_head {
	struct list_head *next, *prev;
};

```

在内核中用链表组织的数据结构内部通常都有一个 struct list_head 成员。比如，文件系统的超级块结构 **super_block**：
```C

/* /include/linux/fs.h */

struct super_block {
	struct list_head	s_list;
	dev_t			s_dev;
	unsigned char		s_blocksize_bits;
	unsigned long		s_blocksize;
	loff_t			s_maxbytes;
	struct file_system_type	*s_type;
	const struct super_operations	*s_op;

	//...


};

```
上面的 s_list 就把Linux系统中的所有文件系统的所有超级块组织成了一个链表。

### 链表的构建过程 ：链表操作接口
#### 链表初始化
有了 struct list_head 结构，如何构建链表呢？或者说，如何初始化一个空链表呢？

这涉及到内核定义的两个宏：
```C

/* /include/linux/list.h */

#define LIST_HEAD_INIT(name) { &(name), &(name) }

#define LIST_HEAD(name) \
	struct list_head name = LIST_HEAD_INIT(name)

```
还是以上面的超级块链表为例，看看这个链表的构建。
```C

/* /fs/super.c */

static LIST_HEAD(super_blocks);

```
在super.c文件中，内核定义了一个static的全局变量 super_blocks。展开后的代码是这样：
```C

static struct list_head super_blocks = LIST_HEAD_INIT(super_blocks);

```
继续展开，代码如下：
```C

static struct list_head super_blocks = { &(super_blocks), &(super_blocks) };

```
可见，实际上，内核定义了一个 struct list_head 的变量 super_blocks，并且把 next 和 prev 字段都指向 super_blocks自己。
这其实就是定义了一个**空链表super_blocks**：
```C

/* /include/linux/list.h */

/**
 * list_empty - tests whether a list is empty
 * @head: the list to test.
 */
static inline int list_empty(const struct list_head *head)
{
	return head->next == head;
}

```
list_empty接口就是通过判断 struct list_head 结构的 next 字段是否指向 链表对象本身，来判断该链表是否为空。

除了通过 LIST_HEAD宏来初始化一个链表，内核还定义了一个 inline 函数 **INIT_LIST_HEAD**：
```C

/* /include/linux/list.h */

static inline void INIT_LIST_HEAD(struct list_head *list)
{
	list->next = list;
	list->prev = list;
}

```
无论是通过 INIT_LIST_HEAD内联函数还是LIST_HEAD宏来初始化一个链表，结果都一样。

#### 链表的操作
**插入**

内核实现了 list_add 和 list_add_tail 两个接口来在链表头和尾插入数据。
```C

/* /include/linux/list.h */

static inline void __list_add(struct list_head *new,
			      struct list_head *prev,
			      struct list_head *next)
{
	next->prev = new;
	new->next = next;
	new->prev = prev;
	prev->next = new;
}

/**
 * list_add - add a new entry
 * @new: new entry to be added
 * @head: list head to add it after
 *
 * Insert a new entry after the specified head.
 * This is good for implementing stacks.
 */
static inline void list_add(struct list_head *new, struct list_head *head)
{
	__list_add(new, head, head->next);
}


/**
 * list_add_tail - add a new entry
 * @new: new entry to be added
 * @head: list head to add it before
 *
 * Insert a new entry before the specified head.
 * This is useful for implementing queues.
 */
static inline void list_add_tail(struct list_head *new, struct list_head *head)
{
	__list_add(new, head->prev, head);
}

```
实际上，内核的这个链表是循环链表，表头的next、prev分别指向链表中的第一个和最后一个node。比如，在mount的时候，内核会调用一个函数sget来查找或者创建一个超级块，并添加到super_blocks链表中。sget的定义如下：
```C

/* /fs/super.c */

struct super_block *sget(struct file_system_type *type,
			int (*test)(struct super_block *,void *),
			int (*set)(struct super_block *,void *),
			int flags,
			void *data)
{
    struct super_block *s = NULL;

	/* ... */

	list_add_tail(&s->s_list, &super_blocks);
	
    /* ... */

	return s;
}

```
也就是说，在挂载一个文件系统的时候，内核会把该文件系统的超级块添加到super_blocks链表的尾部，也就是super_blocks的前一个位置。这里也可以看出，super_blocks链表是一个**带哨兵的双向循环链表**。Linux内核中有多处使用到这种链表数据结构。

super_blocks双向循环链表的示意图如下：

![super_blocks](https://raw.githubusercontent.com/ttyrion/ttyrion.github.io/master/image/linux/kernel/linked-list/super_blocks.png) 

其他的删除、移动、合并等等并没有什么特别，后面重点分析一下内核链表的遍历。

#### 链表的遍历
链表的遍历可以说是链表最常见的操作了。

##### 如何访问内核的链表的数据项？
内核定义了一系列的宏来访问链表数据，这些宏包括：**list_entry**, **list_first_entry**, **list_last_entry**, **list_first_entry_or_null**, **list_next_entry**, **list_prev_entry**等等。从名字基本可以看出它们的作用，**list_entry**是最基本的，其他几个宏都是在**宏list_entry**的基础上实现的。下面就分析**list_entry**是如何访问链表数据的。
```C

/* /include/linux/list.h */

/**
 * list_entry - get the struct for this entry
 * @ptr:	the &struct list_head pointer.
 * @type:	the type of the struct this is embedded in.
 * @member:	the name of the list_head within the struct.
 */
#define list_entry(ptr, type, member) \
	container_of(ptr, type, member)

```
**container_of**也是一个宏，其定义在kernel.h头文件中：
```C

/* /include/linux/kernel.h */

/**
 * container_of - cast a member of a structure out to the containing structure
 * @ptr:	the pointer to the member.
 * @type:	the type of the container struct this is embedded in.
 * @member:	the name of the member within the struct.
 */
#define container_of(ptr, type, member) ({			\
	const typeof( ((type *)0)->member ) *__mptr = (ptr);	\
	(type *)( (char *)__mptr - offsetof(type,member) );})

```
**offsetof**也是一个宏，其定义在stddef.h中：
```C

/* /include/linux/stddef.h */

#undef offsetof
#ifdef __compiler_offsetof
#define offsetof(TYPE,MEMBER) __compiler_offsetof(TYPE,MEMBER)
#else
#define offsetof(TYPE, MEMBER) ((size_t) &((TYPE *)0)->MEMBER)
#endif
#endif

```

看container_of 和 offsetof 这两个宏定义的位置，就知道它们并不仅仅用于链表操作。先看offsetof宏，它取一个TYPE对象中MEMBER的地址，而实际上因为这里是把 **0** 强制转换为 **TYPE \***，也就是说那个TYPE对象自己的地址是0，所以最终MEMBER的地址就是它在该结构体中的偏移。也就是说offsetof(TYPE, MEMBER)得到的就是MEMBER字段在TYPE结构体中的偏移。

有了MEMBER在TYPE中的偏移，以及一个MEMBER对象地址，就能得到对应的TYPE对象的地址。container_of(ptr, type, member)就是利用这个根据 ptr 得到包含（*ptr）对象（在内核这个链表中就是一个struct list_head 对象）的那个type对象的地址，具体是这样的：

container_of宏利用了typeof特性，typeof是 GNU C 提供的扩展特性（不属于ISO C 的特性），它可以获取一个表达式或者类型的类型（有点拗口？英文描述：typeof is used to refer to the type of a type or an expression.）。 container_of 把第一个参数ptr转换成第三个参数member的类型的指针，然后利用offsetof宏，利用结构体的字段偏移地址计算出包含ptr指向的对象的那个type对象的地址。

到这里，就知道内核是如何访问链表数据项的了。比如我想访问super_blocks链表表头的s_dev字段：
```C

struct super_block * psb = list_entry(super_blocks.next, struct super_block, s_list);
psb->s_dev;
...

```

##### 如何遍历链表？
内核中定义了一系列遍历链表的宏，比如向前遍历链表的 list_for_each，向后遍历链表的 list_for_each_prev：
```C

/* /include/linux/list.h */

/**
 * list_for_each	-	iterate over a list
 * @pos:	the &struct list_head to use as a loop cursor.
 * @head:	the head for your list.
 */
#define list_for_each(pos, head) \
	for (pos = (head)->next; pos != (head); pos = pos->next)

/**
 * list_for_each_prev	-	iterate over a list backwards
 * @pos:	the &struct list_head to use as a loop cursor.
 * @head:	the head for your list.
 */
#define list_for_each_prev(pos, head) \
	for (pos = (head)->prev; pos != (head); pos = pos->prev)

```
可以看到，这些宏就是一个for循环。以遍历super_blocks为例，我们可以这样：
```C

struct list_head *iter = NULL;
list_for_each(iter, &super_blocks) {
    struct super_block * psb = list_entry(iter, struct super_block, s_list);
    psb->s_dev;
    ...
}

```

也就是说我们通过list_for_each遍历链表，通过list_entry访问表项。为了简化我们上面的代码，内核同时定义了另外两个宏：

可以看到，这些宏就是一个for循环。以遍历super_blocks为例，我们可以这样：
```C

/* /include/linux/list.h */

/**
 * list_for_each_entry	-	iterate over list of given type
 * @pos:	the type * to use as a loop cursor.
 * @head:	the head for your list.
 * @member:	the name of the list_head within the struct.
 */
#define list_for_each_entry(pos, head, member)				\
	for (pos = list_first_entry(head, typeof(*pos), member);	\
	     &pos->member != (head);					\
	     pos = list_next_entry(pos, member))

/**
 * list_for_each_entry_reverse - iterate backwards over list of given type.
 * @pos:	the type * to use as a loop cursor.
 * @head:	the head for your list.
 * @member:	the name of the list_head within the struct.
 */
#define list_for_each_entry_reverse(pos, head, member)			\
	for (pos = list_last_entry(head, typeof(*pos), member);		\
	     &pos->member != (head); 					\
	     pos = list_prev_entry(pos, member))

```
利用上面的宏，我们的遍历代码可以更简洁一点：

```C

struct super_block *iter = NULL;
list_for_each_entry(iter, &super_blocks, s_list) {
    iter->s_dev;
    ...
}

```

##### 安全性问题
上面的链表操作接口并没有考虑在并行执行的环境下的问题，因此实际上调用这些接口的地方应该自己保证正确地进行并行访问。实际上，以上面的超级块为例，内核在定义超级块链表super_blocks的同时，也定义了一个自旋锁sb_lock:
```C

/* /fs/super.c */

static LIST_HEAD(super_blocks);
static DEFINE_SPINLOCK(sb_lock);


```
内核在访问超级块链表的时候会对sb_lock加锁。

虽然链表操作接口没有锁机制，不过内核还是定义了几个更安全的接口：
```C

/* /include/linux/list.h */


/**
 * list_for_each_safe - iterate over a list safe against removal of list entry
 * @pos:	the &struct list_head to use as a loop cursor.
 * @n:		another &struct list_head to use as temporary storage
 * @head:	the head for your list.
 */
#define list_for_each_safe(pos, n, head) \
	for (pos = (head)->next, n = pos->next; pos != (head); \
		pos = n, n = pos->next)

/**
 * list_for_each_prev_safe - iterate over a list backwards safe against removal of list entry
 * @pos:	the &struct list_head to use as a loop cursor.
 * @n:		another &struct list_head to use as temporary storage
 * @head:	the head for your list.
 */
#define list_for_each_prev_safe(pos, n, head) \
	for (pos = (head)->prev, n = pos->prev; \
	     pos != (head); \
	     pos = n, n = pos->prev)

```
正如注释中说的，这两个接口只是额外处理了一种情况：遍历链表的时候删除了pos节点。这种情况下，如果是调用不带safe后缀的接口，链表的遍历操作就会中断。带safe后缀的接口仅仅只是解决了这个问题，很明显，它们的安全性依然很有限。