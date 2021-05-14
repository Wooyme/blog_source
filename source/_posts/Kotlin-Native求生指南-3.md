---
title: Kotlin Native求生指南(3)
date: 2018-12-22 17:09:55
tags:
    - Kotlin
    - Native
    - C
    - 多线程
---

# 上一篇地址
> https://wooyme.github.io/2018/12/22/Kotlin-Native%E6%B1%82%E7%94%9F%E6%8C%87%E5%8D%97-2/

# 前言
上一篇意外的花了整篇的篇幅写*cinterop*,INTEROP的文档比我想的要长太多。那么这一篇就来写一下Kotlin Native的多线程模型。由于Kotlin Native还在频繁更新中，所以多线程API和注解还是有可能出现较大变化的，本篇文章针对的是**Kotlin Native v0.9**,其他版本如有差异，请忽略这篇文章，以官方为准。
> https://github.com/JetBrains/kotlin-native/blob/master/CONCURRENCY.md

并且这篇文章只讲Kotlin Native的多线程模型，并不涉及协程，想要了解协程的朋友，还是看官方的文档吧。

# Worker
可能说到多线程，我们马上就会想到`pthread`之类的东西。没错`pthread`作为**POSIX**提供的跨平台多线程API可以说是非常深入人心了。你问我Kotlin Native支不支持`pthread`,那肯定是支持的，我们完全可以像在C/C++中做的那样，通过`pthread`创建新的线程，只是要在线程启动的时候运行一下`kotlin.native.initRuntimeIfNeeded()`。  
但是，实际上Kotlin Native给我们提供了一个更好的选择——**Worker**。让我们来看看官方对Worker的介绍
> 不同于线程的是，Kotlin Native引入了Worker这个概念:能够并行处理请求队列的控制流(concurrently executed control flow streams with an associated request queue)。Workers与Actor模型中的Actor很像，一个Worker可以与其他Worker交换数据。**在任何时候可变的对象只会被一个Worker拥有**。

好像不是很好理解，我们看官方给的例子
```Kotlin
package sample.workers

import kotlin.native.concurrent.*

data class WorkerArgument(val intParam: Int, val stringParam: String)
data class WorkerResult(val intResult: Int, val stringResult: String)

fun main() {
    val COUNT = 5
    //创建5个Worker对象
    val workers = Array(COUNT, { _ -> Worker.start() })

    for (attempt in 1..3) {
        val futures = Array(workers.size) { workerIndex ->
            workers[workerIndex].execute(TransferMode.SAFE, {
                //传入要给Worker处理的参数
                WorkerArgument(workerIndex, "attempt $attempt")
            }) { input ->
                //Worker处理参数
                var sum = 0
                for (i in 0..input.intParam * 1000) {
                    sum += i
                }
                WorkerResult(sum, input.stringParam + " result")
            }
        }
        val futureSet = futures.toSet()
        var consumed = 0
        while (consumed < futureSet.size) {
            //等待运行结束的Workers
            val ready = waitForMultipleFutures(futureSet, 10000)
            ready.forEach {
                //处理Worker的运行结果
                it.consume { result ->
                    if (result.stringResult != "attempt $attempt result") throw Error("Unexpected $result")
                    consumed++
                }
            }
        }
    }
    workers.forEach {
        //终止Worker
        it.requestTermination().result
    }
    println("Workers: OK")
}
```
这就是一个通过创建多个Workers来并行处理数据的例子。可以看到跟Actor模型还挺像的。在`Worker.execute`的第一个参数里传入生成待处理数据的lambda，然后在第二个参数里传入处理数据的lambda。然后在需要的时候调用`future.consume`,传入处理结果的lambda。这一切看上去跟Java上的许多异步框架别无二致，但实际上。。。。后面会讲到其中的坑爹之处。  

# 全局变量和单例
> 虽然Worker和线程的实现方式并不相同，但是行为类似，所以后面就不严格区分Worker和线程了

在线程间共享数据最简单的方法就是全局变量了，但同时全局变量也是导致多线程出现各种问题的罪魁祸首。于是Kotlin Native引入了一系列的限制措施来保证全局变量不会影响Worker模型的工作。

- 除非添加了特殊的注解，不然全局变量只能在主线程中被访问，如果其他线程试图访问这个变量会报`IncorrectDereferenceException`异常
- 一个添加了`@ThreadLocal`的全局变量会在每个线程里复制一份，所以每个线程都是独享这个变量的，**某个线程对变量的改动对其他线程不可见**
- 一个添加了@SharedImmutable的变量是能在线程间共享的，但是**它会被冻结**，后续如果程序尝试修改这个变量也是会抛出异常的。需要注意的是，这个冻结不受`var`或是`val`的影响，就算声明的时候是`var`的只要冻结了就不能再被修改了。同时需要注意的是，**一个被冻结的变量是不能被解冻的**
- 对于单例(objects)，**除非添加@ThreadLocal注解,不然会被冻结从而可以在线程间共享**。对于其中的属性，lazy是被允许的。
  
