---
title: Kotlin Native求生指南(1)
date: 2018-12-22 00:13:19
tags:
    - Kotlin
    - Native
    - C
---

# 项目地址
> https://github.com/Wooyme/Wsocks-Naitve-Client

# 前言
> C++是一门好语言，Go也是一门好语言，Kotlin Native不是。

首先确定一个观点，写Kotlin-JVM不一定要很懂Java，比如我自己。但是写Kotlin-Native要是不懂C，那就等着吃屎吧。Kotlin Native不是一个让我们跳过C语言走上Native开发道路的神器，至少现在不是。如果你想要Native又不想学C有关的东西的话，选Go吧，我能在这里列举1000个Go的优点和0个缺点。如果一定要说Go有什么缺点的话，那就只能是程序崩溃的时候Go打印的栈信息不能像Java那样漂亮。  
当然Kotlin Native也不是一无是处，至少它可以督促我再复习一遍C语言的知识。 

# 开发环境
现在IDEA和CLION都可以支持Kotlin Native的开发。这个项目我用的是IDEA和Gradle，在开发这个项目之前我也用过CLION和Cmake的组合，要做个比较的话，还是Gradle更加适合我们这些从Kotlin-JVM转过来的玩家，毕竟对于不熟悉Cmake的人来说，看看别人写的CmakeList就已经够头疼的了，更何况要自己写。  
顺便一提，我是在Ubuntu18.04上开发的。Windows和Linux上开发可能会有比较大的差异，还是推荐Windows上装一下Cygwin，能一定程度上减小这个差异。

# 第一个工(小)程(坑)
从IDEA里创建一个工程是很容易的。不过我遇到了一个小坑，IDEA创完工程之后会出现kt文件打不开的情况，应对的办法是`File->Invalidate Cache/Restart...`  
从这第一个坑开始，我们就踏上了Kotlin Native的漫漫坑爹路。工程创建之后会有一个默认的文件，里面是个hello world，我们编译运行一下。编译这个过程只能从Gradle里执行，在`other`里`runProgram`，顺利的话Hello World就就会成功打印出来。这里应该不会遇到什么问题，Gradle会智能的下载KN的编译器，只是可能会有点慢。

## 我能干什么
爬过第一个坑之后，你可能迫不及待的想写点什么，比如来一个文件IO，你可能会习惯性的写一个`File`然后等待IDEA的语法提示，但是令人失望的是，IDEA除了把你的`File`标记为红色以外什么都不会做。然后你也许还会试一试其他你在JVM里经常用的类，但是它们多半没有，除了`StringBuilder`这个SB，它还是依然坚挺。  
这个时候你可能才意识到，大清亡了，世界变了，原来的那些小伙伴都不在了。剩下的只有少的可怜的*Kotlin标准库*和陌生的*stdlib*、*posix*、*win32*或者*linux*、*darwin*以及我至今不能理解为什么要放在默认支持库里的*zlib*(我觉得唯一的可能就是JetBrains的人想试试cinterop好不好用)。

## 认清现实
好了，既然之前的都用不了了，那就不得不重新开始。让我们来看一看Kotlin Native的文件操作应该是什么样的。
```Kotlin
//读取一个完整的文件，保存到String中
fun main(){
    val fp = fopen(myHome+"save.json", "r") ?: return println("Cannot open")
    val fileStat = nativeHeap.alloc<stat>()
    stat(myHome+"save.json",fileStat.ptr)
    val size = fileStat.st_size.toInt()+1
    val bytes = nativeHeap.allocArray<ByteVar>(size)
    fread(bytes,size.toULong(),size.toULong(),fp)
    val text = bytes.toKString()
    nativeHeap.free(fileStat.ptr)
    fclose(fp)
}
```
OK,先说明一下，官方的例程里很少使用nativeHeap，它们比较喜欢用memScope{}，这两个的区别我会在之后说明。现在先让我们看完这段代码。相信如果你还记得C语言的东西的话，你肯定会说:"这tm不就是C吗"。没错，这tm就是Kotlin版C语言。Old fashion的`fopen`，`stat`，`fread`，当然这一切都无可厚非，毕竟Native的世界和Java的世界本来就大相径庭。我们也可以自己动手封装一个File类出来，我也相信Jetbrains会在某一个版本把这些基本的工具加入到Kotlin Native的标准库中去的。  
除开C的部分，还是有些东西值得我们关注的，比如刚刚说的`nativeHeap`，以及一个有趣的方法——`toKString()`。这个方法真的可以说是Kotlin Native最后的仁慈了。我们知道Kotlin的String和C的const char* 是两个完全不同的体系。String类里有记录String长度的部分，好让我们知道String在什么位置结束，但是const char* 不同，它依赖结尾处的`0x00`来判断字符串是否结束。于是这里Kotlin Native很**贴心**的为我们加入了`toKString()`和`String.cstr`来保证两者间的转换。

# nativeHeap和memScope？
> 参考资料 https://resources.jetbrains.com/storage/products/kotlinconf2018/slides/5_Kotlin-Native%20concurrency%20and%20memory%20model%20(1).pdf

有一点是我们必须要知道的，那就是Kotlin Native是有GC的。在Kotlin自己的世界里，GC是隐藏在代码之下的，就像java一样，它们用引用计数法(*Simple local reference-counter based algorithm*)、"试图删除法"(*Cycle collector based on the trial deletion*)等算法保证GC的工作。但是由于Kotlin Native提供的东西实在是太少了，我们不得不依赖很多*C library*,于是手动分配内存就成了不可避免的事情。  
手动内存管理是场噩梦，这个道理让最顽固的C++也被迫妥协，从析构到现在的智能指针，C++可以说是做了许多让步了。毫无疑问，Kotlin也是懂这个道理的。于是JetBrains推出了memScope这个东西。

