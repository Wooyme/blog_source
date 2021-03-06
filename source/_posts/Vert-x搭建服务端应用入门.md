---
title: Vert.x后端漫游指南（1）
date: 2018-07-01 22:40:34
tags:
  - Vert.x
---

## **1.为什么是Vert.x**
如果你从来没有听说过Vert.x，不要觉得是自己孤陋寡闻或是被时代抛弃了。Vert.x是一个非常小众的服务端框架，我也举不出某个大厂商将Vert.x作为它们后端技术栈的例子，但是这并不能说明Vert.x不够优秀，Vert.x不火很大程度上是因为它没有那些惊世骇俗的噱头，同时它实在是太年轻了。  
学习这样一个小众的框架往往是要付出许多代价的，无论是大段的报错信息还是连报错信息都没有的错误，小众的东西总是让我们费劲心思。好在Vert.x并不属于这一类的小众，Vert.x的文档非常详细(包括中文文档)，并且它的源码注释也非常符合规范，配合文档以及注释，想要学习这个框架还是非常容易的。  
那么问题来了，在现在这个框架爆炸的时代，我们为什么还要学习Vert.x呢？是啊，现在的框架是真的多，好像不管是什么样的语言都可以来写后端服务了，js有express，python有django，go、java、ruby就更不用说了。那Vert.x到底有什么优势呢？  
#### 优势:

* _运行在JVM上_  
也许你不认为运行在JVM上会是一个框架的优势，毕竟JVM上已经有许多非常成熟的解决方案了，大名鼎鼎的Spring，J2EE，这样一个不出名的框架要怎么与这些老牌强者抗衡，但是Vert.x就是可以，因为它与众不同。运行在JVM上让它可以很轻松的利用Java庞大的生态，各种工具，各种成熟的技术还有现在让许多框架头疼的多线程问题。
* _事件循环以及异步_  
如果你了解Netty的话，你对事件循环和异步一定不会陌生，并且我可以告诉你Vert.x的底层实现就是Netty。如果你从来没有听说过Netty那也没关系，我想你多半听说过nodejs，通俗的来说Vert.x就是运行在JVM上的Nodejs。他们都有一个Eventloop，他们的所有API都是异步的，他们都遵守"Don't call us,we'll call you"的原则。只不过Vert.x更加自由，更加安全。异步很大的一个好处是将逻辑与I/O分离，逻辑的部分干逻辑的事，I/O的部分干I/O的事，因为I/O往往是耗时严重的，所以等到I/O完成了自己的工作后再来召唤逻辑，而在I/O工作的时候逻辑就去做其他的事情，这样两方都不会有干等着的时候。
* _多语言支持_  
 JVM的优势在这种时候体现的淋漓尽致，除了JVM以外，还有哪个平台能够做到同时运行多种语言呢？.Net吗。Vert.x支持7种语言，Java(个人认为，Java是最不适合的。至少目前是这样，Java10应该会有所改观), JavaScript, Groovy, Ruby, Ceylon, Scala and Kotlin，并且这种支持不是简单的因为大家都能运行在JVM上。Vert.x为这些语言提供了属于他们自己的API。
* _不仅仅是Web,不仅仅是服务_  
 没错，不仅仅是Web，Vert.x提供的不仅仅是对Web的支持。Vert.x是个插件化的框架，它的核心部分实际上非常的简练。这个核心提供了TCP，UDP甚至是DNS的服务端，也提供了简单的HTTP服务端。在这个核心的基础上，我们可以添加其他的插件，使这个框架能够胜任Web服务端的工作。如果我们碰巧需要一个客户端的话 —— 想象我们现在要开发一个代理工作，我们就同时需要一个服务端和一个客户端 —— 那Vert.x简直就是不二之选，DNSClient,HttpClient,NetClient,DatagramSocketClient，我们想要的它都能提供。

那么同样的，任何事物都有好的与坏的，如果有某样东西完美了，那它为什么还会有竞争对手
#### 劣势:
* _上手困难_  
 不是所有人都能轻松的理解异步的，特别是在java里有很多同步代码的情况下，比如JDBC，习惯了同步的人很容易一不小心就忘了自己在一个异步的世界里，然后阻塞了Eventloop,接着有幸看到Vert.x抛出的超时异常。没错，Vert.x是会检测运行时间的，如果某段代码超时了，你会看到一个非常醒目的异常。
