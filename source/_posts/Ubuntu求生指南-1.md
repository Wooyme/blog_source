---
title: Ubuntu求生指南(1)
date: 2018-07-16 00:42:21
tags:
  - Ubuntu
  - 杂谈
---

## **1.前言**
先抱怨两句，Linux对于各种A卡的支持实在是太烂了。Radeon的驱动下，A卡的跑分还不到Intel的一半。换用AMD自家的amdgpu-pro直接就进不了桌面系统了。搞了好几天还是搞不定，最后还是决定算了，毕竟核显跑Minecraft还是有40-60帧。不用独显说不定还能省点电。  
如果你不幸也是A卡用户，这篇文章了应该能给你点帮助。如果你N卡用户，这篇文章也还是有点用的，说不定哪天就能救人一命。
如果你想看看自己的显卡/各种设备的情况的话，用`lspci`这个命令,参数`-k`的使用率比较高。
## **2.AMDGPU-PRO**
>Ubuntu 18.04对应下载地址  
https://support.amd.com/en-us/kb-articles/Pages/Radeon-Software-for-Linux-18.20-Early-Preview-Release-Notes.aspx

>Ubuntu 16.04对应下载地址  
https://support.amd.com/en-us/kb-articles/Pages/AMDGPU-PRO-Driver-for-Linux-Release-Notes.aspx

相信看看链接大家也就懂了。18.04还是EPR，能不能用就看造化了。16.04应该能用，但是我不确定，因为我是ubuntu gnome16.04。`稍微科普一下，ubuntu17.10前的桌面系统是Unity，然后因为大家喜好有差，就出现了Ubuntu Gnome和KUbuntu(Ubuntu Kde)这两个发行版`然后amdgpu-pro在我的电脑上似乎不太行，如果你是官方的Ubuntu发行版，说不定可以试试。
安装完之后不要忘了修改grub
>        Edit /etc/default/grub as root and modify GRUB_CMDLINE_LINUX_DEFAULT in order to add "amdgpu.vm_fragment_size=9" (without the quotes). The line may look something like this after the change:
        GRUB_CMDLINE_LINUX_DEFAULT="quiet splash amdgpu.vm_fragment_size=9"
        Update grub and reboot as root:
        update-grub;reboot

官方丧心病狂的把这段话写在网页的后半部分，可能很多人都没有注意。

接下来就是非常关键的部分了。如果你安装完之后成功的进入了桌面系统，你可能会在gnome的 *设置->详细信息->图形* 里看到LLVM的字样(顺便说一句，看到LLVM的时候我就总觉得这玩意不太靠谱)，当然根据显卡型号不同，你可能会看到不一样的结果。只要你没看到Intel之类的东西，你多半是成功了。  

对于成功者来说，这部分就到此结束了。但是如果你安装了amdgpu-pro之后，出现开机只能进bash或者登录界面循环的情况，你可能就不得不浪费一些时间了。**Ubuntu Ask** 论坛上有许多关于这种问题的帖子，里面说不定有些解决方案。而这里，我只给一种最简单的方案。  
在amdgpu-pro安装之后，系统会多一个`amdgpu-pro-uninstall`的命令。我们要在boot的时候选择`recovery mode`。进入recovery mode之后，可以先选择一下clean，这样他会自动挂载分区。clean结束之后，就进入root然后运行`amdgpu-pro-uninstall`就可以了，同时不要忘了把grub改回来。

## **2.某不靠谱PPA**
可能在尝试amdgpu-pro之前，你还试过很多操作。毕竟那些年代久远的文章总是会介绍一些奇奇怪怪的方法。。。然后，我就遭众了。  
这里的方法不只是对安装驱动的时候遇到的问题有效，所有因为安装了某ppa的应用而导致问题的情况都可以用这个方法解决。  
*PPA回滚*
>sudo apt-get install ppa-purge  
sudo ppa-purge ppa:你要删除的ppa

跟上面一样，进了`recovery mode`之后就可以运行这两行命令，把刚刚安装的程序删掉就可以了。

## **3.我并不知道我刚刚装了什么东西**
这是最难受的。但是只要我们安装的东西全都是通过apt安装的，那么这个问题就还可以解决。首先apt的安装都是有日志的，我的位置在`/var/log/apt/history.log`,我们只要找到最近一段时间里安装的包并且把他们删掉就行了。
我们可以在`recovery mode`里用vim打开这个log，找到文件尾处的安装信息。有时候会出现一次安装的包太多的情况，这个时候就需要用一些命令处理一下。
>这里给一个范例  
`grep -A 3 'Start-Date: 2018-07-16  22:56:10' /var/log/apt/history.log |sudo tail -1 > tmp.log`  
再用Vim删除一些不用的信息  
`tr ' ' '\n' <tmp.log | sed '/,$/d' > tmp0.log`  
再删除最后一行

最终就可以得到一个完整的安装列表。
写一个脚本
```Shell
# Run as root

# Store packages name in $p
p="$(</tmp/final.packages.txt)"

# Nuke it
apt-get --purge remove $p


#clears out the local repository of retrieved package files
apt-get clean


# Just in case ...
apt-get autoremove
```
运行一下，大功告成。
