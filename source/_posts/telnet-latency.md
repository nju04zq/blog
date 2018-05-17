title: 为何telnet到127.0.0.1会有5秒延迟？
date: 2017-03-03 20:57:14
tags: DNS
---

最近在写一个[简单的HTTP客户端/服务器端](https://github.com/nju04zq/HTTP_POST)。在验证服务器端的时候，用到了telnet工具。但是在每次telnet连接的时候，都会有数秒的延迟。按说telnet连接到本地，不会有网络延迟存在。那这几秒的延迟出在哪个环节上呢？

<!-- more -->

我们可以借助strace来分析telnet背后的细节。这个工具能够追踪进程的一切系统调用。

strace用法如下，-tt选项能够显示毫秒精度的时间戳。

```
strace -tt telnet 127.0.0.1 9000 >& /tmp/strace.log
```

检查log（相关片段如下所示），原来poll系统调用等待了5秒。往前看，有创建一个socket，然后连接到192.168.11.1的53端口。53端口是用来做DNS解析的，这里解析的对象是机器的hostname，*vm-dev*。但是这个IP根本不可达，所以poll只能干等，直到超时。

```
21:21:16.032995 socket(PF_INET, SOCK_DGRAM|SOCK_NONBLOCK, IPPROTO_IP) = 3
21:21:16.033027 connect(3, {sa_family=AF_INET, sin_port=htons(53), sin_addr=inet_addr("192.168.11.1")}, 16) = 0
21:21:16.033066 gettimeofday({1488547276, 33072}, NULL) = 0
21:21:16.033085 poll([{fd=3, events=POLLOUT}], 1, 0) = 1 ([{fd=3, revents=POLLOUT}])
21:21:16.033107 sendto(3, "\3655\1\0\0\1\0\0\0\0\0\0\6vm-dev\0\0\1\0\1", 24, MSG_NOSIGNAL, NULL, 0) = 24
21:21:16.033201 poll([{fd=3, events=POLLIN|POLLOUT}], 1, 5000) = 1 ([{fd=3, revents=POLLOUT}])
21:21:16.033226 sendto(3, "W-\1\0\0\1\0\0\0\0\0\0\6vm-dev\0\0\34\0\1", 24, MSG_NOSIGNAL, NULL, 0) = 24
21:21:16.033275 gettimeofday({1488547276, 33284}, NULL) = 0
21:21:16.033295 poll([{fd=3, events=POLLIN}], 1, 4999) = 0 (Timeout)
21:21:21.037931 socket(PF_INET, SOCK_DGRAM|SOCK_NONBLOCK, IPPROTO_IP) = 4
21:21:21.037995 connect(4, {sa_family=AF_INET, sin_port=htons(53), sin_addr=inet_addr("8.8.4.4")}, 16) = 0
I
```

这里有两个问题，我用telnet连接到127.0.0.1，为什么要解析客户机的hostname？那个192.168.11.1的DNS地址哪里来的？

Linux的DNS服务器地址存放在 */etc/resolv.conf* 中，strace抓到的系统调用log也能看到读取这个文件的记录。在这个文件里面确实找到了上面见到的IP地址，192.168.11.1。这个Linux系统跑在虚拟机里面，宿主系统是OS X。前一段时间我曾经给OS X系统配置过DNS地址，就是上面的IP地址。看样子虚拟机也把这个配置拿进来用了。不过最近局域网的网段改了，那个DNS地址却没记得删掉。现在那个地址自然是不可达的。

那还剩下一个关键问题，telnet为什么要解析系统的hostname？没办法，只能从telnet的源代码入手了。把gdb叫过来，安装需要的debug模块，在5秒延迟的时候敲CTRL+C暂停telnet进程，检查进程的调用栈。调用栈长这样，

```
#8  0x00007ffff76dff30 in getaddrinfo (name=0x7fffffffe0f0 "vm-dev",
    service=<value optimized out>, hints=0x7fffffffe0b0, pai=0x7fffffffe0e0)
    at ../sysdeps/posix/getaddrinfo.c:2385
#9  0x00007ffff7ff0def in env_init () at commands.c:1676
#10 0x00007ffff7ff7439 in init_telnet () at telnet.c:145
#11 0x00007ffff7ff1bc3 in tninit () at main.c:68
#12 0x00007ffff7ff1be8 in main (argc=3, argv=0x7fffffffe2e8) at main.c:123
```

调用栈中的*getaddrinfo*函数就是用来做DNS解析的。这跟strace观察到的结果一致。用*list*看一下*commands.c:1676*的源代码。原来telnet为了给某个环境变量*DISPLAY*赋值，尝试着获取一个更规范的系统hostname（按照源码，hostname里面要有*.*字符），也就是DNS解析结果的cname字段。

按照[telnet的rfc](https://tools.ietf.org/html/rfc1572)描述，DISPLAY环境变量的用处和格式如下。

```
DISPLAY  This variable is used to transmit the X display location
         of the client.  The format for the value of the DISPLAY
         variable is:

         <host>:<dispnum>[.<screennum>]
```

至此，5秒的延迟算是水落石出了。解决这个问题很简单，删掉那个不可达的DNS地址就行了。

这个问题也很容易重现。步骤如下，

1. 启动httpd服务，sudo service httpd start
2. 修改/etc/resolv.conf文件，最开始加上nameserver 192.168.11.1
3. telnet 127.0.0.1 80，就可以重现了。
4. 记得把/etc/resolv.conf改回去。:-p