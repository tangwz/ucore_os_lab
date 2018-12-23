# 实验报告Lab2

# 实验目的
* 理解基于段页式内存地址的转换机制
* 理解页表的建立和使用方法
* 理解物理内存的管理方法

# 实验内容
## 练习0：填写已有实验
本实验依赖实验1。请把你做的实验1的代码填入本实验中代码中有“LAB1”的注释相应部分。

首先找出 “LAB1” 的位置，使用命令：
`grep -r "LAB1" lab2`
可以看到需要对 lab2/kern/trap/trap.c 和 lab2/kern/debug/kdebug.c 文件进行填入。

再使用 diff 命令制作 patch 补丁：

`diff -Nur lab2/kern/trap/trap.c lab1/kern/trap/trap.c > /tmp/ucore1.patch`

再使用 patch 命令打上补丁：

`patch -p0 < /tmp/ucore1.patch`

使用同样的命令对 lab2/kern/debug/kdebug.c  打上补丁。


## 练习1：实现 first-fit 连续物理内存分配算法
在实现 first fit 内存分配算法的回收函数时，要考虑地址连续的空闲块之间的合并操作。提示:在建立空闲页块链表时，需要按照空闲页块起始地址来排序，形成一个有序的链表。可能会修改 `default_pmm.c` 中的 `default_init`，`default_init_memmap`，`default_alloc_pages`， `default_free_pages`等相关函数。请仔细查看和理解 `default_pmm.c` 中的注释。

### 1. 准备
要实现 First-Fit Memory Allocation (FFMA)，我们使用双向列表 `list` 管理空闲内存，数据结构 `free_area_t` 用来管理空闲内存。（位于 `libs/list.h`）
使用下列的宏定义可以将 `list` 结构转换为制定的数据结构（例如 `page`）：`le2page`(in memlayout.h), (and in future labs: `le2vma` (in vmm.h), `le2proc` (in proc.h), etc).

### default_init
可以使用 `default_init` 函数来初始化 `free_list` 和置 `nr_free` 为 0，`free_list` 用来记录空闲内存块，`nr_free` 是空闲块的数量。

### default_init_memmap
调用图：`kern_init` --> `pmm_init` --> `page_init` --> `init_memmap` --> `pmm_manager` --> `init_memmap`
该函数用来初始化一个空闲块（使用参数 `addr_base`, `page_number`）。为了初始化一个空闲块，首先需要初始化该块的每一页（定义在 `memlayout.h`），这一流程包括：
 - 设置 `p->flags` 为 `PG_property`，即这一页可用。在函数 `pmm_init` 中，`p->flags` 已经被置为 `PG_reserved`；
 - 如果该页为空，且不是空闲块的第一页，那么 `p->property` 被设置为 0；
 - 如果该页为空，且是空闲块的第一页，那么 `p->property` 为该块的总页数；
 - `p->ref` 应被设置为 0，因为当前 `p` 为空且没有引用。
 - 在这之后，我们使用 `p->page_link` 指向 `free_list`。(e.g.: `list_add_before(&free_list, &(p->page_link));` )
 - 最后，我们更新空闲内存块的总数： `nr_free += n`，

###  default_alloc_pages
查找 free list 的第一个空闲块（块的 size >= n），然后 resize 该块，返回该块的地址作为 `malloc` 的参数。
- 可以使用如下方式搜索 free list：

```
list_entry_t le = &free_list;
while((le=list_next(le)) != &free_list) {
...
```

 - 在整个循环中，检查 `page` 的 `p->property` 是否 >= n

```
struct Page *p = le2page(le, page_link);
if(p->property >= n){ ...
```

- 如果获得了 `p`，意味着我们找到了一个空闲块且其大小 >= n，该块的前 n 页可以被分配。该页的一些标志位需要设置如下：PG_reserved = 1`, `PG_property = 0`. 然后从 free list unlink 该页。
- 如果 `p->property > n` ，我们需要重新计算该块的剩余页。(e.g.: `le2page(le,page_link))->property = p->property - n;`)
- 重新计算 `nr_free` (number of the the rest of all free block).
- 返回 `p`
- 如果我们找不到一个空闲块 >= n，则返回 `NULL`

### default_free_pages
re-link free list 中的页，然后把一些小的块合并为大的。
- According to the base address of the withdrawed blocks, search the free list for its correct position (with address from low to high), and insert the pages. (May use `list_next`, `le2page`, `list_add_before`)

- Reset the fields of the pages, such as `p->ref` and `p->flags` (PageProperty)

- Try to merge blocks at lower or higher addresses. Notice: This should change some pages' `p->property` correctly.

