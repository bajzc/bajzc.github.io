---
title: 浅析Kernel-6.5新系统调用
slug: cachestat-in-kernel-65
date: 2023-07-14T12:19:35+07:00
tags: 
- kernel 
- sys_call
- linux
categories:
- Review 
---

	.. figure:: /images/Tex_6.5.png
		:align: center
	

.. note::
	本文系统环境：
	
	Linux gentoo 6.5.0-rc1

如果你想自己尝试，你需要知道的几个命令：

1. 创建随机文件用于测试
	.. code-block:: bash
	
		#signle random file "a"
		dd if=/dev/urandom of=a bs=1M count=128
		
		# random files "f1"-"f8"
		for i in {1..8}
		do
		dd if=/dev/urandom of=f${i} bs=1M count=128
		done
		
#. 清除系统cache：

``root# echo 3 > /proc/sys/vm/drop_caches``
	.. warning::
		在每一次运行测试程序之前都应该清除缓存，因为在生成文件时，它其实已经被写入cache了！同时，在运行过一次程序后，可能会有文件残留的cache

#. 限制内存： ``root# echo 1073741824 > /sys/fs/cgroup/user.slice/memory.max``
	.. caution::
		要解除内存限制千万 **不能** 将文件改为空或者0，
		
		而应该将内容改为 **max**

.. note::
	由于glibc目前还没有支持最新的系统函数调用，这里我们先借用kernel里的函数实现：
	
	安装kernel的头文件， `参考这里 <https://www.kernel.org/doc/Documentation/kbuild/headers_install.txt>`_ ：

		.. code-block:: bash
	
			# run as root
			cd /usr/src/linux-6.5-rc1 #where the kernel source is
			make headers_install ARCH=x86 INSTALL_HDR_PATH=/usr

1. 如何使用？
=================

读取一个128M的随机文件a,记录cache的使用情况：

``test1.cpp``:

	.. code-block:: C++
	
		#include <fcntl.h>
		#include <linux/mman.h>
		#include <stdio.h>
		#include <stdlib.h>
		#include <sys/stat.h>
		#include <sys/types.h>
		#include <syscall.h>
		#include <unistd.h>
		
		// 自己来实现调用接口
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
		
		// 间隔读取文件，使文件尽可能的被写入cache
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

输出：

	.. code-block:: bash
	
		❯ ./test1.out
		File [a] has 0 pages in cache, 0 evicted
		File [a] has 32768 pages in cache, 0 evicted
		
说明读取文件的过程中，载入了32768个cache pages

查看单位页缓存大小
	.. code-block:: bash
	
		❯ getconf PAGESIZE
		4096

符合等式： (128*1024*1024)/4096 = 32768

2. 问题场景
=================

现在我们有8个128M的随机文件f1-f8，循环打开128次：

只需要修改main()函数

``test2.cpp``:

	.. code-block:: C++
	
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

输出：

.. image:: /images/cache_test2.png
	:align: center

在内存足够的情况下，使用了 **7.5** 秒。

但如果内存刚好不够用呢？比如我们只有1G内存，并且还需要给系统本身分配内存。这时，新的文件（比如f8）就会一直刷新cache（f1-f7的内容）,而一轮循环后，f1又会去抢占别的文件的cache。

这时就会造成巨大的性能损失：

.. image:: /images/cache_test2_limited.png

用时 **50** 秒！

3.运用cachestat解决问题
=============================================

这时我们就可以更具cachestat提供的信息，在每一轮新循环时，依据文件在cache中的大小排序读取顺序。
	.. code-block:: bash

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

完整文件：

``test3.cpp``:
	.. code-block:: C++
		
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

输出：

	.. image:: /images/cache_test3.png

此时可用内存仍然是1G,但只用了 **30** 秒！

4. 总结
=========

综上，对于总大小1G的f1-f8文件的循环读取：

内存充足 > 内存不足，但根据cachestat优化读取策略 >> 内存不足，且无优化

的时间消耗。

看得出，cachestat()对于服务器在资源有限时，提供动态规划读写策略很有帮助。这次6.5的这个更新其实也是将以前提供的接口统一成一个数据结构，更方便我们调用。

目前文档有限，可以参考git log中提交时附带的说明：

``git log cf264e1329fb0307e044f7675849f9f38b44c11a``

概要：

	.. code-block:: C
	
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
			    
.. note::
	参考
	B站博主@lddlinan的 `视频 <https://www.bilibili.com/video/BV1xj411d7yc/?share_source=copy_web&vd_source=97fae6e3f40019d75299f7c145705b5a>`_ 
