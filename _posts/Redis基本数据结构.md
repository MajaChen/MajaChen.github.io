五种基本数据结构

字符串：字符串

散列：map

列表：list

集合：set

有序集合：sortedset



SDS：Simple Dynamic String，简单动态字符串，有点类似于常见的动态数组

底层存储了一个C语言的字符串数组(以\0结尾)，还有这个数组用了多少，有多少空闲空间

1.安全性：防止缓冲区溢出

2.效率：快速获取字符串长度

减少缓冲区的分配次数：在分配的过程中优先使用“未被使用的空间”，而不是立即向操作系统申请空间

在释放的过程中将被释放的空间作为“未使用的空间”，而不是立即将空间归还给操作系统（未使用的空间什么时候会被释放回去？）



链表：用于。。。采用双向链表结构

字典：用于。。。多层级的结构，自底向上依次是键值对、散列表、字典。

字典包含了两个散列表，用于rehash，而且采用了渐进式hash策略。通过链地址法解决冲突。