* 没了  


## **2.跨出的一小步**
说了这么多，不如来看一看怎么写一个简单的http服务端吧。这个入门应该会出一个系列，从第一个服务端程序到一个比较完整的后端。可能会涉及到一点后端渲染，但是其实我不太喜欢后端渲染的东西。前端渲染有很多好处，无论是CDN还是利用浏览器的缓存，前端渲染都能减轻服务器负担。  
首先，我们需要Vert.x。这个可以在Vert.x的官方网站上获取到
>https://vertx.io/download/

官网甚至提供了一个构建Vert.x工程的工具
>http://start.vertx.io/

如果你不知道怎么配置的话，可以直接在这里开始。  
在这个系列里，我可能不常使用Java，而更加倾向于使用Kotlin。因为Kotlin和Vert.x的契合度非常高，Kotlin提供的协程可以很好的解决一些因为异步导致的问题，并且Kotlin完全兼容Java的特点也让Kotlin在许多其他语言中脱颖而出。
那让我们来看一个最简单的Http服务端是怎么实现的。
```Kotlin
import io.vertx.core.Vertx
fun main(args:Array<String>){
  val vertx = Vertx.vertx()
  vertx.createHttpServer().requestHandler{
    it.response().end("Welcome to our first Http Server!")
  }.listen(80){
    if(it.failed())
      it.cause.printStackTrace()
    else
      println("Listening on 80")
  }
}
```
这是用Kotlin实现一个简单HttpServer的版本。如果你从来没有接触过Kotlin的话，你只要记住一点func{}等于func({}),函数的最后一个lambda参数是可以放到小括号外面去的。然后对于只有一个参数的lambda，这个参数默认的名称叫做it。  
我们可以看看这段代码到底做了什么。首先，我们得到了一个vertx对象，这个vertx是一切的起源。接着我们调用了createHttpServer方法，创建了一个Server。在创建完Server之后我们为它添加了一个handler，可以看到，Vert.x的API大多是流式的，写起来非常漂亮，fp的美感。在Kotlin里，这个handler表现为一个lambda，实际上这个requestHandler所需要的参数是一个Handler<HttpServerRequest>接口，所以在Java里，写法就变成了
```Java
vertx.createHttpServer().requestHandler(new Handler<HttpServerRequest>(){
  @Override
  void handle(HttpServerRequest req){
    req.response().end("Welcome to our first Http Server!");
  }
});
```
当然这种写法比较过时，java8已经加入了lambda，但是为了更加清晰一点，这里我把这种原始的版本表示出来。  
这个handler是在服务端启动后，每当有用户访问我们的服务端，服务端就会返回一个欢迎词，就像nginx的默认页面一样。同样的，listen也是差不多的过程，第一个参数是监听的端口，第二个参数则是用来处理监听结果的handler，这个handler在端口成功打开或是打开失败的时候被调用，所以我们可以知道我们的服务端有没有成功运行。  
其实这里异步的特点就已经有所体现了，我们没有等待用户访问，也没有等待端口打开，所有的一切都是异步的。当有结果的时候，Vert.x就会来调用我们设定的handler，而当没有结果的时候，Evnetloop就会在那里自己循环，只要我们不写阻塞代码，Eventloop就永远不会停止循环。

## *小结*

第一篇就大概介绍一下Vert.x的理念，实际上Vert.x的许多特性比介绍的要复杂的多，有兴趣的人可以去看中问文档，里面有讲关于Verticle的，关于分布式的，以及一些类似于热部署的。这里写的一个例子也非常的简单，但是如果你从来没有接触过异步，这对你来说可能是个挑战。你得明白，为什么这段代码不是按照顺序执行的，以及什么是lambda(这个不是非常重要，lambda只是一种表现形式而已，我们实例化一个接口也完全可以实现同样的功能)。  
下一篇应该是会讲请求的处理，get的和post的，以及Router这个扩展。
