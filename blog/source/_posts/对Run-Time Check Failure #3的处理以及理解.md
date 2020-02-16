---
title: Time Check Failure 处理以及理解
date: 2016-04-21 01:24:24
tags: 
-  c/c++
---

对 Run-Time Check Failure #3 - The variable 'a' is being used without being initialized 的处理以及理解

<!--more-->

对于这个问题，我们先来看一个简单的C程序:

```c++
#include <stdio.h>

void Fun1()
{
    int a = 48;
}

void Fun2()
{
    int a;
    printf("%d\n",a);
}

int main()
{
    Fun1();
    Fun2();

    return 0;
}
```

你们猜猜运行结果是什么？在VS下直接崩溃。。。
然后我又用VC和Linux下GCC测试，结果分别为-858993460和随机值。
于是。。。（懵逼）

通过分析，发现这与RTC（Run-Time Check，运行时检查）机制有关（以下都是以VS2012为标准）。

首先普及一下RTC（Run-Time Check）机制，包括**堆栈帧（RTCS）**、**未初始化变量（RTCu）**、**两者都有**、以及**默认值**四种。

在VS2012编译器中，项目-》属性-》配置属性-》C/C++ -》生成代码-》基本运行时检测 

1. 当开启RTCu（对未初始化变量运行时的检查）时，程序会崩溃。提示错误RTC Failure#3：使用了未初始化的变量。

2. 当开启RTCs（对堆栈帧运行时的检查）时，可以运行，结果为固定值，针对以上程序，结果为-858993460（%d打印），0xcccccccc（%#x打印）。实际上调用Fun2()开辟新栈帧时，栈帧被0xcc初始化，打印出来的结果如上。

3. 当两者都开启时，与情况一一样，程序崩溃。提示错误RTC Failure#3：使用了未初始化的变量

4. 当使用默认值时，程序可以运行，结果为相对应位置的值。例如本程序中结果为48，若程序Fun1()函数中只有一个变量，值为多少，结果就为多少。说白了，函数Fun2()中变量a的值与其本身无关，而与相对应的内存中存放的值有关。原因是默认值这种模式，当有新的栈帧开辟时，不会有0xcc这个初始化的过程。

以下为本人对如上问题的理解，如有偏颇，希望同仁批评指正，谢谢！

转载自:**http://blog.csdn.net/qq_29894329/article/details/51184920**