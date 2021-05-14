---
title: Graalvm native-image JavaFx初体验
date: 2019-06-04 19:08:17
tags:
---

# 前言
尝试了一段时间的Go，发现一是还是舍不得Kotlin，二是Go的UI库实在有点蛋疼，所以最近还是寻找了一些重返Kotlin的解决方案。Graalvm我早有耳闻，也有过一次不愉快的尝试，但是看到它的版本号在今年脱离了RC的标志来到了19.1.1，我决定还是再给它一个机会。

# Graalvm
> 官网地址: https://www.graalvm.org/

还是稍微介绍一下*Graalvm*吧。就目前看来Graalvm算是近年来Oracle最有创造力的产品了。
> GraalVM is a universal virtual machine for running applications written in JavaScript, Python, Ruby, R, JVM-based languages like Java, Scala, Groovy, Kotlin, Clojure, and LLVM-based languages such as C and C++.
GraalVM removes the isolation between programming languages and enables interoperability in a shared runtime. It can run either standalone or in the context of OpenJDK, Node.js or Oracle Database.

> Graalvm是一款通用虚拟机，可以运行由Javascript，Python，Ruby，R和基于JVM(Java,Kotlin,Scala,Groovy)以及基于LLVM(C和C++)的语言开发的程序。GraalVM去除了语言之间的隔阂，并且允许它们在运行过程中保持互通。GraalVm既可以独立运行，也可以在OpenJDk，NodeJs或者Oracle数据库的环境中运行(这句话，不明所以)。

以及一些特性

*  Polyglot(通用性)
   > Zero overhead interoperability between programming languages allows you to write polyglot applications and select the best language for your task.
   > 不同语言间无痛交互，根据业务情况任意选择语言
* Native(原生,这是重点)
    > Native images compiled with GraalVM ahead-of-time improve the startup time and reduce the memory footprint of JVM-based applications.
    > GraalVM生成的Native image(可以认为就是可执行文件) 使用AOT技术加快启动时间并且减少基于JVM的应用的启动内存(Java系客户端的两大痛处)
*  Embeddable(嵌入式，不是单片机那个)
    >GraalVM can be embedded in both managed and native applications. There are existing integrations into OpenJDK, Node.js and Oracle Database
    > 这个Embeddable的特性我还没有接触过，所以并不理解它所说的与OpenJDK这些一起运行是个什么意思

GraalVM的这么多特性，对于我来说，最有价值的就是Native了。相信对于大部分Java系的开发者来说也是如此，“我Java系的生态要啥有啥，还要其他语言干嘛？”  
所以接下来就写一下GraalVM这个不成熟到能跟Kotlin Native 55开的Native image吧。

# Native image
> 官方文档 https://www.graalvm.org/docs/reference-manual/aot-compilation

话是说的很漂亮，解决了Java写客户端的两大痛处，但是实际用起来那可真全是痛处。当然这里说的痛处是开发中的痛处，在成功build了native image之后，确实是如官网所说，启动速度快，内存占用小(除了莫名其妙的申请了32G虚拟内存)。

首先一点是，在19.0之后，native-image不再包含在Graal的基础包中了，要安装的话，需要运行个命令
> gu install native-image
 
 Linux上这个操作可能需要sudo。文件是从gtihub上下载的，所以国内下载可能会很慢，官网上表示可以配置代理，但是我好像不太成功。总归等了挺久之后也算是下载下来了。

目前Native image还有很多的限制。按照github上的官方文档，Native image对一些Java的特性还不能支持

| 特性 | 支持情况 |
|------|----------|
| 动态类加载     |  不支持        |
|  反射    |  需要配置文件        |
|   动态代理   |    需要配置文件      |
|   JNI   |     大多数支持     |
|  Unsafe Memory Access    |      大多数支持    |
|   类初始化(static里初始化类)   |    支持      |
|   InvokeDynamic   |   不支持       |
|   Lambda Expressions   |     支持(有问题)     |
|  Synchronized, wait, and notify    |     支持     |
| Finalizers     |    不支持      |
|   References   |     大多数支持     |
|  Threads    |    支持      |
|  Identity Hash Code    |   支持       |
|  Security Manager    |   不支持       |
| JVMTI, JMX, other native VM interfaces |  不支持 |
| JCA Security Services |  支持  |

首先动态类加载肯定是不用想了，native的部分与非native的部分明显不能建立联系。而反射可以通过传入配置文件来获得支持，这就让许多框架的使用成为了可能。

# Native的客户端

