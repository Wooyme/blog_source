---
title: Linux 虚拟文件系统
date: 2021-04-28 22:36:08
tags: 
    - Linux
    - C
---

# 前言
很久没有写博客了。一是同居之后就懒了，二是也一直没有什么有意思的idea。一年多的时间里除了参加了下game-off2020也没干别的什么事。不过最近因为想做个东西，学习了一下虚拟文件系统VFS(Virtual File System)，然后遇到了一些坑，所以写篇文章，记录一下。

# VFS
众所周知，Linux下有个概念叫做“一切皆文件”。关于这个概念有个很明显的体现，那就是`/dev`目录。我们进入`/dev`目录可以发现许多“虚假”的文件，像是`/dev/core`,`/dev/cpu`,`/dev/stdin`等。曾经我还天真的以为这些真的是文件，甚至还在思考Linux怎么用这么原始的方式来记录内核、CPU这些信息。现在想来还是too young too simple.  
不过`/dev`目录下的*文件*与*VFS*还是有不同的。通过`/dev`的名称**dev->device**就可以发现，这其中的文件实际上应该与某个设备对应。所以`/dev`中的*文件*更加趋向于是一些驱动。关于如何写一个驱动，可以看这篇文章。

>https://olegkutkov.me/2018/03/14/simple-linux-character-device-driver/

驱动的实现与VFS的实现有相同的地方，也可以看做是一个单一的虚拟文件。  
那么什么是**虚拟文件系统**呢？从名字就可以看出，它的重点在**虚拟**上。实际上，Linux允许开发者通过加载内核模块的方式来安装一个**不需要实际硬件**的文件系统,所有对这个文件系统的系统调用最终会调用到内核模块中开发者定义的函数里。因此这就成为了一个**虚拟**的文件系统。

# 内核模块
那么实现一个虚拟文件系统，首先要实现一个内核模块。我们先从一段简单的代码看起
>main.c
```C
#include <linux/init.h>
#include <linux/module.h>
#include <linux/kernel.h>
static int __init my_init(){
    printk("My module init");
    return 0;
} 

static void __exit my_exit(){
    printk("My module exit");
}

module_init(my_init);
module_exit(my_exit);
```
>Makefile
```Makefile
BINARY     := my
KERNEL      := /lib/modules/$(shell uname -r)/build
ARCH        := x86
C_FLAGS     := -Wall
KMOD_DIR    := $(shell pwd)
TARGET_PATH := /lib/modules/$(shell uname -r)/kernel/drivers/char

OBJECTS := main.o

ccflags-y += $(C_FLAGS)

obj-m += $(BINARY).o

$(BINARY)-y := $(OBJECTS)

$(BINARY).ko:
	    make -C $(KERNEL) M=$(KMOD_DIR) modules
install:
	    cp $(BINARY).ko $(TARGET_PATH)
	        depmod -a
uninstall:
	    rm $(TARGET_PATH)/$(BINARY).ko
	        depmod -a
clean:
	    make -C $(KERNEL) M=$(KMOD_DIR) clean
```
在su下执行命令就可以安装我们的模块了
> make clean && make && insmod my.ko

安装后可以到`/var/log/kern.log`查看打印的日志。模块的代码很短，可以发现比较关键的部分是`module_init`和`module_exit`。这是两个宏，更多的信息可以访问该地址

> https://www.cs.bham.ac.uk/~exr/teaching/lectures/opsys/13_14/docs/kernelAPI/r14.html

# 注册文件系统

