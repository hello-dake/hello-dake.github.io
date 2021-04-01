## HashMap详解

HashMap底层数据结构，在JDK1.8中是数组+红黑树+链表结构。1.8之前是数组和链表。

<table><tr><td bgcolor=yellow>为什么要改成数组+链表+红黑树？</td></tr></table>
为了提升查找性能，红黑树查找性能O(logn)，链表O(n)。







