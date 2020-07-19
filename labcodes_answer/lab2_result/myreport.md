# LAB2 物理内存管理
实验目的：

    理解基于段页式内存地址的转换机制
    理解页表的建立和使用方法
    理解物理内存的管理方法

实验内容

    本次实验包含三个部分。首先了解如何发现系统中的物理内存；
    然后了解如何建立对物理内存的初步管理，即了解连续物理内存管理；
    最后了解页表相关的操作，即如何建立页表来实现虚拟内存到物理内存之间的映射，对段页式内存管理机制有一个比较全面的了解。
    本实验里面实现的内存管理还是非常基本的，并没有涉及到对实际机器的优化，比如针对 cache 的优化等。如果大家有余力，尝试完成扩展练习。
## 速览代码
关注和lab1中不同的代码
- bootasm.c
- entry.S
- init.c
    新增函数调用`pmm_init()`，该函数定义在`pmm.c`中，是物理内存管理的关键函数。

## exercise 1
重写pmm_manager中的几个函数：
> default_init,default_init_memmap,default_alloc_pages, default_free_pages.
采用`fisrt fit mem-alloc`的算法。

1. prepare <br/>
首先看一下`list_entry_t`和相关的函数，他们定义在`libs/list.h`。

2. `default_init`<br/>
根据描述我们可以重用原本的demo，这里有一个结构体,
将在稍后给出

3. `default_init_memmap` <br />
用于初始化一个释放掉的block，使用addr_base 和page_number.其中page结构体定义在
memlayout.h中，它的成员在`defualt_pmm.c：24`中有解释.此处还有这部分具体的做法。

注意：
- free_list 中链接的是页头用来记录总页数，被设置了PG_property位，
- page->page_link 也是一个`list_entry_t`结构，free_list上其实是一串page的page_link域.

4. `default_alloc_pages` <br />
这个函数用来申请内存.这个地方少一张图
TODO

5. `default_free_pages` <br />

TODO



