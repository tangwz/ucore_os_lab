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