## memScope
memScope的作用是当memScope的作用域结束的时候，自动释放在里面分配的所有内存，以刚才的例子来说
```Kotlin
val bytes = nativeHeap.allocArray<ByteVar>(size)
```
应该写成
```Kotlin
val text = memScope{
    val bytes = nativeHeap.allocArray<ByteVar>(size)
    fread(bytes,size.toULong(),size.toULong(),fp)
    bytes.toKString()
}
```
这样当memScope结束的时候，bytes就被自动释放了，而text则是通过`toKString()`方法实例化的一个String类，可以被KN自己的GC机制回收。所以这段代码不会导致任何内存泄露。除此之外还有很多方法是需要在memScope中才能执行的，比如说`CValues<T>.ptr`这是用来获取CValues的指针，关于这个稍微说一下我的看法,ptr的getter应该是*重新分配*了一块内存，然后把CValues中的值*复制*了进去，因此才需要在memScope中执行，以保证内存不会泄露，不过这又导致了一个新的坑，我在之后会提到。

## nativeHeap
那为什么我们还需要nativeHeap这样的方式来分配内存呢。还是举个例子吧。
```Kotlin
fun foo() {
    val mgr = nativeHeap.alloc<mg_mgr>().ptr
    mg_mgr_init(mgr, null)
    while (flag) {
        mg_mgr_poll(_mgr, 100)
    }
    mg_mgr_free(mgr)
}
```
mg是一个C的网络库，我们启动了一个事件循环来处理各种网络请求。可以看到，这个库自带了一个释放函数`mg_mgr_free`,被`mg_mgr_init`初始化过的内存应该由这个库本身来释放。如果我们把代码写成下面这样
```Kotlin
fun foo() {
    memScope{
        val mgr = alloc<mg_mgr>().ptr
        mg_mgr_init(mgr, null)
        while (flag) {
            mg_mgr_poll(_mgr, 100)
        }
        mg_mgr_free(mgr)
    }
}
```
那么在循环终止，程序运行完`mg_mgr_free`之后很有可能就会因为`double free`而*崩溃*。其实我也不是很理解为什么一块内存不能释放两次，或者应该说free为什么不能对同一块内存执行两次，就算已经释放了，给我返回个false也好，何必搞个崩溃呢。总之这也算是刚开始写KN时很容易遇到的一个坑。

## 内存管理导致的坑
在我的工程里，遇到了这么个情况。我有一个系统托盘的功能，里面有一些菜单元素也就是Item，这些Item都是要显示一些字，图标之类的，当然还有回调，每次菜单变化的时候都需要调用一个`update`函数，这个函数的执行过程实际上就是重新初始化一个整个菜单。  
最初我的代码是这样的
```Kotlin
val tray = nativeHeap.alloc<tray>()
fun init(){
    //设成2是因为tray的C实现需要以空为结尾
    val menus = nativeHeap.allocArray<tray_menu>(2)
    memScope{
        menus[0].text = "设置".cstr.ptr
        //staticCFunction看名字就能知道是为了提供C语言中的"函数指针"
        menus[0].cb = staticCFunction { _ ->
            //balabalabalabala
            tray_update(tray.ptr)
        }
        tray.menu = menus
        tray_init(tray.ptr)
    }
}
```
代码看上去没什么问题，启动的时候也没什么问题，但是当运行`menus[0]`的`callback`执行的时候问题就来了。update之后”设置“这两个字变成了乱码，而罪魁祸首就是memScope。memScope在作用域结束的时候释放掉了`"设置".cstr.ptr`这个指针(ptr)对应的内存，也就是说在`tray_init`之后`menus[0].text`已经是个野指针了。于是当我们执行`tray_update`的时候，这块内存会是什么样子已经不是我们能够控制的了。于是，我被迫写出了这样的代码
```Kotlin
val tray = nativeHeap.alloc<tray>()
fun init(){
    //设成2是因为tray的C实现需要以空为结尾
    val menus = nativeHeap.allocArray<tray_menu>(2)
    val text = "设置".cstr
    val textPtr = nativeHeap.allocArray<ByteVar>(text.size)
    memScope{
        //把内存复制到不会被自动释放的地方
        memcpy(textPtr,text.ptr,text.size.toULong())
    }
    menus[0].text = textPtr
    menus[0].cb = staticCFunction { _ ->
            //balabalabalabala
            tray_update(tray.ptr)
        }
        tray.menu = menus
        tray_init(tray.ptr)
}

```
这只能说是很傻逼了,可能KN提供了一些更加优雅的方式只是我不知道。但总之Kotlin Native的`memScope`在某些情况下是会与C产生冲突的，而且这种问题往往很隐蔽，这也就是为什么野指针会成为困扰C/C++这么久的问题。

# 小结
OK，去掉代码大概有3000个字了。  
第一篇就先到此为止，大概写了一下我对Kotlin Native的看法和对它内存模型的认识以及使用中遇到的几个坑。下一篇会应该会写一下Cinterop，和Kotlin Native的多线程模型。KN的多线程对于初学者来说也是个神坑，就没见过这样的线程模型，而且文档也比较含糊，不过毕竟还在频繁更新，很多东西变的太快了，文档也确实比较难写。
