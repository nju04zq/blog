---
title: python中subprocess如何引用.bashrc?
date: 2017-03-27 19:00:32
tags: [python,linux,process group]
---

在python中运行外部程序一般会用到subprocess模块。subprocess模块的Popen类可以方便的创建一个子进程。但是Popen并不会预先读入~/.bashrc中的环境变量，即使指定用bash运行外部程序。如果需要引用.bashrc中的一些环境变量，比如PATH，或者alias定义，怎样才能实现呢？

<!-- more -->

我们先看一个例子。

```python
import subprocess

def run_cmd(cmdline):
    cmdline = ["bash", "-c", cmdline]
    print "*** Running {0} ***".format(cmdline)
    p = subprocess.Popen(cmdline,
                         stdin=None,
                         stdout=subprocess.PIPE,
                         stderr=subprocess.STDOUT)
    while True:
        line = p.stdout.readline()
        if line != "":
            print line,
            continue
        rc = p.poll()
        if rc is not None:
            break

run_cmd('grep ll ~/.bashrc')
run_cmd('ll ~/.bashrc')
```

上面这段代码通过Popen执行其他程序，并指定用bash运行。输出如下，

```
*** Running ['bash', '-c', 'll ~/.bashrc'] ***
bash: ll: command not found
*** Running ['bash', '-c', 'grep ll ~/.bashrc'] ***
alias ll="ls -l"
```

很明显，我们在.bashrc中有定义*alias ll*，Popen运行的时候却找不到ll。这是因为bash仅在**-i**模式下才会主动读入.bashrc。好的，我们略作修改，把cmdline定义成*cmdline = ["bash", "-i", "-c", cmdline]*。然后再次运行，输出如下，

```
*** Running ['bash', '-i', '-c', 'll ~/.bashrc'] ***
-rw-r--r--. 1 Qiang Qiang 373 Mar 21 19:05 /home/Qiang/.bashrc
*** Running ['bash', '-i', '-c', 'grep ll ~/.bashrc'] ***

[1]+  Stopped                 python run_cmd.py
```

这次终于能够正确地执行ll这个alias。但是下面一行怎么回事，run_cmd.py怎么被stop了放到后台？如果需要继续运行，必须通过fg命令。这显然不是我们希望看到的。

这背后究竟发生了什么事情？是什么导致了run\_cmd.py被stop？为了解答这个问题，我们得依赖strace。调用strace的参数如下，

```
strace -tt -ff -o result python run_cmd.py
```

其中参数 *-ff* 加上 *-o* 把跟踪过程中每个进程/线程的系统调用存到文件中。strace运行后我们通过下面的命令确定其中涉及的进程号。

```
$ ps a -o "pid ppid pgid sid command" | grep run_cmd
31357 30386 31357 30386 strace -tt -ff -o result python run_cmd.py
31362 31357 31357 30386 python run_cmd.py
31431 31381 31430 31381 grep run_cmd
$ ps a -o "pid ppid pgid sid command" | grep 30386
30386 30385 30386 30386 -bash
31357 30386 31357 30386 strace -tt -ff -o result python run_cmd.py
31362 31357 31357 30386 python run_cmd.py
31376 31362 31357 30386 bash -i -c grep ll ~/.bashrc
31442 31381 31441 31381 grep 30386
```
这里面pid是进程号，ppid是父进程号，pgid是进程组(process group)id，sid是session的id。当你通过ssh登录到一台机器，从bash shell到所有后续运行的程序都会归属一个session，session的id就是bash shell的pid，bash shell这时也是session的leader process。进程组则是下一层组织架构，一个session内部有若干进程组，而一个进程组内部有若干进程。进程组内也有一个leader process，进程组的id就是leader process的pid。因此30386是我们做实验session的id。我们通过搜索这个id可以把实验中处于stop状态的进程都找出来。

首先查看文件result.31362，里面有run\_cmd.py对应进程的系统调用细节。其中clone系统调用创建子进程，并且出现过两次，对应的是run\_cmd.py两次调用bash跑命令。第二次clone之后就出现了异常，进程被信号SIGTTIN stop了，如下所示。

