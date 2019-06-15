## linux 内存管理

页置换：

当物理内存不足时，会选择一些页面flush回磁盘或swap分区。

swap分区：暂存没有文件背景的页面，即**匿名页（anonymous page）**，如堆，栈，数据段等。

磁盘的其他分区：存文件页

参考：

https://blog.csdn.net/jasonchen_gbd/article/details/79462014