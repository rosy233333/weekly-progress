# vdso私有数据实现调研

继续[vdso在用户态的重定位调研](25.6.19~25.6.25\vdso私有数据实现调研.md)中的调研，用于研究vdso是否支持每个地址空间拥有自己的私有数据（例如，全局变量）拷贝。

## 共享库的私有数据实现机制

共享库的私有数据的物理页共享和写时复制特性，均由将文件映射到内存的`mmap`函数提供。

当不同的进程使用`mmap`函数映射同一个文件时，其会将文件内容仅放在一块物理内存区域，再将这块物理内存映射到多个进程。而当在调用`mmap`时启用了`MAP_PRIVATE` flag时，对文件的修改即会发生写时复制。

[认真分析mmap：是什么 为什么 怎么用](https://www.cnblogs.com/huxiao-tee/p/4660352.html)

在glibc动态链接器中，`glibc/elf/dl-map-segments.h`中使用`mmap`加载共享库，且启用了`MAP_COPY`flag。在Linux中，`MAP_COPY`起到了与`MAP_PRIVATE`类似的作用。

```C
// glibc/elf/dl-load.h

/* The right way to map in the shared library files is MAP_COPY, which
   makes a virtual copy of the data at the time of the mmap call; this
   guarantees the mapped pages will be consistent even if the file is
   overwritten.  Some losing VM systems like Linux's lack MAP_COPY.  All we
   get is MAP_PRIVATE, which copies each page when it is modified; this
   means if the file is overwritten, we may at some point get some pages
   from the new version after starting with pages from the old version.

   To make up for the lack and avoid the overwriting problem,
   what Linux does have is MAP_DENYWRITE.  This prevents anyone
   from modifying the file while we have it mapped.  */
#ifndef MAP_COPY
# ifdef MAP_DENYWRITE
#  define MAP_COPY      (MAP_PRIVATE | MAP_DENYWRITE)
# else
#  define MAP_COPY      MAP_PRIVATE
# endif
#endif
```

因此，通过vdso实现的共享库因为没有通过`mmap`映射到真正的文件，因此应该无法通过该方式实现私有数据的写时复制。
