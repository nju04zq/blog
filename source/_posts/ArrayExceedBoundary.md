---
title: 一个数组越界的问题
date: 2017-07-21 21:01:17
tags: [c语言,汇编]
---
写数组越界之后会发生什么事情呢？
先看一段代码，

<!-- more -->

```c
int main() {
    int a[10], i;

    for (i = 1; i <= 10; i++) {
        a[i] = 0;
    }
    return 0;
}
```
这段代码看起来似乎是个笑话，一个完全没有编程常识的人才会写出这么搞笑的代码。但是深思一下，这段代码执行之后会有什么结果？Segment Fault？死循环？这都取决于越界的a[10] = 0究竟写到了哪里。如果&a[10]对应变量i的地址，这样就会出现死循环。如果&a[10]对应的是存放rbp寄存器的位置，main函数return的时候就会出问题。

## Mac OSX平台

首先我们在Mac OSX系统上看这个问题。这段代码在OSX运行之后会异常退出。我们打开GDB，可以看到main函数在return时会检查堆栈，就是在这里发生的异常。

```
(gdb) run
Starting program: /Users/Qiang/workspace/go/test

Program received signal SIGABRT, Aborted.
0x00007fff93d31f06 in __pthread_kill () from /usr/lib/system/libsystem_kernel.dylib
(gdb) bt
#0  0x00007fff93d31f06 in __pthread_kill ()
   from /usr/lib/system/libsystem_kernel.dylib
#1  0x00007fff8628f4ec in pthread_kill ()
   from /usr/lib/system/libsystem_pthread.dylib
#2  0x00007fff985be77f in __abort () from /usr/lib/system/libsystem_c.dylib
#3  0x00007fff985bf05e in __stack_chk_fail () from /usr/lib/system/libsystem_c.dylib
#4  0x0000000100000f89 in main () at test.c:7
(gdb)
```
反汇编main函数，可以得到下面这块汇编代码。

```nasm
(gdb) disass main
Dump of assembler code for function main:
   0x0000000100000f20 <+0>:	push   %rbp
   0x0000000100000f21 <+1>:	mov    %rsp,%rbp
   0x0000000100000f24 <+4>:	sub    $0x40,%rsp
   0x0000000100000f28 <+8>:	mov    0xe1(%rip),%rax        # 0x100001010
   0x0000000100000f2f <+15>:	mov    (%rax),%rax
   0x0000000100000f32 <+18>:	mov    %rax,-0x8(%rbp)
   0x0000000100000f36 <+22>:	movl   $0x0,-0x34(%rbp)
   0x0000000100000f3d <+29>:	movl   $0x1,-0x38(%rbp)
   0x0000000100000f44 <+36>:	cmpl   $0xa,-0x38(%rbp)
   0x0000000100000f48 <+40>:	jg     0x100000f68 <main+72>
   0x0000000100000f4e <+46>:	movslq -0x38(%rbp),%rax
   0x0000000100000f52 <+50>:	movl   $0x0,-0x30(%rbp,%rax,4)
   0x0000000100000f5a <+58>:	mov    -0x38(%rbp),%eax
   0x0000000100000f5d <+61>:	add    $0x1,%eax
   0x0000000100000f60 <+64>:	mov    %eax,-0x38(%rbp)
   0x0000000100000f63 <+67>:	jmpq   0x100000f44 <main+36>
   0x0000000100000f68 <+72>:	mov    0xa1(%rip),%rax        # 0x100001010
   0x0000000100000f6f <+79>:	mov    (%rax),%rax
   0x0000000100000f72 <+82>:	cmp    -0x8(%rbp),%rax
   0x0000000100000f76 <+86>:	jne    0x100000f84 <main+100>
   0x0000000100000f7c <+92>:	xor    %eax,%eax
   0x0000000100000f7e <+94>:	add    $0x40,%rsp
   0x0000000100000f82 <+98>:	pop    %rbp
   0x0000000100000f83 <+99>:	retq
   0x0000000100000f84 <+100>:	callq  0x100000f8a
End of assembler dump.
```
<+0><+1>初始化栈帧，<+4>扩展堆栈。<+8><+15><+18>很有意思，它从代码段读了8字节，然后存到了堆栈上面。<+22>到<+67>对应着C代码中的循环体。<+72><+79><+82>尝试校验之前存放在堆栈上的8字节信息。由于代码段是只读的，因此堆栈上的这8字节正常情况下不会变化，如果被改变了，必定栈被写坏了。运行时的异常\_\_stack\_chk\_fail似乎跟这块有关系。