如果只是一个简单的，单线程的console应用，那么native image的使用基本上是不会有任何问题。但是一旦涉及了GUI，这个native image就有点搞人心态了。 

Java上大家熟知的GUI方案无外乎Swing和Javafx，在graal的issues中可以看到现在尚且不能支持swing，而且由于swing已经是个如此年迈的框架，我也不倾向于使用它。那么现在的选择就只有javafx了。

## JavaFX

GraalVM有两个版本，一个CE，一个EE，其中CE版是不带JavaFx的。而EE中所带的JavaFx会在编译成native image的过程中出现无数的问题，于是我搜索了graal的issues，并且成功找到了一些前人的经验

> https://github.com/oracle/graal/issues/403
> 
> https://github.com/oracle/graal/issues/994

*其中issue403是Glavo大佬提出的*

**在 403 的评论中可以看到一个叫做gluonhq的产品**，
>  https://github.com/gluonhq/client-samples

应该有人通过它成功编译了javafx的native image。我也按照gluonhq的文档做了一些尝试，非常遗憾的是，我并没有成功，失败的原因也不是说出现了什么错误，而是这货占用的内存量有点夸张。几次尝试编译都因为内存不足而告终，后来我曾试图把开发环境搬到生产服务器上(一台16核32G的虚拟服务器)，但最终因为gluonhq不支持windows而宣告失败。所以如果看到这篇文章的人中有不用Windows且内存达到16G的话，可以尝试一下直接使用gluonhq client。

虽然用gluonhq失败了，但是至少说明已经有人成功了吧。于是我在994中的最后一条评论里找到了这个仓库
> https://github.com/maxum2610/HelloJFX-GraalSVM

maxum2610表示要构建naive image需要自己build一个OpenJFX。这简直是打开了新世界的大门。既然都自己构建OpenJFX了，那稍微改改代码总也不是什么大事吧。于是参考maxum2610的文档，我开始了自己的旅程。

## OpenJFX
build OpenJFX的过程中只有遇到了一个问题，这货要求gradle的版本是1.8，然后IDEA默认的是5.1，所以要自己调整一下。

整个构建的过程是很愉快的，把构建得到的`/build/sdk/rt/lib`里的文件复制到Graalvm的jre下就可以得到一个自己的JavaFx环境了。接着我开始尝试编译Native image,并且不出意外的失败了。

报的错误是issue1376中的错误
> https://github.com/oracle/graal/issues/1376
> 
> Invoke with MethodHandle argument could not be reduced to at most a single call: java.lang.invoke.MethodHandleImpl.buildVarargsArray(MethodHandle, MethodHandle, int)

解决方案是把所有报这个错的地方的lambda全部改成java6的语法。

在全部改完之后就可以成功生成naive image了。当然走到了这步还没有成功。
在运行的过程中会发现加载libglass.so失败的问题，一开始我以为是路径有问题，后来发现是在调用JNI_Onload的时候返回了-1。

于是本着既然都改了源码了，再改点C的代码又怎么样呢的想法，大概看了一下glass_general.cpp中JNI_Onload函数的内容，发现是在加载`sun/misc/GThreadHelper`的时候报错了，
```C++
   clazz = env->FindClass("sun/misc/GThreadHelper");
   if (env->ExceptionCheck()) return JNI_ERR;
   if (clazz) {
       jmethodID mid_getAndSetInitializationNeededFlag = env->GetStaticMethodID(clazz, "getAndSetInitializationNeededFlag", "()Z");
       if (env->ExceptionCheck()) return JNI_ERR;
       jmethodID mid_lock = env->GetStaticMethodID(clazz, "lock", "()V");
       if (env->ExceptionCheck()) return JNI_ERR;
       jmethodID mid_unlock = env->GetStaticMethodID(clazz, "unlock", "()V");
       if (env->ExceptionCheck()) return JNI_ERR;

       env->CallStaticVoidMethod(clazz, mid_lock);

       if (!env->CallStaticBooleanMethod(clazz, mid_getAndSetInitializationNeededFlag)) {
           init_threads();
       }

       env->CallStaticVoidMethod(clazz, mid_unlock);
   } else {
        env->ExceptionClear();
        init_threads();
   }
```
确认了一下jniconfig，发现确实没有这个库。在搜索了一圈之后决定干脆不加载了，直接走else分支。成功！

我修改的OpenJFX已经传到github上了
>  https://github.com/Wooyme/openjdk-jfx

重新build OpenJFX之后编译出的naive image就可以正常运行了。

## 小结

Graalvm依然是个很不成熟的技术，要尝试的话就得做好踩坑的觉悟。