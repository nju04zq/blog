title: linux库的链接问题
date: 2016-08-24 19:58:33
tags: [link]
---
最近调试OpenSSL的证书验证功能，遇到了一个诡异的linux库的链接问题。

<!-- more -->

大背景如下，产品库 *libtp.so* 需要对下载的CA根证书做校验。在调用OpenSSL的某个校验函数时居然失败了。通过GDB跟踪OpenSSL源码，在调用函数 *EVP\_get\_digestbyname* 获取 *SHA256* 哈希算法的时候出现了问题。该函数理应返回 *SHA256* 算法的描述结构，但是调用后却返回了空指针。拿不到算法的描述信息，校验自然会失败。

首先怀疑的是，OpenSSL没有正确初始化。在使用OpenSSL库之前需要调用 *OPENSSL\_add\_all\_algorithms\_noconf* 做初始化。 检查代码，然后GDB跟踪，初始化一切正常。

于是，对函数 *EVP\_get\_digestbyname* 下断点。该函数仅调用 *OBJ\_NAME\_get* 获取数据。此外，算法描述信息被存放在一个哈希表中，变量 *names_lh* 指向这个哈希表。 函数 *OBJ\_NAME\_get* 就是从这个哈希表里面拿数据。根据变量 *names_lh* 的值也能证明OpenSSL库已经初始化完毕。

```c
Breakpoint 9, EVP_get_digestbyname (name=0x2aaaab80f2c7 "SHA256") at names.c:118
118             cp=(const EVP_MD *)OBJ_NAME_get(name,OBJ_NAME_TYPE_MD_METH);
(gdb) p names_lh
$30 = (struct lhash_st_OBJ_NAME *) 0x63ed10
(gdb) p &names_lh
$31 = (struct lhash_st_OBJ_NAME **) 0x2aaaaaf643e0
```

接着step进入函数 *OBJ\_NAME\_get*，再次打印变量 *names_lh*。期间没有其他赋值操作发生（也可以肯定其他线程不会调用OpenSSL库），变量 *names_lh* 的值居然变成了空，而且变量的地址也发生了变化。 要知道，这可是一个全局静态变量，一旦进程运行，它的地址不可能改变！而且变量 *names_lh* 值为空说明OpenSSL库没有被初始化。。。

```c
(gdb) s
OBJ_NAME_get (name=0x2aaaab80f2c7 "SHA256", type=1) at o_names.c:158
158             int num=0,alias;
(gdb) p &names_lh
$32 = (struct lhash_st_OBJ_NAME **) 0x2aaaaba88680
(gdb) p names_lh
$33 = (struct lhash_st_OBJ_NAME *) 0x0
```

只有一个解释，我们的进程内部有两份相同的OpenSSL源代码。初始化和获取数据发生在不同的源代码中。我们可以对比初始化和获取数据时的调用栈来证实这一点。

这个是初始化过程的调用栈。

```c
#0  EVP_add_digest (md=0x2aaaaaf44fa0) at names.c:85
#1  0x00002aaaaabf088e in OpenSSL_add_all_digests () at c_alld.c:71
#2  0x00002aaaaabf0103 in OPENSSL_add_all_algorithms_noconf () at c_all.c:84
```

这个是校验时候的调用栈。（略去了调用参数）

```c
#0  OBJ_NAME_get ("SHA256") at o_names.c:158
#1  0x00002aaaab78e2d5 in EVP_get_digestbyname ("SHA256") at names.c:118
#2  0x00002aaaab7fc118 in CMS_SignerInfo_verify () at cms_sd.c:783
#3  0x00002aaaab7f9e4e in CMS_verify () at cms_smime.c:382
```

然后利用 *ps aux* + *grep* 找到进程的pid，比如8499。打开/proc/8499/smaps，可以看到进程的虚拟地址空间详细信息。其中，我们只关心上面两个调用栈在其中的位置。两串地址分别分布在如下两个虚拟地址段中。

```c
2aaaaaab0000-2aaaaad32000 r-xp 00000000 00:1e 91778381
/xxx/libgch.so
Size:              2568 kB
Rss:               1344 kB
Shared_Clean:         0 kB
Shared_Dirty:         0 kB
Private_Clean:     1336 kB
Private_Dirty:        8 kB
Swap:                 0 kB
Pss:               1344 kB

2aaaab678000-2aaaab863000 r-xp 00000000 00:1e 96464261
/xxx/libcrypto.so
Size:              1964 kB
Rss:                444 kB
Shared_Clean:         0 kB
Shared_Dirty:         0 kB
Private_Clean:      432 kB
Private_Dirty:       12 kB
Swap:                 0 kB
Pss:                444 kB
```

第一个 *libgch.so* 是产品的核心库文件，第二个 *libcrypto.so* 是OpenSSL的库文件。问题来了，产品的库文件里面怎么会有OpenSSL的源代码？

检查编译配置，OpenSSL通过静态库方式链接到了 *libgch.so* 。因此生成的 *libgch.so* 包含了OpenSSL的源代码。静态库链接时，如果静态库里面某个 *.o* 文件完全没有用到，那么这个 *.o* 中的函数是不会被放入目标文件中。

本案中， *libgch.so* 有调用 *OPENSSL\_add\_all\_algorithms\_noconf* 初始化OpenSSL（发送HTTPS需要用到）。但是校验函数 *CMS\_verify* 却没有用到。因此静态链接时前者被链入 *libgch.so* ，后者则不在其中。其后将 *libgch.so* 和 *libtp.so* 一起链接到主程序的时候，对于 *libtp.so* 中引用的 *OPENSSL\_add\_all\_algorithms\_noconf* 优先在 *libgch.so* 中找到（链接时*libgch.so*排在OpenSSL库的前面）。而校验函数 *CMS\_verify* 则只能链接到OpenSSL的库 *libcrypto.so* 中。因此就出现了这么个奇怪的问题。

解决方案 & 总结：很简单，都用动态库。自己挖的坑自己填。