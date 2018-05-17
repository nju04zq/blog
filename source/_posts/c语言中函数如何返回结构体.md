title: c语言中函数如何返回结构体
date: 2015-08-26 21:13:48
tags: [c语言,汇编]
---

如果我们需要函数返回结构体，一般会使用返回指针的方式。今天看到有库函数直接返回结构体，不禁很好奇这如何能够实现。因为X86平台上一般用eax/rax寄存器存放返回值，一个结构体可以很大，寄存器如何放的下？

<!-- more -->

要了解个中细节还得从汇编角度分析。

下面进一段C语言示例程序和对应的反汇编结果

C程序源码

```c
typedef struct {
    int a;
    int b;
    char c[3];
} foo_t;
	
foo_t global_a;
	
static foo_t
foo (void)
{
    global_a.a = 1;
    global_a.b = 2;
    global_a.c[0] = 'a';
    global_a.c[1] = 'b';
    global_a.c[2] = 'c';
    return global_a;
}
	
int main (void)
{
    foo_t a;
	
    a = foo();
	
    return 0;
}
```

linux x86_64上反汇编后的（gcc -s 1.c 然后查看1.s文件）

```c
	.type   foo, @function
	foo:
	.LFB2:
	        pushq   %rbp
	.LCFI0:
	        movq    %rsp, %rbp
	.LCFI1:
	        movl    $1, global_a(%rip)
	        movl    $2, global_a+4(%rip)
	        movb    $97, global_a+8(%rip)
	        movb    $98, global_a+9(%rip)
	        movb    $99, global_a+10(%rip)
	        movq    global_a(%rip), %rax
	        movq    %rax, -16(%rbp)
	        movl    global_a+8(%rip), %eax
	        movl    %eax, -8(%rbp)
	        movq    -16(%rbp), %rax
	        movl    -8(%rbp), %edx
	        leave
	        ret
	.LFE2:
	        .size   foo, .-foo
	.globl main
	        .type   main, @function
	main:
	.LFB3:
	        pushq   %rbp
	.LCFI2:
	        movq    %rsp, %rbp
	.LCFI3:
	        subq    $32, %rsp
	.LCFI4:
	        call    foo
	        movq    %rax, -32(%rbp)
	        movl    %edx, -24(%rbp)
	        movq    -32(%rbp), %rax
	        movq    %rax, -16(%rbp)
	        movl    -24(%rbp), %eax
	        movl    %eax, -8(%rbp)
	        movl    $0, %eax
	        leave
	        ret
	.LFE3:
```

上一段程序里面返回了foo\_t这个结构，foo\_t占12字节（考虑padding），所以一个寄存器（8字节）根本无法容纳。那编译器怎么处理的呢？用两个寄存器呗！

这里简单解释一下那段汇编。8-12行对应C里面12-16行的赋值。第13行把global\_a结构开始的8字节内容存到rax内，然后紧接着14行再把rax的内容暂存到堆栈上。第15行将global\_a结构的最后4字节内容存入eax，第16行再把eax内容同样暂存到堆栈上。第17，18行将之前暂存的数据放入rax和edx内做为返回数据。那为啥不直接拷贝到rax和edx里面？我怀疑编译器一根筋吧。。。

那么调用者怎么处理这些返回值的呢？第33行调用完毕之后，34/35行将rax/edx内的返回值暂存到堆栈中。然后36-39行把这些暂存值填到变量a里面。后面40-42行设置返回值，平衡堆栈，搞定收工。

这里多说两句36-39行代码，a的地址应该是-16(%rbp)，要记住堆栈是逆序生长的，而变量的地址是内存区的起始地址。

上面演示的这段foo\_t才12字节。那如果结构体很大，编译器怎么处理的？一切用代码说话。

这段c代码定义了一个巨大无比的结构（相对寄存器大小）。

```c
typedef struct {
    int a;
    int b;
    char c[128];
} foo_t;

foo_t global_a;

static foo_t
foo (void)
{
    global_a.a = 1;
    global_a.b = 2;
    global_a.c[0] = 'a';
    global_a.c[1] = 'b';
    global_a.c[2] = 'c';
    return global_a;
}

int main (void)
{
    foo_t a;

    a = foo();

    return 0;
}
```

我们看看反汇编后的代码

```c
foo:
.LFB2:
	pushq	%rbp
.LCFI0:
	movq	%rsp, %rbp
.LCFI1:
	pushq	%rbx
.LCFI2:
	subq	$8, %rsp
.LCFI3:
	movq	%rdi, %rbx
	movl	$1, global_a(%rip)
	movl	$2, global_a+4(%rip)
	movb	$97, global_a+8(%rip)
	movb	$98, global_a+9(%rip)
	movb	$99, global_a+10(%rip)
	movq	%rbx, %rdi
	movl	$global_a, %esi
	movl	$136, %edx
	call	memcpy
	movq	%rbx, %rax
	addq	$8, %rsp
	popq	%rbx
	leave
	ret
.LFE2:
	.size	foo, .-foo
.globl main
	.type	main, @function
main:
.LFB3:
	pushq	%rbp
.LCFI4:
	movq	%rsp, %rbp
.LCFI5:
	subq	$144, %rsp
.LCFI6:
	leaq	-144(%rbp), %rdi
	call	foo
	movl	$0, %eax
	leave
	ret
```

从汇编代码可以看到，编译器很机智，直接把结构体a的地址作为一个隐含参数来调用foo。具体来说，在39行调用foo之前，36行分配了堆栈来存放变量a（由于call指令会将紧着着的指令地址入栈，比如这里的第40行，144字节里面有8字节存放这个地址），38行把a的地址放入寄存器edi内。然后在函数foo内部，12-16行赋值。在11行会将rdi的值暂存rbx，17行再取出来，我估计为了防止12-16行之间的代码会用到rdi。什么？看了8遍没看出来哪行指令会用到rdi？还是那句话，编译器一根筋，这么循规蹈矩弄一下没啥坏处。然后18行设置目的地址寄存器（global_a的地址），19行设置待拷贝内存长度，20行直接memcpy，然后21行将返回值设置为变量a的地址。后面22-25行堆栈平衡，收工。

当然也可以在gdb里面跟踪上述汇编代码，简单说来，gcc加g选项添加调试信息。然后gdb调试对应程序，disassemble命令反汇编出当前函数，然后ni/si运行下一条指令，根据pc值跟之前用disassemble命令反汇编的对应起来。ni不进入函数跟踪，si进入函数跟踪。info reg rax查看rax寄存器的值。

最后来个收尾。简单一行调用居然隐藏了这么多细节。所以个人并不喜欢这种直接返回结构体的写法，还是用指针更加一目了然。
