# xv6-lab学习笔记


**注：标题所指的xv6是MIT的计算机操作系统课程[S.081](https://pdos.csail.mit.edu/6.828/2020/schedule.html)。**

### LEC 1: util

#### 环境部署

上来就在环境部署上遇到了点小问题，虽然别人不太可能遇到，但仍记录在此，仅作参考。

我用的是Gentoo Linux，通过[crossdev](https://wiki.gentoo.org/wiki/Crossdev)编译出的``riscv64-unkown-linux-gnu``是不可用的，个人猜测是因为它链接了Linux的一些库，而xv6是裸二进制的elf环境。（对于toolchain并不熟悉，欢迎指正）。

Gentoo官方配置编译出的[qemu](https://packages.gentoo.org/packages/app-emulation/qemu)对于xv6也是不可用的状态,没有任何输出（QEMU_SOFTMMU_TARGETS=&#34;riscv64&#34;），可能是版本太高了导致不兼容...

&gt; 2024.7.4更新：
&gt; 关于qemu没有输出的问题，可以换用2023版的xv6(**git clone git://g.csail.mit.edu/xv6-labs-2023**)。

二者的解决方案是一样的，就是按照课程的教程下载编译安装对应的软件。安装位置可以选在`/opt`,最后在`.[zsh,bash]rc`里加上:`export PATH=&#34;/opt/qemu/bin:/opt/riscv-gcc/bin:%PATH&#34;`

#### xargs.c
一开使有一个小问题没注意到，xv6要求的xargs是将每一行的输入都与argv单独合并。

比如，假设xv6的echo支持转义符，`echo &#34;hello\nworld&#34; | xargs echo print:`，应该打印出两行两个`print:`，而非两行一个`print:`

所以最后执行`exec`的时候，应该维护一个`*xargv[]`,将`argv[1:]`与`stdin`的一行输入进行合并。

同时，应该注意到`read()`读取到`\n`的时候应该在字符串末尾写入`\0`而非`\n`。

#### primes.c
这个程序难就难在debug环节上，很难仅靠程序本身打印log的方式纠错。我的解决方案是替换掉头文件(*unistd.h*)然后在本地环境下编译调试。

需要注意到如果在父进程中关闭了管道(pipe)，管道里的缓存会被同时清空。所以父进程可以在输出完后再写入一个**0(EOF)**，并交由子进程关闭管道。

在编写的时候**不宜过早优化**，可以在确定算法正确性后再优化管道数使用的问题。可以用一个栈(stack)来管理可用的管道，至于要不要给栈上锁，我觉得在这个例子里是不需要的：唯一的问题就是栈可能为空。

### LEC 2: syscall

#### trace
这题本身不难，只要想到用类似`p-&gt;mask &amp; 1 &lt;&lt; num`来判定flag，就解决了一半。

依照“没有困难，创造困难也要上”的最高指示，所有的编译环境是部署在Thinkpad T61上——一个年过15的老同志上。它来处理一个虚拟内存只有128M的虚拟机是绰绰有余的，但是问题出在这道题的测试样例上:
```C
// user/usertests.c:917

void
forkforkfork(char *s)
{
    ...
    while(1){
      int fd = open(&#34;stopforking&#34;, 0);
      if(fd &gt;= 0){
        exit(0);
      }
      if(fork() &lt; 0){
        close(open(&#34;stopforking&#34;, O_CREATE|O_RDWR));
      }
    }
    ...
    wait(0);
    ...
}
```

由于`fork()`本身的特性，它的返回时间依赖它的所有子进程。所以在通过了所有其他测试样例的大前提下，这个测试却没有任何输出：

![大概是这样](/images/syscall-trace.png)

在继约半小时撸码后的两个小时debug时间里，一度怀疑人生。
最后发现bug在电脑上，所有`fork()`执行完需要5min，而测试的时限是30s...

换成7900x跑了一下只要1.3s...

### LEC 4: pgtbl

这里的三道题都要求你看懂与其相关的大部分函数，也或多或少都是可以通过魔改现有函数来解题的。
比如`copyin`里，要在`exec()`里将对进程页表的修改同时也映射到私有的内核页表里。直接修改`uvmalloc()`应该也是可行的，
但是也可以借鉴一下`uvmcopy()`来一次性的映射所有内存。

题目有的指示比较隐晦，比如*You&#39;ll need a way to free a page table without also freeing the leaf physical memory pages*
看上去似乎要找出什么小trick，但实际上只要看一眼`freewalk()`的代码就明白了。

### LEC 8 lazy:

题目相对比较简单，也是因为主要思路和代码已经在课上给出。

但还有一些小坑：第三问第四小条——
*Handle the case in which a process passes a valid address from sbrk() to a system call such as read or write, but the memory for that address has not yet been allocated.*
简单阅读源代码后可以定位到`syscall.c`里的`argaddr()`函数，简单的复制`trap.c`里的修改是通过不了`validatetest`的。
首先需要调用`walkaddr`函数验证`*ip`是否已经分配过了。然后很重要的一点来了：
如果地址不合法，`p-&gt;killed = 1`并不会像在`usertrap()`里那样使进程正确的退出，应直接`return -1`交由对应的SYS_*函数处理。


### LEC 10 COW:

这个lab是我目前花时间最多的一个，比pgtbl还多。原因有几个，题目给的提示比较少，至少相对于它的难度而言；
再有我个人的问题，前期急于验证程序的正确性而到处添加修改，存有侥幸心理，忽略了这题可以将函数解耦从而简化调试。

一开始开了一个`ref_count[PHYSTOP/PGSIZE]`的大数组用来引用计数，然后想着全部初始化为0会方便一些，结果有些页总是会被提前释放。
之后改成了初始化为1，父进程的引用也算一次，逻辑上也更说得通。

最麻烦的是要通过`usertests`，很多组测试都会在`memmove`上触发`kerneltrap`。用gdb看了一下，参数`src`，和`dst`都会出现为0的情况，
说明复制COW页面的时候缺少了检查`pa`和`va`的步骤。这些条件在`usertrap()`和`copyout()`里都要用到。

这里也发现了一个使用gdb调试的小技巧——要想跳转到`memmove`在内核崩溃前的最后一次（接近）执行，可以先加载用户程序的符号，在测试函数那里断下。再加载内核符号，在`memmove`断下，这样就不用一步一步跳转了。

### LEC 11 thread:

这一个lab比较简单，唯一一个卡住的点在第一题。在`thread_create()`里忘记了栈是向上增长的，把寄存器`sp`直接指向了`t-&gt;stack`。
这里导致了一个很有意思的结果，`uthread`只会运行c的循环，输出类似于：
```
$ uthread
thread_a started
thread_b started
thread_c started
thread_c 0
thread_c 1
thread_c 2
...
thread_c 99
thread_c: exit after 100
thread_schedule: no runnable threads
$ 
```
这是因为stack在向下生长的时候把内存里相邻的上一个`thread`结构体改写了，其中的`state`如果被改写成别的，那么`thread_schedule`就可能永远不会切换到它。

### LEC 13 lock:

又是一个比较难的lab，但运气很好，没怎么调试就过了——**GDB无启动通关**（逃。这个实验的描述很长，给的hints也不是按线索一条一条的，
所以并没有一步一步边写边调试地推进；而是在脑子里构思好后直接一步到位。

`bcache`这里可以参考一下[2023](https://pdos.csail.mit.edu/6.828/2023/labs/lock.html)年的提示，相比于[2020](https://pdos.csail.mit.edu/6.828/2020/labs/lock.html)年，
并没有要求使用`ticks`来选择`buf`。我在删去有关`ticks`的所有判断后，从之前的循环700&#43;直接降到稳定50&#43;
个人认为这是一个取巧妥协的解法，只判断`refcnt == 0`肯定会释放掉一些最近使用的缓存。但这只是没有profiling的一些猜测

至于hash table的实现，我借用了上一个lab里`ph.c`的实现，开了13个`bucket`，都初始化为空的链配上一个锁。
`bget()`如果在哈希表里搜不到就会直接去`buf`里找`refcnt == 0`，`buf`就是原本的`bcache`结构体里的`buf[NBUF]`。
这样实现可以直接复用`buf`结构体里的链表，比较简单。

### LEC 15 FS:

两个lab标定的难度都是中等，看文档的难度却是困难：几层封装下来十几个重要的函数加上“若隐若现”的locks可以说是隐藏难度的巅峰之作了。

第一个：支持大文件，报错**panic: virtio_disk_intr status**，查了半天发现是`inode`结构体没改，导致`addrs`大小变成12了。但想了半天也没明白为什么会是在这里panic

第二个：软连接，提示说把`target`存到一个地方，可以是“inode&#39;s data blocks”，我理解成在`inode`结构体里加一个字符串存储。
又专门写了个函数去递归找原始文件。写的时候才发现还有这么个函数`writei`，才明白“data blocks”就是字面意思……

总之就是多看看文档，关注一下关键函数。

### LEC 17 mmap:

大杂烩lab，文件系统、页表和中断都涉及到了。工程量比较大，但难度不大。

实现完`mmap`后遇到一个问题：`mmaptest`在test1报错读不到0。看了一下test1的代码，它只往文件里写了3/2个`PGSIZE`（6144）的&#39;A&#39;，所以应该是期望出发page fault后惰性分配的时候先把页初始化成0。

还有个问题就是`filewrite`函数会导致调用`sched`，所以不能在持有锁的情况下调用它。以及一些`vm.c`里的函数会检查`PTE_V`，都可以忽略掉，因为我们是lazy alloc，会出现这种情况。

### To be continue...


---

> 作者: Jason Li  
> URL: http://localhost:1313/zh-cn/posts/2296229/  

