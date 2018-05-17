title: Linux内核疑点若干
date: 2015-09-27 10:30:53
tags: Linux内核
---
本文涉及的内核代码版本为2.4.0。（其余文章如不作特殊说明，也使用该版本代码。）

<!-- more -->

## 1. get\_fs()/set\_fs()
这两个函数乍一看似乎用来控制fs寄存器的值。如果不细看其内部实现，很容易被误导。比如fs/namei.c中有如下代码，segment都用来描述了，很容易被误导。

```c
    if (!segment_eq(get_fs(), KERNEL_DS))
        return -EFAULT;
```

其实这两个函数在内核平台无关代码中有调用，因此绝不可能用来表示一个平台相关的概念。(fs寄存器是intel特有的)

还好，内核代码中有说明为什么要这么命名。

```c
/*
 * The fs value determines whether argument validity checking should be
 * performed or not.  If get_fs() == USER_DS, checking is performed, with
 * get_fs() == KERNEL_DS, checking is bypassed.
 *
 * For historical reasons, these macros are grossly misnamed.
 */

#define MAKE_MM_SEG(s)	((mm_segment_t) { (s) })


#define KERNEL_DS	MAKE_MM_SEG(0xFFFFFFFF)
#define USER_DS		MAKE_MM_SEG(PAGE_OFFSET)

#define get_ds()	(KERNEL_DS)
#define get_fs()	(current->addr_limit)
#define set_fs(x)	(current->addr_limit = (x))
```
注释中说的很清楚，这个fs仅用来判断是否要验参数。而且由于历史原因，这个晕死人不偿命的名称就一直沿用下来。

举个例子，

```c
/*
 * Like copy_strings, but get argv and its values from kernel memory.
 */
int copy_strings_kernel(int argc,char ** argv, struct linux_binprm *bprm)
{
	int r;
	mm_segment_t oldfs = get_fs();
	set_fs(KERNEL_DS); 
	r = copy_strings(argc, argv, bprm);
	set_fs(oldfs);
	return r; 
}
```
函数copy\_strings\_kernel()在do_execve()中会被调用，用来拷贝内核中的字符串到内核空间。这边复用了从用户空间拷贝字符串到内核空间的函数copy\_strings()。但是调用之前需要把fs设置为KERNEL\_DS。因为copy\_strings()当中会有检查字符串地址是否小于current进程的addr\_limit，而内核空间地址在用户空间之上，如果不设置fs为KERNEL\_DS，则检查必然失败。（为什么要检查？如果内核代码对用户空间地址随意放行，用户程序就可以钻空子任意妄为了。）

例如，copy\_strings()中有如下检查，

```c
// copy_strings() -> strnlen_user() -> __addr_ok()
#define __addr_ok(addr) ((unsigned long)(addr) < (current->addr_limit.seg))
```

## 2. 内核代码中的物理地址和虚拟地址

## 3. get\_pte\_fast()

## 4. 内核地址空间页表如何映射到用户进程