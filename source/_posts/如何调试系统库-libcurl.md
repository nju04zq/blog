title: 如何调试系统库(libcurl)
date: 2015-08-26 21:24:51
tags: [gdb]
---
最近用libcurl库的时候遇到一些问题，想用GDB深入跟踪库的源代码。奈何系统自带库一则不附源码，二则编译的时候加过优化选项，没法用GDB深入调试。捣鼓了半天，找到了调试系统库的办法。

<!-- more -->

首先，到网上下载libcurl库对应的源代码。然后解压，configure。configure的时候指定库的存放位置（必须是绝对路径），而且要告诉它用debug模式编译。比如运行如下命令，

    ./configure --prefix=“/xxxxx/curl/build" --enable-debug

然后是老一套，make，make install。编译好的库就在build/lib下面。

接着，需要在编译目标程序的时候指定使用这个库，可以这么干，

    gcc -g -o test test.c -L/xxxxx/curl/build/lib -lcurl

记得设置环境变量，不然系统不知道到哪里找这个库，
    
    export LD_LIBRARY_PATH=/xxxxx/curl/build/lib:$LD_LIBRARY_PATH
    
最后确认一下，编译好的test是不是用的我们指定的库，

    ldd test
    
应该会看到
    
    libcurl.so.4 => /xxxxx/curl/build/lib/libcurl.so.xxxx

到这里就完工了，可以幸福的用GDB调试libcurl库咯。
