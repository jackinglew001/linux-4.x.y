Vmalloc
========================================

我们知道物理上连续的映射对内核是最好的，但并不总能成功地使用。在分配一大块内存时，可能竭尽全力
也无法找到连续的内存块。在用户空间中这不是问题，因为普通进程设计为使用处理器的分页机制，当然
这会降低速度并占用TLB。在内核中也可以使用同样的技术。内核分配了其虚拟地址空间的一部分，用于建立
连续映射。类似如下所示:

https://github.com/novelinux/linux-4.x.y/tree/master/arch/arm/mm/README.md

在ARM系统中，紧随直接映射的前760MiB物理内存，在插入的8 MiB安全隙之后，是一个用于管理不连续内存
的区域。这一段具有线性地址空间的所有性质。分配到其中的页可能位于物理内存中的任何地方。通过修改
负责该区域的内核页表，即可做到这一点。

每个vmalloc分配的子区域都是自包含的，与其他vmalloc子区域通过一个内存页分隔。类似于直接映射和
vmalloc区域之间的边界，不同vmalloc子区域之间的分隔也是为防止不正确的内存访问操作。这种情况只会
因为内核故障而出现，应该通过系统错误信息报告，而不是允许内核其他部分的数据被暗中修改。因为分隔
是在虚拟地址空间中建立的，不会浪费宝贵的物理内存页。

Data Structure
----------------------------------------

### struct vm_struct

https://github.com/novelinux/linux-4.x.y/blob/master/include/linux/vmalloc.h/struct_vm_struct.md

### struct vmap_area

https://github.com/novelinux/linux-4.x.y/blob/master/include/linux/vmalloc.h/struct_vmap_area.md

APIS
----------------------------------------

### Allocation

#### vmalloc

vmalloc是一个接口函数，内核代码使用它来分配在虚拟内存中连续但在物理内存中不一定连续的内存。

path: include/linux/vmalloc.h
```
extern void *vmalloc(unsigned long size);
```

该函数只需要一个参数，用于指定所需内存区的长度，与此前讨论的函数不同，其长度单位不是页而是字节，
这在用户空间程序设计中是很普遍的。使用vmalloc的最著名的实例是内核对模块的实现。因为模块可能在
任何时候加载，如果模块数据比较多，那么无法保证有足够的连续内存可用，特别是在系统已经运行了比较
长时间的情况下。如果能够用小块内存拼接出足够的内存，那么使用vmalloc可以规避该问题。内核中还有
大约400处地方调用了vmalloc，特别是在设备和声音驱动程序中。

因为用于vmalloc的内存页总是必须映射在内核地址空间中，因此使用ZONE_HIGHMEM内存域的页要优于
其他内存域。这使得内核可以节省更宝贵的较低端内存域，而又不会带来额外的坏处。因此，vmalloc
是内核出于自身的目的(并非因为用户空间应用程序)使用高端内存页的少数情形之一。

https://github.com/novelinux/linux-4.x.y/tree/master/mm/vmalloc.c/vmalloc.md

#### vmalloc_32

工作方式与vmalloc相同，但会确保所使用的物理内存总是可以用普通32位指针寻址。如果某种体系结构的
寻址能力超出基于字长计算的范围，那么这种保证就很重要。

#### vmap

使用一个page数组作为起点，来创建虚拟连续内存区。与vmalloc相比，该函数所用的物理内存位置不是
隐式分配的，而需要先行分配好，作为参数传递。此类映射可通过vm_map实例中的VM_MAP标志辨别。

#### ioremap

是一个特定于处理器的函数，必须在所有体系结构上实现。它可以将取自物理地址空间、由系统总线用于
I/O操作的一个内存块，映射到内核的地址空间中。

### Free

```
vfree         vunmap
 |              |
 +-> __vunmap <-+
     |
     +-> remove_vm_area
     |
     +-> __free_page
```

有两个函数用于向内核释放内存，vfree用于释放vmalloc和vmalloc_32分配的区域，而vunmap用于释放由
vmap或ioremap创建的映射。这两个函数都会归结到__vunmap.

#### vfree

https://github.com/novelinux/linux-4.x.y/tree/master/mm/vmalloc.c/vfree.md

#### vunmap

https://github.com/novelinux/linux-4.x.y/tree/master/mm/vmalloc.c/vunmap.md