我们用gdb检查a[10]的地址就能证实了。

```
(gdb) info r rbp
rbp            0x7fff5fbffac0	0x7fff5fbffac0
(gdb) p &a[10]
$1 = (int *) 0x7fff5fbffab8
(gdb) p &i
$2 = (int *) 0x7fff5fbffa88
(gdb)
```
果不其然，&a[10]对应的正是堆栈保护区域-0x8(%rbp)。OSX的编译器Clang提供了堆栈保护功能。正因为如此，OSX上这次数组越界会被发现，程序也会异常退出。

汇编代码中<+4>处扩展堆栈直接申请了64字节的空间（0x40）。这个是x86\_64的ABI规定的，rsp寄存器必须保证16字节对齐。（[x86\_64 ABI](https://software.intel.com/sites/default/files/article/402129/mpx-linux64-abi.pdf) P35）

## Linux平台
我们用GCC外加编译选项"-g -O0"来编译这段代码，然后运行。居然一切正常！

这是怎么回事呢？先用GDB反汇编一把，

```nasm
(gdb) disass main
Dump of assembler code for function main:
   0x00000000004004d0 <+0>:	push   %rbp
   0x00000000004004d1 <+1>:	mov    %rsp,%rbp
   0x00000000004004d4 <+4>:	movl   $0x1,-0x4(%rbp)
   0x00000000004004db <+11>:	jmp    0x4004ee <main+30>
   0x00000000004004dd <+13>:	mov    -0x4(%rbp),%eax
   0x00000000004004e0 <+16>:	cltq
   0x00000000004004e2 <+18>:	movl   $0x0,-0x30(%rbp,%rax,4)
   0x00000000004004ea <+26>:	addl   $0x1,-0x4(%rbp)
   0x00000000004004ee <+30>:	cmpl   $0xa,-0x4(%rbp)
   0x00000000004004f2 <+34>:	jle    0x4004dd <main+13>
   0x00000000004004f4 <+36>:	mov    $0x0,%eax
   0x00000000004004f9 <+41>:	pop    %rbp
   0x00000000004004fa <+42>:	retq
End of assembler dump.
(gdb)
```

GCC没有像Clang那样搞堆栈保护。<+0><+1>初始化栈帧，<+4>相当于i=1，<+11>到<+34>对应循环，<+36>之后就是return 0了。顺便提一下<+16>的cltq指令，<+13>中相当于eax=i，但<+18>需要用rax计算偏移，因此需要将eax扩展到rax，相当于int64(i)。

我们看一下&a[10]的地址。

```
(gdb) p &i
$1 = (int *) 0x7fffffffe3dc
(gdb) p &a[10]
$2 = (int *) 0x7fffffffe3d8
(gdb) info r rbp
rbp            0x7fffffffe3e0	0x7fffffffe3e0
(gdb)
```
GCC把数组放到堆栈顶端，变量i则在栈帧的底部区域，紧邻rbp在堆栈上的存放值。为了保证堆栈8字节对齐，一共扩展了48字节堆栈（<+18>中0x30），数组a用了40字节，变量i占用4字节，多出来的4字节空档正好在&a[10]上。因此数组越界对程序的运行没有产生任何影响！

## 发散思考
如果我们编译的时候加优化会发生什么？如果数组大小为11，又会发生什么？下面我们在Linux上回答这些问题。

### 编译加优化选项
使用"-g -O1"编译，然后到gdb里面反汇编，

```nasm
(gdb) disass main
Dump of assembler code for function main:
   0x00000000004004d0 <+0>:	mov    $0x0,%eax
   0x00000000004004d5 <+5>:	retq
End of assembler dump.
(gdb)
```
看到没有，优化之后只剩下一个简单的return 0。那段孤立的循环代码直接被编译器抛弃掉了。

### 数组大小设为11
将数组定义以及for循环中的10改为11，编译选项“-g -O0”，我们看看会有什么结果。

```
(gdb) p &a[11]
$2 = (int *) 0x7fffffffe3dc
(gdb) p &i
$3 = (int *) 0x7fffffffe3dc
```
这时改&a[11]直接修改了变量，自然会死循环。

那么如果加优化选项编译呢，当然要做点事情保证GCC不会抛弃那段代码。改好的代码如下，

```c
int main() {
    int a[11], i;

    a[0] = 1; // see where does GCC put the array
    for (i = 1; i <= 11; i++) {
        a[i] = 0;
    }
    return a[0];
}
```
使用"-g -O1"编译，然后反汇编，

```nasm
(gdb) disass main
Dump of assembler code for function main:
   0x00000000004004d0 <+0>:	movl   $0x1,-0x30(%rsp)
   0x00000000004004d8 <+8>:	lea    -0x2c(%rsp),%rax
   0x00000000004004dd <+13>:	mov    %rsp,%rdx
   0x00000000004004e0 <+16>:	movl   $0x0,(%rax)
   0x00000000004004e6 <+22>:	add    $0x4,%rax
   0x00000000004004ea <+26>:	cmp    %rdx,%rax
   0x00000000004004ed <+29>:	jne    0x4004e0 <main+16>
   0x00000000004004ef <+31>:	mov    -0x30(%rsp),%eax
   0x00000000004004f3 <+35>:	retq
End of assembler dump.
(gdb)
```
变量i在优化之后直接消失了。<+0>对应a[0]=1。<+8>中，-0x2c(%rsp)对应&a[1]，相当于rax=&a[1]。<+13>相当于rdx=&a[12]。<+16>到<+29>对应for循环。由于扩展堆栈的时候需要保证16字节对齐，&a[11]落在空档中，所以优化之后的程序跑起来没有任何问题！

### 数组大小设为12 + 优化
将11都替换为12，使用"-g -O1"编译，然后反汇编，

```nasm
(gdb) disass main
Dump of assembler code for function main:
   0x00000000004004d0 <+0>:	movl   $0x1,-0x30(%rsp)
   0x00000000004004d8 <+8>:	lea    -0x2c(%rsp),%rax
   0x00000000004004dd <+13>:	lea    0x4(%rsp),%rdx
   0x00000000004004e2 <+18>:	movl   $0x0,(%rax)
   0x00000000004004e8 <+24>:	add    $0x4,%rax
   0x00000000004004ec <+28>:	cmp    %rdx,%rax
   0x00000000004004ef <+31>:	jne    0x4004e2 <main+18>
   0x00000000004004f1 <+33>:	mov    -0x30(%rsp),%eax
   0x00000000004004f5 <+37>:	retq
End of assembler dump.
(gdb) start
(gdb) info r rsp
rsp            0x7fffffffe3e8	0x7fffffffe3e8
(gdb) p &a[12]
$1 = (int *) 0x7fffffffe3e8
(gdb) x/1gx 0x7fffffffe3e8
0x7fffffffe3e8:	0x00000033b941ed1d
(gdb)
```
这时&a[12]就跑出当前函数的栈帧了，从<+13>中的0x4(%rsp)就能看出问题。这个地方存放的是函数的返回地址。

在return a[0]处下断点，继续跑，

```
Breakpoint 2, main () at test.c:9
9	}
(gdb) x/1gx 0x7fffffffe3e8
0x7fffffffe3e8:	0x0000003300000000
(gdb) ni
0x00000000004004f5	9	}
(gdb)
0x0000003300000000 in ?? ()
(gdb) ni

Program received signal SIGSEGV, Segmentation fault.
0x0000003300000000 in ?? ()
(gdb)
```
此时返回地址的低32位被清零，最终被Segmentation fault终结。

### 最后，优化后的栈帧
上面优化之后的汇编代码还有另外一个变化，使用rbp建立栈帧的操作通通消失了，于是一个可以还原调用栈的链表也就不存在了。如果没有rbp，怎么能得到函数的调用栈呢？

其实很简单，程序员不用工具几乎不可能在这种情况下拿到完整的调用栈，但是诸如GDB的调试工具却能够做到。GDB可以分析文件获取编译信息，因为编译结束后，每个扩展栈的操作已经定了，也就是说每个函数的工作栈大小是没有变数的。GDB根据这些信息自然可以还原调用栈。[这篇文章](http://yosefk.com/blog/getting-the-call-stack-without-a-frame-pointer.html)有详细的解释。