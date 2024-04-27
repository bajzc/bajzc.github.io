---
category: |
  Review
date: "2023-07-14 12:19:35 UTC+07:00"
lang: |
  en
slug: |
  cachestat-in-kernel-65-rc1
tags: ["kernel", "sys_call", "linux"]
title: cachestat(), a new syscall in Kernel 6.5-rc1
translation: |
  true
---

![](/images/Tex_6.5.png){.align-center}

:::: note
::: title
Note
:::

The environment we are in:

Linux gentoo 6.5.0-rc1
::::

::: contents
:::

If you want to try those demos below, you need to know:

1.  generate random files for test

``` bash
#signle random file "a"
dd if=/dev/urandom of=a bs=1M count=128

# random files "f1"-"f8"
for i in {1..8}
do
dd if=/dev/urandom of=f${i} bs=1M count=128
done
```

2.  clean all the cache:

`# echo 3 > /proc/sys/vm/drop_caches`

:::: warning
::: title
Warning
:::

The cache must be cleared every time before running the test program, if
you want to get accurate results. Because,

1.  when the file is generated, it has actually been written to the
    cache!
2.  after running any demo once, there may be cache remaining
::::

3.  limit memory:

For most systemd users, limit your memory to 1G can be done by:

`# echo 1073741824 > /sys/fs/cgroup/user.slice/memory.max`

:::: caution
::: title
Caution
:::

To remove the memory limit, DO NOT rewrite the file into empty or 0,

Instead, rewrite it into max:

`# echo max > /sys/fs/cgroup/user.slice/memory.max`
::::

:::: note
::: title
Note
:::

Since Glibc does not yet support the latest sys_call(), here we first
borrow the functions in the kernel to implement:

