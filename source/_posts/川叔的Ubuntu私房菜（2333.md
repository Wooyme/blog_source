---
title: 川叔的Ubuntu私房菜（2333
date: 2019-02-12 15:04:45
tags:
    - Ubuntu
    - 杂谈
---

# 前言
**本篇针对Ubuntu18.04或Ubuntu Gnome**
推荐一本书《鸟哥的Linux私房菜》，虽然我没有看完，但真的是一本很棒的书。由于鸟哥的Linux写的非常深入，所以对于不做运维的人来说有些太难了。  
这篇《川叔的Ubuntu私房菜》就作为我记录各种在使用Ubuntu时学到的小技巧的地方。**顺便一提利根川赛高**。

# crontab
linux的开机自启动方式很多，去搜索的话一般的建议都是添加到服务里。但是有时候我们就一个简单的脚本添加到服务也太麻烦了。所以我找到了一种我认为最简单的开机启动方法。
```bash
crontab -e
```
如果你的系统里有多种编辑器的话，会进入一个编辑器选择的步骤  
然后在打开的文本里添加
```
@reboot your_command
```
这样下次启动的时候就会执行这个命令，路径什么的需要注意一下。  
除了`@reboot`之外，还有个服务器上非常常用的`@daily`，即每天自动执行，当然`crontab`还有很多其他的定时任务。有兴趣的可以看这个
> http://man7.org/linux/man-pages/man5/crontab.5.html

# SCP (不是基金会)
> Linux之间传文件有除了ftp外更简单的方式

不知道为什么，每次在服务器上装vsftpd都是一件非常痛苦的事情，总会遇到各种奇奇怪怪的问题。以至于后来有一段时间，我甚至选择使用网盘、github这种第三方存储作为上传文件到服务器的媒介。  
这种情况终止于一天逛**Ubuntu Forums**的时候发现有人用
```bash
scp xxx root@xxx:/home/xxx
```
的命令。查了一下才知道，这就是基于ssh的文件上传。
```
usage: scp [-346BCpqrv] [-c cipher] [-F ssh_config] [-i identity_file]
           [-l limit] [-o ssh_option] [-P port] [-S program]
           [[user@]host1:]file1 ... [[user@]host2:]file2

```
example
```bash
scp this.zip root@my.server.com:/home/www/this.zip
```
要注意的是目录需要提前建好，不然会报*没有找到目录或文件*的错。

# kill -9 PID & xkill
有时会遇到执行了`kill PID`但是进程还是杀不死的情况。这个时候加上`-9`参数就可以杀死进程了。
另外在桌面端的时候有时候会遇到Chrome、QQ、LibreOffice卡死的情况，这个时候可以`Ctrl+Alt+T`打开终端输入`xkill`,然后鼠标就会变成一个"x"，点一下卡死的窗口就可以关掉了。顺便一提的是有时候Ubuntu会弹出系统崩溃、系统错误这样的对话框，实际上其中只有很小一部分是真正的系统崩溃，大部分都是某个软件崩溃了，具体内容可以展开详情看看。

# QQ & TIM
由于腾讯压根就没有提供Linux的版本所以在Ubuntu上用QQ是一个比较麻烦的事情。Github上有很多QQ for Linux的项目，但是其实最终都指向了一个唯一的解决方案——`Wine`。  
> 说个冷知识，`Wine`的全称是`Wine Is Not an Emulator`

Wine给在Linux、Mac上运行Windows程序提供了可能。虽然各种奇怪的bug和内存泄露是经常的事，但是有总比没有好。不过呢如果真的要从安装Wine、安装各种.Net库、字体开始到最终安装QQ、TIM，那也太麻烦了。比较简单的是去Github上找已经打成`AppImage`的发行版，但是还有一种更通用、更稳定的方法——`Crossover`。

`Crossover`是一款商业软件、所以是收费的，终身是100多，不想付的话网上也应该能找到破解版。
>官网 http://www.crossoverchina.com/goumai.html?lid=1984

装了`Crossover`之后安装Windows上的程序就完全傻瓜化了，而且会有各种软件在Linux上兼容性的评分。QQ和TIM间建议安装TIM，QQ有时候会突然崩溃，比较僵硬。

# 系统监视器(任务管理器)
Ubuntu是有个类似于任务管理器的东西的，只不过Gnome并没有为他绑定快捷键，如果想要像Windows一样使用任务管理器的话需要在`设置`->`设备`->`键盘`里添加。系统监视器对应的命令是`gnome-system-monitor`

# 美化桌面
虽然18.04换了Gnome，但是完完全全的继承了原本Unity的丑。看看隔壁`Kali`的桌面，帅到爆。再看看Ubuntu，开机基佬紫就不说了，桌面侧边栏真是丑到没话说。
首先要安装`User Themes`，
> https://extensions.gnome.org/extension/19/user-themes/

为了更方便的安装插件、需要在Chrome中先安装`extensions.gnome.org`的插件。

安装好User Themes后(最好再装一个`dash to dock`或者`dash to panel`)就可以到gnome-look里挑一个中意的主题了。
> https://www.gnome-look.org

主题的作者一般会在下载的地方写好安装方法。

# 美化登录
如果说桌面还可以接受的话、那登录界面真是人神共愤了。不过好在gnome-look里一样提供了很多gdm的主题。我个人比较推荐这个
> https://www.gnome-look.org/p/1241489/

作者的README写的也非常清楚

>复制bg-boat.jpg到背景目录
>cp ~/Downloads/Ocean-blue-GDM3/bg-boat.jpg /usr/share/backgrounds/
>
>备份默认的ubuntu.css
>cp /usr/share/gnome-shell/theme/ubuntu.css /usr/share/gnome-shell/theme/ubuntu.bk
>
>覆盖ubuntu.css
>cp ~/Downloads/Ocean-blue-GDM3/ubuntu.css /usr/share/gnome-shell/theme/
>
>重启
>reboot -f
