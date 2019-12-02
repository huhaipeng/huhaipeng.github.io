# epoll、select、poll

epoll 和select本质上都是调用poll。poll函数在各自的文件系统中自己实现，具体在`file_operation`结构体里注册。若无实现poll函数则无法使用epoll或者select。像普通磁盘文件，是无法使用epoll或者select的。一般使用epoll或者select的都是socket。其他的像inotify文件系统也是可以使用epoll或者select的。

 同步非阻塞其实是两个过程，首先等待就绪，然后同步复制。等待就绪也就是等待缓冲区内有数据（或者发送缓冲区有空余位置）。而磁盘文件（也就是regular file）是没有就绪这过程的，它们随时是就绪的，至少模型上是这样，因此不适用同步非阻塞。 

https://www.zhihu.com/question/52989189



阻塞和非阻塞的区别是：

阻塞是在fd没有准备好的情况下继续等待，而在非阻塞是在fd没准备好的情况下马上返回。在fd准备好的情况下阻塞和非阻塞都是等待数据拷贝完成才返回。