```
20:27:46.113939 read(3, 0x2307063, 1)   = ? ERESTARTSYS (To be restarted if SA_RESTART is set)
20:27:46.117938 --- SIGTTIN {si_signo=SIGTTIN, si_code=SI_USER, si_pid=31376, si_uid=501} ---
20:27:46.117952 --- stopped by SIGTTIN ---
```
我们不急着分析其中的原因，接着检查文件result.31376。这个是第二个bash命令对应进程的系统调用输出。其中也有类似的段落，进程被信号SIGTTIN stop了。而且在被stop之前，使用kill向整个进程组广播信号SIGTTIN。这就是为什么父进程31362为什么会收到SIGTTIN。

```
20:27:46.117873 ioctl(255, TIOCGPGRP, [31363]) = 0
20:27:46.117889 rt_sigaction(SIGTTIN, {SIG_DFL, [], SA_RESTORER, 0x33b9432660}, {SIG_IGN, [], SA_RESTORER, 0x33b9432660}, 8) = 0
20:27:46.117907 kill(0, SIGTTIN)        = 0
20:27:46.117931 --- SIGTTIN {si_signo=SIGTTIN, si_code=SI_USER, si_pid=31376, si_uid=501} ---
20:27:46.117947 --- stopped by SIGTTIN ---
```

那么子进程为什么会收到信号SIGTTIN呢？根据 *The Linux Programming Interface* 一书中的解释，

```
SIGTTIN
When running under a job-control shell, the terminal driver sends
this signal to a background process group when it attempts to 
read() from the terminal. This signal stops a process by default.
```

子进程31376作为一个后台进程，尝试着去操作终端，因此被通知到信号SIGTTIN。

```
20:27:46.117795 open("/dev/tty", O_RDWR|O_NONBLOCK) = 3
20:27:46.117815 getrlimit(RLIMIT_NOFILE, {rlim_cur=1024, rlim_max=4*1024}) = 0
20:27:46.117830 fcntl(255, F_GETFD)     = -1 EBADF (Bad file descriptor)
20:27:46.117844 dup2(3, 255)            = 255
20:27:46.117859 close(3)                = 0
20:27:46.117873 ioctl(255, TIOCGPGRP, [31363]) = 0
20:27:46.117889 rt_sigaction(SIGTTIN, {SIG_DFL, [], SA_RESTORER, 0x33b9432660}, {SIG_IGN, [], SA_RESTORER, 0x33b9432660}, 8) = 0
20:27:46.117907 kill(0, SIGTTIN)        = 0
20:27:46.117931 --- SIGTTIN {si_signo=SIGTTIN, si_code=SI_USER, si_pid=31376, si_uid=501} ---
20:27:46.117947 --- stopped by SIGTTIN ---
```

那如何修改呢？我们可以调用**os.tcsetpgrp()**把终端控制权拿回来，但是要注意，后台进程直接去拿终端会收到SIGTTOU。这个信号同样会导致进程stop。因此我们还需要改变SIGTTOU的handler，让进程忽略这个信号。

修改完的代码如下：

```python
import os
import signal
import subprocess

signal.signal(signal.SIGTTOU, signal.SIG_IGN)

def run_cmd(cmdline):
    cmdline = ["bash", "-i", "-c", cmdline]
    print "*** Running {0} ***".format(cmdline)
    p = subprocess.Popen(cmdline,
                         stdin=None,
                         stdout=subprocess.PIPE,
                         stderr=subprocess.STDOUT)
    while True:
        line = p.stdout.readline()
        if line != "":
            print line,
            continue
        rc = p.poll()
        if rc is not None:
            break
    fd = os.open("/dev/tty", os.O_RDWR)
    os.tcsetpgrp(fd, os.getpgrp())

run_cmd('ll ~/.bashrc')
run_cmd('grep ll ~/.bashrc')
```

改完之后就一切正常了。上文的解决方案部分参考stackoverflow[这篇文章](http://stackoverflow.com/questions/25099895/from-python-start-a-shell-that-can-interpret-functions-and-aliases)。进程组和session的知识可以参考*The Linux Programming Interface* 第34章。
