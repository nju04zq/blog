title: linux内核x86汇编小结
date: 2015-09-10 15:08:22
tags: [linux内核, 汇编]
---
汇编在linux内核中比重不大，但是很难啃。一部分原因在于汇编指令，某些有段时间不看就忘记了。另外一部分原因是C中内联汇编比较难懂。这里做个小结，方便以后复习汇编知识。

<!-- more -->

# x86汇编指令

关于x86指令，GNU使用的和INTEL文档里面的有些不同。一个区别是，对于双字节，GNU用l，INTEL用d。 比如INTEL文档里面的指令MOVSD，对应GNU的版本就是movsl。[这里](https://docs.oracle.com/cd/E19120-01/open.solaris/817-5477/index.html)有GNU格式汇编的文档。

## MOVSB/MOVSW/MOVSD/MOVSQ

MOVB指令将ESI中内存地址存放的内容（单字节）复制到EDI指向的内存区域，然后将ESI/EDI中的地址+1（DF标志位为0）或者-1（DF标志位为1）。

其余指令做类似的事情，除了复制的内容大小有区别

## STOSB/STOSW/STOSD/STOSQ—Store String

STOSB指令将寄存器AL中的内容复制到EDI中的目的地址，然后将EDI中地址+1（DF标志位为0）或者-1（DF标志位为1）。

其余指令做类似的事情，除了复制的内容大小有区别

## LODSB/LODSW/LODSD/LODSQ—Load String

LODSB指令将EDI指向的内容复制到AL，然后将ESI中地址+1（DF标志位为0）或者-1（DF标志位为1）。

其余指令做类似的事情，除了复制的内容大小和目的寄存器有区别

## SCASB/SCASW/SCASD—Scan String

SCASB指令将寄存器中AL的内容和EDI中目的内存地址对应的内容比较，并设置相应的标志位。然后将EDI中地址+1（DF标志位为0）或者-1（DF标志位为1）。

其余指令做类似的事情，除了复制的内容大小有区别

## REP/REPE/REPZ/REPNE/REPNZ

REP前缀主要配合这些指令，INS, OUTS, MOVS, LODS, STOS，其余的主要配合如CMPS, SCAS(REP本身不是一个独立指令)。

REP前缀的执行逻辑可以用如下伪代码描述

```c
while ECX != 0:
    Execute associated string instruction
    ECX -= 1
    if ECX == 0:
        break
    if FLAG meets prefix: #for REPE/REPZ/REPZE/REPNZ
        break
```
下面这个表格列举了REP族指令的结束条件

| Repeat Prefix|Termination Condition 1|Termination Condition 2|
| ------------- |:--------------------:|:------:|
| REP           | RCX or (E)CX = 0     | None   |
| REPE/REPZ     | RCX or (E)CX = 0     | ZF = 0 |
| REPNE/REPNZ   | RCX or (E)CX = 0     | ZF = 1 |

## LEA—Load Effective Address

LEA指令的主要作用是做地址计算。 GNU汇编中的寻址写法是，section:disp(base, index, scale)，对应地址section:(base+index*scale+disp)。其中，base和index必须是通用寄存器，scale取值必须是1,2,4,8中之一。

下面一段内联汇编程序说明了LEA的用法，以及和MOV指令的区别。（gdb中可以通过命令disassembly/ni/info register xxx调试汇编）

```c
asm("xorl %ecx, %ecx");
asm("lea (%ebp), %ecx");    //%ecx = %ebp
asm("xorl %ecx, %ecx");
asm("movl %ebp, %ecx");     //%ecx = %ebp
asm("xorl %ecx, %ecx");
asm("lea -4(%ebp), %ecx");  //%ecx = %ebp-4
asm("xorl %ecx, %ecx");
asm("movl (%ebp), %ecx");   //%ecx = *%ebp
```

LEA指令另外一个比较常见的用法是，做一些简单的计算。比如，

```c
asm("movl $100, %ecx");
asm("movl $10, %edx");
asm("lea 1(%ecx, %edx, 4), %ecx"); //%ecx = %ecx+%ebx*4+1
```

# 内联汇编

内联汇编的基本用法可以参考linux内核情景分析和[这篇文章](http://www.ibm.com/developerworks/library/l-ia/index.html)。 这篇[howto](http://www.ibiblio.org/gferg/ldp/GCC-Inline-Assembly-HOWTO.html)则详细介绍了内联汇编。

内联汇编的格式如下

```c
asm(assembler template
    : output operands             (optional, 输出部)
    : input operands              (optional, 输入部)
    : list of clobbered registers (optional, 损坏部)
    );
```

输入输出部里面的寄存器或者内存标示符参考如下表格（只列举了常见的）

| 标示符         | 含义                         |
| ------------- |-----------------------------|
| a, b, c, d    | eax, ebx, ecx, edx          |
| q             | one in eax/ebx/ecx/edx      |
| S, D          | esi, edi                    |
| r             | any register                |
| &             | Make sure input/ouput in different register|
| m             | Read from memory            |
| i             | An immediate integer        |

## 几个例子

### strlen

```c
static inline size_t strlen(const char * s)
{
int d0;
register int __res;
__asm__ __volatile__(
	"repne\n\t"    //循环至al != *edi, 循环中edi++, ecx--
	"scasb\n\t"
	"notl %0\n\t"  //ecx的值取反，想当于0xffffffff-ecx，对于空字符串，ecx现在为1
	"decl %0"      //循环中ecx在检查flag之前递减，这边需要补偿
	:"=c" (__res), "=&D" (d0) // __res=ecx, d0=edi
	:"1" (s),"a" (0), "0" (0xffffffff)); //esi=s, eax=0, ecx= 0xffffffff
return __res;
}
```

### strnlen\_user

```c
/*
 * Return the size of a string (including the ending 0)
 *
 * Return 0 on exception, a value greater than N if too long
 */
//要注意看注释，返回的是字符串大小，包括\0
//当然这种函数名本身是个奇葩
//如果strlen(s)>=n, 返回n+1, 如果strlen(s)<n, 返回strlen(s)+1
//错误情况下返回0
long strnlen_user(const char *s, long n) //n是最大长度
{
    //addr ok, mask=0xffffffff, otherwise 0x0
    unsigned long mask = -__addr_ok(s);
    unsigned long res, tmp;

    __asm__ __volatile__(
        "    andl %0,%%ecx\n" //ecx &= n, 如mask==0则ecx=0, 否则ecx=n
        "0:  repne; scasb\n"  //循环至*edi==0或ecx == 0
        "    setne %%al\n"    //如因ecx==0循环结束, 则al=1, 否则al=0
                              //注意repne不会设置标志位
        "    subl %%ecx,%0\n" //n -= ecx, for empty string, n=1
        "    addl %0,%%eax\n" //eax += n, 如果strlen>=n, 则返回n+1
        "1:\n"
        ".section .fixup,\"ax\"\n"
        "2:  xorl %%eax,%%eax\n"
        "    jmp 1b\n"
        ".previous\n"
        ".section __ex_table,\"a\"\n"
        "    .align 4\n"
        "    .long 0b,2b\n"
        ".previous"
        :"=r" (n), "=D" (s), "=a" (res), "=c" (tmp)
        //输出res=eax,其余输出没有作用
        :"0" (n), "1" (s), "2" (0), "3" (mask)
        //输入%0=n, edi=s, eax=0, ecx=mask
        :"cc");
    return res & mask;
}
```

### \_\_copy\_user\_zeroing

```c
#define __copy_user_zeroing(to,from,size)                \
do {                              \
    int __d0, __d1;               \
    __asm__ __volatile__(         \
        "0:  rep; movsl\n"        \ /* 拷贝ecx个四字节 */
        "    movl %3,%0\n"        \ /* ecx=剩余单字节数目 */
        "1:  rep; movsb\n"        \ /* 拷贝ecx个单字节 */
        "2:\n"                    \
        ".section .fixup,\"ax\"\n"\ /* 拷贝时候出问题 */
        "3:  lea 0(%3,%0,4),%0\n" \ /* ecx=ecx*4+%3，剩余未拷贝字节数目 */
        "4:  pushl %0\n"          \ /* ecx在下面的循环中会改变，先保存 */
        "    pushl %%eax\n"       \
        "    xorl %%eax,%%eax\n"  \
        "    rep; stosb\n"        \ /* 把目的内存剩余部分填0 */
        "    popl %%eax\n"        \
        "    popl %0\n"           \ /* 返回未拷贝的字节数 */
        "    jmp 2b\n"            \
        ".previous\n"             \
        ".section __ex_table,\"a\"\n"\
        "    .align 4\n"          \
        "    .long 0b,3b\n"       \
        "    .long 1b,4b\n"       \
        ".previous"               \
        : "=&c"(size), "=&D" (__d0), "=&S" (__d1)        \
        /* 输出size=ecx */
        : "r"(size & 3), "0"(size / 4), "1"(to), "2"(from)    \
        /* 输入%3=单字节数目，ecx=四字节数目，edi=to, esi=from */
        : "memory");                        \
} while (0)
```

### \_\_do\_strncpy\_from\_user

```c
#define __do_strncpy_from_user(dst,src,count,res) \
do {\
    int __d0, __d1, __d2;            \
    __asm__ __volatile__(            \
        "    testl %1,%1\n"          \ /* 待拷贝字符串长度是否为0 */
        "    jz 2f\n"                \ /* 待拷贝字符串长度为0则直接退出 */
        "0:  lodsb\n"                \ /* al  = *(esi++) */
        "    stosb\n"                \ /* *(edi++) = al */
        "    testb %%al,%%al\n"      \ /* if al == 0? */
        "    jz 1f\n"                \ /* if al == 0： goto 1f */
        "    decl %1\n"              \ /* ecx-- */
        "    jnz 0b\n"               \ /* if ecx > 0: goto 0b */
        "1:  subl %1,%0\n"           \ /* edx -= ecx */
        "2:\n"                       \
        ".section .fixup,\"ax\"\n"   \ /* 如果拷贝的时候出现问题 */
        "3:  movl %5,%0\n"           \ /* edx = -EFAULT */
        "    jmp 2b\n"               \ /* 退出 */
        ".previous\n"                \
        ".section __ex_table,\"a\"\n"\
        "    .align 4\n"             \
        "    .long 0b,3b\n"          \
        ".previous"                  \
        : "=d"(res), "=c"(count), "=&a" (__d0), "=&S" (__d1),\
          "=&D" (__d2)\
          /* 输出res=edx, count=ecx */
        : "i"(-EFAULT), "0"(count), "1"(count), "3"(src), "4"(dst) \
          /* 输入%5=-EFAULT, edx=count, edx=count, esi=src, edi=dst */
        : "memory");\
} while (0)
```

### strchr

```c
static inline char * strchr(const char * s, int c)
{
    int d0;
    register char * __res;
    __asm__ __volatile__(
        "movb %%al,%%ah\n"        //ah=目标字符
        "1: lodsb\n\t"            //al=*(esi++)
        "   cmpb %%ah, %%al\n\t"  //ah == al?
        "   je 2f\n\t"            //if ah == al: goto 2f
        "   testb %%al, %%al\n\t" //al == ‘0’？
        "   jne 1b\n\t"           //if al != '0': goto 1f
        "   movl $1,%1\n"         //eax=1, return 0
        "2: movl %1,%0\n\t"       //eax=esi, return esi-1
        "   decl %0"              //eax--
        :"=a" (__res), "=&S" (d0)
        //输出__res=eax
        : "1" (s),"0" (c));
        //输入esi=s, eax=c
    return __res;
}
```

### get\_user

```c
/*------------ exec.c/count() ------------*/

static int count(char ** argv, int max)
{
    int i = 0;

    if (argv != NULL) {
        for (;;) {
            char * p;
            int error;

            error = get_user(p,argv);
            if (error) //出问题了
                return error;
            if (!p)    //到末尾了，最后一个是NULL
                break;
            argv++;
            if(++i > max)
                return -E2BIG;
        }
    }
    return i;
}

/*------------ uaccess.h/get_user() ------------*/

#define __get_user_x(size,ret,x,ptr) \
    __asm__ __volatile__("call __get_user_" #size \
        :"=a" (ret),"=d" (x) \
        //输出ret=eax, x=edx
        :"0" (ptr))
        //输入eax=ptr

// 成功情况x=*ptr, return 0
// 失败情况x=0, return -14
#define get_user(x,ptr)                            \
({    int __ret_gu,__val_gu;                        \
    switch(sizeof (*(ptr))) {                    \
    case 1:  __get_user_x(1,__ret_gu,__val_gu,ptr); break;        \
    case 2:  __get_user_x(2,__ret_gu,__val_gu,ptr); break;        \
    case 4:  __get_user_x(4,__ret_gu,__val_gu,ptr); break;        \
    default: __get_user_x(X,__ret_gu,__val_gu,ptr); break;        \
    }                                \
    (x) = (__typeof__(*(ptr)))__val_gu;                \
    __ret_gu;                            \
})

/*------------ getuser.S/__get_user_4 ------------*/
.align 4
.globl __get_user_4
__get_user_4:            //从用户空间指针取四个字节, 即return *eax
    addl $3,%eax         //eax+=3, 得到四字节内存区域末端
    movl %esp,%edx       //edx=esp
    jc bad_get_user      //如果eax+=3过程中有进位。防止eax==0xffffffff,
                         //eax+=3之后溢出，反而变成了有效地址
    andl $0xffffe000,%edx//将进程系统堆栈指针8KB对齐，即取task_struct
    cmpl addr_limit(%edx),%eax //eax >= task_struct.addr_list?
    jae bad_get_user     //超过地址上限
3:  movl -3(%eax),%edx   //edx=*(eax-3)
    xorl %eax,%eax       //eax=0
    ret

bad_get_user:            //用户地址非法
    xorl %edx,%edx       //edx=0
    movl $-14,%eax       //eax=-14
    ret

.section __ex_table,"a"
    .long 1b,bad_get_user
    .long 2b,bad_get_user
    .long 3b,bad_get_user
```