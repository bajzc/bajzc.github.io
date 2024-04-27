---
title: xv6学习笔记
subtitle:
date: 2024-04-27T08:48:00+07:00
slug: 2296229
draft: false
description:
keywords:
license:
comment: false
weight: 0
tags:
  - Linux
  - Debug
categories:
  - Review
hiddenFromHomePage: false
hiddenFromSearch: false
hiddenFromRss: false
hiddenFromRelated: false
summary:
toc: true
math: false
lightgallery: false
password:
message:
repost:
  enable: true
  url:

---

**注：标题所指的xv6是MIT的计算机操作系统课程[S.081](https://pdos.csail.mit.edu/6.828/2020/schedule.html)。**

### LEC 1: util

#### 环境部署

上来就在环境部署上遇到了点小问题，虽然别人不太可能复现，但仍记录在此，仅作参考。

我用的是Gentoo Linux，通过[crossdev](https://wiki.gentoo.org/wiki/Crossdev)编译的``riscv64-unkown-linux-gnu``是不可用的（链接错误），个人猜测是因为它链接了Linux的一些库，而xv6是裸二进制的elf环境。（对于toolchain并不熟悉，欢迎指正）。

Gentoo官方编译出的[qemu](https://packages.gentoo.org/packages/app-emulation/qemu)对于xv6也是不可用的状态,没有任何输出（QEMU_SOFTMMU_TARGETS="riscv64"），可能是版本太高了导致不兼容...

二者的解决方案是一样的，就是按照课程的教程下载编译安装对应的软件。安装位置可以选在`/opt`,最后在`.[zsh,bash]rc`里加上:`export PATH="/opt/qemu/bin:/opt/riscv-gcc/bin:%PATH"`

#### xargs.c
一开使有一个小问题没主意到，xv6要求的xargs是将每一行的输入都与argv单独合并。

比如假设xv6的echo支持转义符，`echo "hello\nworld" | xargs echo print:`，应该打印出两行两个`print:`，而非两行一个`print:`

所以最后执行`exec`的时候，应该维护一个`*xargv[]`,将`argv[1:]`与`stdin`的一行输入进行合并。

同时，应该注意到`read()`读取到`\n`的时候应该在字符串末尾写入`\0`而非`\n`。

#### primes.c
这个程序难就难在debug环节上，很难仅靠程序本身打印log的方式纠错。我的解决方案是替换掉头文件(*unistd.h*)然后在本地环境下编译调试。

需要注意到如果在父进程中关闭了管道(pipe)，管道里的缓存会被清空。所以父进程可以在输出完后再写入一个**0(EOF)**，并交由子进程关闭管道。

在编写的时候**不宜过早优化**，可以在确定算法正确后再优化管道数的问题。可以用一个栈(stack)来管理可用的管道，至于要不要给栈上锁，我觉得在这个例子里是不需要的：唯一的问题就是栈可能为空。

### LEC 2: syscall

#### trace
这题本身不难，只要想到用类似`p->mask & 1 << num`来判定flag，就解决了一半。

依照“没有困难，创造困难也要上”的最高指示，所有的编译环境是部署在Thinkpad T61上——一个年过15的老同志上。它来处理一个虚拟内存只有128M的虚拟机是绰绰有余的，但是问题出在这道题的测试样例上:
```C
// user/usertests.c:917

void
forkforkfork(char *s)
{
    ...
    while(1){
      int fd = open("stopforking", 0);
      if(fd >= 0){
        exit(0);
      }
      if(fork() < 0){
        close(open("stopforking", O_CREATE|O_RDWR));
      }
    }
    ...
    wait(0);
    ...
}
```

由于`fork()`本身的特性，它的返回时间依赖它的所有子进程。所以在通过了所有其他测试样例的大前提下，这个测试却没有任何输出：

![大概是这样](/images/syscall-trace.png)

在继约半个小时的撸码后的两个小时debug里，我一度怀疑人生。心里想着再不行就睡觉了的最后一次尝试，结果洗完澡回来就有输出了。
最后发现bug在电脑上，所有`fork()`执行完需要5min，而测试的时限是30s...

换成7900x跑了一下，只要1.3s，不得不感叹芯片的发展之快。

### To be continue...