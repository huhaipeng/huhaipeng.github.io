---
layout:     post
title:      Linux内核调试 之 qemu+gdb
subtitle:   
date:       2019-12-29
author:     HHP
header-img: img/post-bg-unix-linux.jpg
catalog: true
tags:
    - linux 
---

## 前言

### 关于QEMU

qemu是一个广泛使用的开源计算机模拟器和虚拟机。

当作为模拟器时，可以在一种架构（如x86 PC）下运行另一种架构（如ARM）下的操作系统和程序。通过使用动态翻译，它可以获得非常好的性能。

作为虚拟机时，qemu可以使用其他虚拟机管理程序（ [KVM](https://wiki.archlinux.org/index.php/KVM_(简体中文))）来使用CPU扩展进行虚拟化，通过在主机CPU上直接执行客户机代码来获得接近于宿主机的性能。

### 关于内核调试

目前调试 Linux 内核与模块主要有 printk, /proc 和 kgdb 等方法，其中最常用的的 printk。

#### printk

printk 是调试内核代码时最常用的一种技术。在内核代码中的特定位置加入 printk() 调试调用，可以直接把所关心的信息打印到屏幕上，从而可以观察程序的执行路径和所关心的变量、指针等信息。

在使用 printk 时要注意优先级的问题。通过附加不同日志级别（loglevel），或者说消息优先级，可让 printk 根据这些级别所标示的严重程度，对消息进行分类。一般采用宏来指示日志级别。在头文件 <linux/kernel.h> 中定义了 8 种可用的日志级别字符串：KERN_EMERG，KERN_ALERT，KERN_CRIT，KERN_ERR，KERN_WARNING，KERN_NOTICE，KERN_INFO，KERN_INFO。共有 8 种优先级，用户可以根据需要进行配置，也可以通过 proc/sys/kernel/printk 动态修改设置。

使用 printk 来调试内核，明显的优点是：门槛低，上手快，定位问题有帮助。其缺点是要不断的加打印和重编内核。由于 syslogd 会一直保持对其输出文件的同步刷新，每打印一行都会引起一次磁盘操作，因此大量使用 printk 会严重降低系统性能。

#### /proc 文件系统

在 /proc 文件系统中，对虚拟文件的读写操作是一种与内核通信的手段。/proc 文件系统是一种特殊的、由程序创建的文件系统，内核使用它向外界输出信息。/proc 下面的每个文件都绑定于一个内核函数，这个函数在文件被读取时，动态地生成文件的“内容”。例如，/proc/modules 列出的是当前载入模块的列表。

#### kdb

kdb 是 Linux 内核的补丁，它提供了一种在系统能运行时对内核内存和数据结构进行检查的办法。kdb 还有许多其他的功能，包括单步调试（根据指令，而不是 C 源代码行），在数据访问中设置断点，反汇编代码，跟踪链表，访问寄存器数据等等。加上 kdb 补丁之后，在内核源码树的 Documentation/kdb 目录可以找到完整的手册页。

kdb 的优点是不需要两台机器进行调试。缺点是只能在汇编代码级进行调试。

#### gdb

gdb 全称是 GNU Debugger，是 GNU 开源组织发布的一个强大的 UNIX 下的程序调试工具。gdb 主要可帮助工程师完成下面 4 个方面的功能：

- 启动程序，可以按照工程师自定义的要求随心所欲的运行程序。
- 让被调试的程序在工程师指定的断点处停住，断点可以是条件表达式。
- 当程序被停住时，可以检查此时程序中所发生的事，并追索上文。
- 动态地改变程序的执行环境。

#### kgdb

kgdb 是一个在 Linux 内核上提供完整的 gdb 调试器功能的补丁，不过仅限于 x86 系统。它通过串口连线以钩子的形式挂入目标调试系统进行工作，而在远端运行 gdb。使用 kgdb 时需要两个系统——一个用于运行调试器，另一个用于运行待调试的内核（也可以是在同一台主机上用 vmware 软件运行两个操作系统来调试）。和 kdb 一样，kgdb 目前可从 oss.sgi.com 获得。

使用 kgdb 可以进行对内核的全面调试，甚至可以调试内核的中断处理程序。如果在一些图形化的开发工具的帮助下，对内核的调试将更方便。但是，使用 kgdb 作为内核调试环境最大的不足在于对 kgdb 硬件环境的要求较高，必须使用两台计算机分别作为 target 和 development 机。尽管使 用虚拟机的方法可以只用一台 PC 即能搭建调试环境，但是对系统其他方面的性能也提出了一定的要求，同时也增加了搭建调试环境时复杂程度。另外，kgdb 内 核的编译、配置也比较复杂，需要一定的技巧。当调试过程结束后时，还需要重新制作所要发布的内核。使用 kgdb 并不能 进行全程调试，也就是说 kgdb 并不能用于调试系统一开始的初始化引导过程。



#### 使用qemu + gdb调试内核的优势

* 配置简单，无需对内核编译打kgdb补丁。
* 轻量级，无需使用两台机器进行调试，对系统的性能要求低。



## 正文

### 搭建所需环境

1、编译需要进行调试的linux内核代码。

* 进入linux源码目录，先执行`make defconfig`生成默认.config文件（也可跳过这一步，直接`make menuconfig`），然后执行`make menuconfig`，选择自己需要的编译选项。最后需要勾选Kernel hacking-->Compiler-time checks and compiler options --->Compile the Kernel with debug info(不用内核版本选项名字可能稍有出入，但大致相同)，这一步 相当于我们平时运行gdb -g的效果，在生成的内核二进制中包含调试信息。
* 保存之后执行编译（make && make install）。详细过程可参考 [linux内核入门]([https://huhaipeng.top/2018/08/11/linux%E5%85%A5%E9%97%A8/](https://huhaipeng.top/2018/08/11/linux入门/))。

2、安装qemu

在debian系环境中，直接执行`apt install qemu`即可完成安装。

3、下载安装编译根文件系统busybox

从[busybox官网](https://busybox.net/about.html)下载源码，解压 ，然后cd进源码目录，运行

```
make menuconfig
```

进入第一行的"Busybox Settings"，在"Build Options"节中，选中“Build Busybox as a static binary”。

还有安装目录，默认是当前目录的_install目录下。设置好后退出保存配置，然后

```
make && make install
```

进入当前目录的_install下 ，就可以看到生成的东西。

至此，busybox编译完成，好容易。

4、制作rootfs磁盘镜像文件。

先用qemu-img命令生成磁盘镜像文件：

```
qemu-img create qemu_rootfs.img  1g
```

  其中qemu_rootfs.img是文件名，1g是磁盘大小，根据需要修改。

  创建ext4文件系统

```
mkfs.ext4 qemu_rootfs.img
```

5、挂载img文件到宿主系统：

```
sudo mount -o loop qemu_rootfs.img  qemu_rootfs
```

这里-o loop的意思是将qemu_rootfs.img作为硬盘文件，挂载在qemu_rootfs目录下。

挂载之后就可以在qemu_rootfs里面对qemu_rootfs.img进行操作了。

先把busybox的_install目录下的东西往里面拷贝，再创建一些目录、文件：

```
cd qemu_rootfs
sudo mkdir proc sys dev etc etc/init.d
```

  新建etc/init.d/rcS，内容如下：

```
#!/bin/sh
mount -t proc none /proc
mount -t sysfs none /sys
/sbin/mdev -s
```

  然后修改rcS文件权限，加上x。当busybox的init运行起来后，就会运行这个/etc/init.d/rcS脚本。

  最后卸载文件系统：

```
sudo umount qemu_rootfs
```

  至此，文件系统就制作完成了。

补充一点，这里我们制作的rootfs.img除了是一个真正的根文件以外，还承担了作为initrd.img的作用，如下图，initrc即为我们的initrd.img。系统利用initrd初始化完成之后，会挂载真正的根文件系统，这里的根文件系统也就是我们的 qemu_rootfs.img。

![](https://ftp.bmp.ovh/imgs/2019/12/cb68e573aa613171.png)

6、给gdb打补丁

原版gdb在用来调试内核的时候，会报`Remote 'g' packet reply is too long`的error。我们需要改一下gdb的代码，然后重新编译安装。

需要修改的代码在`gdb/remote.c`  8037行左右。

```c
    if (buf_len > 2 * rsa->sizeof_g_packet){
    /* 需要注释的原版代码 */
    //error (_("Remote 'g' packet reply is too long (expected %ld bytes, got %d "
	  //     "bytes): %s"), rsa->sizeof_g_packet, buf_len / 2, rs->buf);
    
    /* 新增补丁代码 */
    rsa->sizeof_g_packet = buf_len ;
    for (i = 0; i < gdbarch_num_regs (gdbarch); i++) {
        if (rsa->regs[i].pnum == -1)
            continue;
        if (rsa->regs[i].offset >= rsa->sizeof_g_packet)
            rsa->regs[i].in_g_packet = 0;
        else  
            rsa->regs[i].in_g_packet = 1;
    } 
   
  }
```

然后直接执行

```shell
make && make install
```



### 尝试使用qemu运行虚拟机

  激动人心的时候来了。我们来运行一个迷你的虚拟的Linux系统：

```
qemu-system-x86_64  \
    -kernel vmlinuz  \
    -hda qemu_rootfs.img \
    -append "root=/dev/sda rootfstype=ext4 rw" \
    --enable-kvm
```

 因为系统本身精简，没什么外设，加上qemu牛逼的速度，虚拟机很快就开完机了。这里我们制作的rootfs.img除了是一个真正的根文件以外，还承担了作为initrd.img的作用，如下图，initrc即为我们的initrd.img。

### 使用qemu + gdb进行内核调试

1、运行

```shell
qemu-system-x86_64 \
-kernel vmlinuz \
-hda qemu_rootfs.img \
-append "root=/dev/sda rootfstype=ext4 rw nokaslr"\ #这里nokaslr是关闭地址随机化，不关无法用gdb调试
-s -S # -s的意思是等待外面gdb的链接，默认开启1234端口进行监听；-S是在内核的入口打断点。
```

运行之后会看到运行了一个新的qemu终端窗口。

![](https://ftp.bmp.ovh/imgs/2019/12/5a0ae44b6f6b5a00.png)

2、新起一个终端运行

```shell
gdb vmlinux # vmlinx就是编译完成之后在源码x86/arch/目录下面的未压缩二进制文件
```

3、在gdb进程里执行

```
target remote localhost：1234
```

执行之后会链接到qemu里面，如下图显示。

![](https://ftp.bmp.ovh/imgs/2019/12/d36eb2a372186767.png)

4、尝试打断点（下图是在start kernel里面打的断点）。发现确实在该断点停住了。

![](https://ftp.bmp.ovh/imgs/2019/12/1bad8e36ff00e1fb.png)

5、gdb里面输入continue，代码继续跑，如下图，证明我们成功了。

![](https://ftp.bmp.ovh/imgs/2019/12/5a85daeb92ba8801.png)





——END——





