---
title: Intel CPU的微架构
date: 2017-08-12 22:17:07
tags: CPU
---

一张表格让你了解Intel的CPU微架构历史。

<!-- more -->

| Codename | Year | Transistors | Example Model |
|:---------|:----:|:-----------:|:-------------|
| [Icelake](https://en.wikipedia.org/wiki/Icelake) | 2018??? | 10nm | ??? |
| [Cannonlake](https://en.wikipedia.org/wiki/Cannonlake) | 2017??? | 10nm | ??? |
| [Coffee Lake](https://en.wikipedia.org/wiki/Coffee_Lake) | 2017 | 14nm | i3-8100<br>i5-8400<br>i7-8700 |
| [Kaby Lake](https://en.wikipedia.org/wiki/Kaby_Lake) | 2016 | 14nm | i3-7101<br>i5-7400<br>i7-7700<br>Xeon E3-1220 v6 | 
| [Skylake](https://en.wikipedia.org/wiki/Skylake_%28microarchitecture%29) | 2015 | 14nm | i3-6100<br>i5-6400<br>i7-6700<br>Xeon E3-1220 v5 |
| [Broadwell](https://en.wikipedia.org/wiki/Broadwell_%28microarchitecture%29) | 2014 | 14nm | i3-5005<br>i5-5200<br>i7-5500<br>Xeon E5-2603 v4 |
| [Haswell](https://en.wikipedia.org/wiki/Haswell_%28microarchitecture%29) | 2013 | 22nm | i3-4130<br>i5-4670<br>i7-4770<br>Xeon E7-4809 v3 |
| [Ivy Bridge](https://en.wikipedia.org/wiki/Ivy_Bridge_%28microarchitecture%29) | 2012 | 22nm | i3-3210<br>i5-3330<br>i7-3770<br>Xeon E5-2403 v2 |
| [Sandy Bridge](https://en.wikipedia.org/wiki/Sandy_Bridge) | 2011 | 32nm | i3-2100<br>i5-2300<br>i7-2600<br>Xeon E5-2403 |
| [Westmere](https://en.wikipedia.org/wiki/Westmere_%28microarchitecture%29) | 2010 | 32nm | i3-530<br>i5-650<br>i7-970<br>Xeon E5620 |
| [Nehalem](https://en.wikipedia.org/wiki/Nehalem_%28microarchitecture%29) | 2008 | 45nm | i5-750<br>i7-860<br>Xeon E5506 |
| [Core](https://en.wikipedia.org/wiki/Intel_Core_%28microarchitecture%29) | 2006 | 65nm-45nm | Core 2 Duo<br>Core 2 Quad<br>Xeon E53xx |
| [NetBurst](https://en.wikipedia.org/wiki/NetBurst_%28microarchitecture%29) | 2000 | 180nm-65nm | Pentium 4<br>Xeon |
| [Pentium M](https://en.wikipedia.org/wiki/Pentium_M) | 2003 | 90nm | Pentium M<br>Core Duo<br>Core Solo |
| [P6](https://en.wikipedia.org/wiki/P6_%28microarchitecture%29) | 1995 | 350nm-130nm | Pentium Pro<br>Pentium II<br>Pentium III |

Notes:

- Pentium M其实是P6的移动平台版本。

- 首款支持x86\_64指令集的CPU由AMD在2000年左右推出。而Intel在2004年才有类似产品（Xeon，CPU代号Nocona， 使用NetBurst微架构），在移动平台则于2006年支持（代号Merom，Core 2）。

- 流水线长度并不是越长越好。NetBurst有20级pipeline，后期甚至提高到31级。主频虽然提上去了（2004年的[Prescott](http://ark.intel.com/products/codename/1791/Prescott)主频可以达到4.0G，相比较而言，2015年Skylake架构下的高端型号[i7-6700K](http://ark.intel.com/products/88195/)也不过4.0G主频），但是指令延迟也增加了，整体而言CPU的性能却下降了。因此后面的Core微架构直接把pipeline降到了12级。

- L1 cache的大小在Nehalem（2008年推出）之后的CPU架构上是64KB每核心，L2 cache是256KB每核心，L3 cache在2MB以上，可能是所有核心共享。如果不加cache，使用CPU+DRAM的结构，CPU访问内存的延迟在80个时钟周期左右。如果加上cache，L1的延迟4个周期左右，L2的延迟15个周期左右，L3的延迟50个周期左右，此时由于添加cache的缘故，CPU访问内存的延迟提高到120个时钟周期左右。如果CPU需要的内容，90%在L1 Cache里有，6%在L2 Cache里有，3%在L3 Cache里有，1%要去找DRAM拿。那么整个存储体系的等效延迟就是：7.2个时钟周期。参考[知乎](https://www.zhihu.com/question/22431522)。

- HyperThread（HT，超线程）TODO

- SuperScalar（超标量）TODO

- 主频，倍频 TODO