相信对于从JVM转过来的人来说，我们还是习惯写单例而不是全局变量。就算是保存一个全局变量，也还是习惯于放在一个单例里作为单例的属性。于是坑就出现了。
```Kotlin
object Foo{
    lateinit var A:String
    init{
        //某些操作....
        A = ......
    }

    fun foo(){
        //某些操作.....
        A = .....
    }
}
```
根据上面的规则，猜猜看结果会是什么样吧。我想大部分人都会认为`init`中对A的赋值应该是可以的，至于`foo`中的赋值则要看是在哪个线程里执行了。但事实是，两个都不行，无论什么情况都不行。即使A是`lateinit`的，只要Foo被`frozen`了，里面的属性就不能再被修改。Kotlin Native是真的很严格，就算我们自始至终只有一个主线程也不行。所以`object`几乎是必须加上`@ThreadLocal`注解。  
但是加上`@ThreadLocal`真的就能解决问题了吗，规则里说，加上`@ThreadLocal`的变量会在每个线程间复制一份，不知道你们是怎么理解的，反正我最开始看到这端话的时候觉得这个*复制*应该会在*新的线程创建的时候把创建这个线程的线程中的变量复制到新的线程上*。举个例子
```Kotlin
@ThreadLocal
object Foo{
    var A:String? = null
}
fun main(){
    Foo.A = "Hello"
    Worker.start().execute(Transport.SAFE,{}){
        println(Foo.A)
    }.consume{}
}
//我期望的输出
Hello
//实际上的输出
null
```
> 这谁顶得住啊

总之就是被Kotlin Native的文档秀了一脸。它说的复制，就是保证其他线程上也有一个叫做`Foo`的单例。而单例里面是什么则完全取决于它初始化完了是什么。这简直是颠覆了我对多线程的认识。那么正确的做法是什么呢。
```Kotlin
@ThreadLocal
object Foo{
    var A:String? = null
}
fun main(){
    Foo.A = "Hello"
    Worker.start().execute(Transport.SAFE,{
        Foo.A
    }){ str->
        Foo.A = str
        println(Foo.A)
    }.consume{}
}
```
`execute`的第一个lambada参数就给我们用来干这个事情的，这是唯一一个能让其他线程与主线程产生联系的地方。我们需要在这里把想要*复制*的值传到子线程，然后在子线程里做*粘贴*的工作。  

# 共享可变量
但是这样的*复制*很明显是不够的，比如说我想在Worker里运行一个Event Loop，我必然要在Worker运行的时候向它传递一些数据。于是这个时候就必须要跳出Kotlin Native的管理，寻求C的帮助了。  
虽然Kotlin的变量在线程间是独立的，但是通过`nativeHeap.alloc`分配的内存在线程间依然是共享的。所以我们就可以写出这样的代码
```Kotlin
fun main(){
    val ptr = nativeHeap.allocArray<ByteVar>(10)
    memScope{
        memcpy(ptr,"A".cstr.ptr,"A".cstr.size)
    }
    val future = Worker.start().execute(Transport.SAFE,{
        ptr
    }){ _ptr->
        while(true){
            println(_ptr.toKString())
            sleep(100)
        }
    }
    sleep(100)
    memScope{
        memcpy(ptr,"B".cstr,ptr,"B".cstr.size)
    }
    future.consume{}
}
```
通过这种方法就可以在线程间共享可变的变量了。当然这和传统的多线程模型一样，如果设计不当依然会导致各种多线程中常见的问题。  
这样的共享方式是比较原始的，毕竟我们共享的是最原始的数据，如果想要共享一个对象的话，就需要使用`Object subgraph detachment`
> 一个没有外部引用的对象子图可以通过`DetachedObjectGraph<T>`解除连接(disconnected)变成一个`COpaquePointer`从而存储在C的结构体中，接着可以在任意的线程或Worker中通过`DetachedObjectGraph<T>.attach()`重新连接得到这个对象子图。

配合前面的方法，就可以在线程间共享对象了。

# 小结
Kotlin Native的多线程模型就如Kotlin的空安全机制一样，可以说是为了解决传统多线程中的问题做出了许多设计上的规范，但是为了应对一些特殊的情况，这种规范也是要做出让步的，这种时候还是得我们自己来注意多线程间的问题。  
那么至此，Kotlin Native的系列就算是结束了。基本上把我在做KN开发时候遇到的一些坑都写在里面了，要说体验如何的话，一是资料太少了，官方的又只有英文，有些是它没说清楚，有些是我理解有差，导致了许多莫名其妙的问题。二是库太少了，整个github上，关于Kotlin Native的库只有两个，一个是只有IOS版本的Ktor，另一个是libui的Kotlin Native Binding，而且好久没有更新了。三是太依赖C语言了，没点C/C++开发基础还真搞不定。  
总之Kotlin Native作为一个连1.0版本都没到的语言(姑且叫它语言吧)，能用它写出一个还算像样的工具已经是挺不错的。相信只要JetBrains没有放弃KN，KN应该也会成为一门像Go一样大众的语言。