Install the header files from Kernel, [refer to
here](https://www.kernel.org/doc/Documentation/kbuild/headers_install.txt)

``` bash
# run as root
cd /usr/src/linux-6.5-rc1 #where the kernel source is
make headers_install ARCH=x86 INSTALL_HDR_PATH=/usr
```
::::

# 1. How to use it?

Given there is a 128M random file `a`. Read `a` and record the usage of
cache

`test1.cpp`:

``` C++
#include <fcntl.h>
#include <linux/mman.h>
#include <stdio.h>
#include <stdlib.h>
#include <sys/stat.h>
#include <sys/types.h>
#include <syscall.h>
#include <unistd.h>

// Implement the call interface yourself
long sys_cachestat(unsigned int fd,
        struct cachestat_range *cstat_range,
        struct cachestat *cstat, unsigned int flags) {
    return syscall(__NR_cachestat, fd, cstat_range, cstat, flags);
}

typedef struct {
    char fn[32];
    int fd;
    FILE *fp;
    struct cachestat cs;
} FNode;

int retrieve_cachestat(FNode *n) {
    struct cachestat_range cr = {0, 1024 * 1024 * 1024};
    int r = sys_cachestat(n->fd, &cr, &n->cs, 0);
    if (r != 0)
        perror("Failt to call sys_cachestat\n");
    else {
        printf("File [%s] has %d pages in cache, %d evicted\n", n->fn,
            (int)(n->cs.nr_cache), (int)(n->cs.nr_evicted));
    }
    return r;
}

// Read the file so that the file is written into cache as much as possible
void fetch_fileecontent(FNode *n) {
    FILE *fp = n->fp;
    char buf[8];
    while (1) {
        if (fread(buf, 1, sizeof(buf), fp) <= 0)
            break;
        if (fseek(fp, 4096, SEEK_CUR))
            break;
    }
}
int openfile(FNode *on) {
    on->fp = NULL;
    FILE *fp = fopen(on->fn, "r");
    if (fp == NULL)
        return -1;
    int fd = fileno(fp);
    on->fd = fd;
    on->fp = fp;
    return 0;
}

void closefile(FNode *on) {
    if (on->fp)
        fclose(on->fp);
    on->fp = NULL;
}

int main() {
    struct cachestat cs;
    FNode fn;
    sprintf(fn.fn, "a");
    openfile(&fn);
    retrieve_cachestat(&fn);
    fetch_fileecontent(&fn);
    retrieve_cachestat(&fn);
    closefile(&fn);
    return 0;
}
```

Output:

``` bash
❯ ./test1.out
File [a] has 0 pages in cache, 0 evicted
File [a] has 32768 pages in cache, 0 evicted
```

This means, in the process of reading the file, 32768 cache pages were
loaded.

We can check the unit page cache size by running: .. code-block:: bash

> ❯ getconf PAGESIZE 4096

matches our expectation, as `(128*1024*1024)/4096 = 32768`

# 2. Problem

Now we have 8 random files f1-f8 of 128M, which are read 128 times in a
loop:

`test2.cpp` (Just change the main() function)

``` C++
#define MAXN 8
FNode fns[MAXN];
int main() {
    int i, k;
    long counter = 0, m;
    for (k = 0; k < 128; k++) {
        for (i = 1; i <= MAXN; i++) {
            sprintf(fns[i].fn, "f%d", i);
            openfile(&fns[i]);
        }
        for (i = 1; i <= MAXN; i++) {
            retrieve_cachestat(&fns[i]);
            m = fns[i].cs.nr_cache;
            fetch_fileecontent(&fns[i]);
            retrieve_cachestat(&fns[i]);
            counter += fns[i].cs.nr_cache - m;
        }
        for (i = 1; i <= MAXN; i++)
            closefile(&fns[i]);
    }
    printf("Total %ld pages loaded\n", counter);
    return 0;
}
```

Output:

![image](/images/cache_test2.png){.align-center}

With sufficient memory, **7.5** seconds were used.

But what if the memory is just not enough? For example, we only have
**1G** of memory, however, we also need to allocate memory to the system
itself.

At this time, the new file (such as f8) will keep refreshing the cache
(the content of f1-f7), and after a loop, f1 will take up the cache of
other files.

This will cause a huge performance loss:

![image](/images/cache_test2_limited.png)

It took **50** seconds!

# 3. Use cachestat() to solve it

At this time, we can use the information provided by cachestat(), and in
each new cycle, sort the reading order according to the size of the
files in the cache.

``` bash
❯ diff test2.cpp test3.cpp
9a10
> using std::sort;
60c61,65
< FNode fns[MAXN];
---
> FNode fns[MAXN+4];
> int ix[MAXN+4];
> bool mycmp(int a,int b){
>   return fns[a].cs.nr_cache>fns[b].cs.nr_cache;
> }
70a76,80
>       ix[i-1]=i;
>     }
>     sort(ix,ix+MAXN,mycmp);
>     for(int j=0;j<MAXN;j++){
>       i=ix[j];
```

The whole file, `test3.cpp` :

``` C++
#include <fcntl.h>
#include <linux/mman.h>
#include <stdio.h>
#include <stdlib.h>
#include <sys/stat.h>
#include <sys/types.h>
#include <syscall.h>
#include <unistd.h>
#include <algorithm>
using std::sort;
long sys_cachestat(unsigned int fd, struct cachestat_range *cstat_range,
        struct cachestat *cstat, unsigned int flags) {
    return syscall(__NR_cachestat, fd, cstat_range, cstat, flags);
}
typedef struct {
    char fn[32];
    int fd;
    FILE *fp;
    struct cachestat cs;
} FNode;

int retrieve_cachestat(FNode *n) {
    struct cachestat_range cr = {0, 1024 * 1024 * 1024};
    int r = sys_cachestat(n->fd, &cr, &n->cs, 0);
    if (r != 0)
        perror("Failt to call sys_cachestat\n");
    else {
        // printf("File [%s] has %d pages in cache, %d evicted\n", n->fn,
        //        (int)(n->cs.nr_cache), (int)(n->cs.nr_evicted));
    }
    return r;
}
void fetch_fileecontent(FNode *n) {
    FILE *fp = n->fp;
    char buf[8];
    while (1) {
        if (fread(buf, 1, sizeof(buf), fp) <= 0)
            break;
        if (fseek(fp, 4096, SEEK_CUR))
            break;
    }
}
int openfile(FNode *on) {
    on->fp = NULL;
    FILE *fp = fopen(on->fn, "r");
    if (fp == NULL)
        return -1;
    int fd = fileno(fp);
    on->fd = fd;
    on->fp = fp;
    return 0;
}

void closefile(FNode *on) {
    if (on->fp)
        fclose(on->fp);
    on->fp = NULL;
}

#define MAXN 8
FNode fns[MAXN+4];
int ix[MAXN+4];
bool mycmp(int a,int b){
    return fns[a].cs.nr_cache>fns[b].cs.nr_cache;
}
int main() {
    int i, k;
    long counter = 0, m;
    for (k = 0; k < 128; k++) {
        for (i = 1; i <= MAXN; i++) {
            sprintf(fns[i].fn, "f%d", i);
            openfile(&fns[i]);
        }
        for (i = 1; i <= MAXN; i++) {
            retrieve_cachestat(&fns[i]);
            ix[i-1]=i;
        }
        sort(ix,ix+MAXN,mycmp);
        for(int j=0;j<MAXN;j++){
            i=ix[j];
            m = fns[i].cs.nr_cache;
            fetch_fileecontent(&fns[i]);
            retrieve_cachestat(&fns[i]);
            counter += fns[i].cs.nr_cache - m;
        }
        for (i = 1; i <= MAXN; i++)
            closefile(&fns[i]);
    }
    printf("Total %ld pages loaded\n", counter);
    return 0;
}
```

Output:

![image](/images/cache_test3.png)

At this time, the available memory is still **1G** , but it only took
**30** seconds!

# 4. Summary

In summary, the time taken for cyclic reading of f1-f8 files with a
total size of 1G:

Sufficient memory \> Insufficient memory, but reading sequence is
optimized according to cachestat \>\> Insufficient memory, and no
optimization

It can be seen that cachestat() is very helpful for the server to
provide dynamic planning read and write strategies when resources are
limited.

At present, the documentation is limited, you can refer to the
instructions attached to the submission in the git log:

`git log cf264e1329fb0307e044f7675849f9f38b44c11a`

SYNOPSIS:

``` C
#include <sys/mman.h>

struct cachestat_range {
        __u64 off;
        __u64 len;
};

struct cachestat {
        __u64 nr_cache;
        __u64 nr_dirty;
        __u64 nr_writeback;
        __u64 nr_evicted;
        __u64 nr_recently_evicted;
};

int cachestat(unsigned int fd, struct cachestat_range *cstat_range,
        struct cachestat *cstat, unsigned int flags);
```
