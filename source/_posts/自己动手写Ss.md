---
title: 自己动手写Ss
date: 2018-12-16 00:28:59
tags:
    - Kotlin
    - VPN
    - Vert.x
---
# 项目地址
> https://github.com/Wooyme/Wsocks
# 前言
最近VPN不太太平，无论是商业的还是自己租服务器私建的，都或多或少有些遭众。据说是GFW进行了一波升级，但是我个人还是觉得，它只是又更新了一轮黑名单。总之不管怎么样，最近的科学上网是不太稳当了。  
一周前，我用了一年的服务器遭到了封禁。一时间大有一种大难临头的感觉，再加上各种流言蜚语，于是决定自己来做一个代理工具。

# 原理
用过SS的应该都知道SS由两个部分组成——**客户端**与**服务端**。**客户端**往往运行着一个Socks5代理，能够从浏览器之类的程序获取请求，然后*加密*发送到**服务端**，**服务端**收到请求后，*解密*再发送给真正的**目标服务器**并监听**目标服务器**的返回，得到返回后再*加密*发送给**客户端**，**客户端**再*解密*后发送给浏览器。  
> 浏览器 --(明文)--> 客户端 --(密文)--> 服务端 --(明文)--> 目标服务器  
> 目标服务器 --(明文)--> 服务端 --(密文)--> 客户端 --(明文)--> 浏览器
> 
整体流程其实非常简单，只是因为有加解密的过程，显得有些繁琐。

# 实现
要实现这样一个代理，核心是解决客户端与服务端之间的交互。实现这个交互的方式有很多种，无论是直接基于UDP，UTP，TCP这样的底层协议开发，还是使用HTTP，WebSocket这样应用层的协议都是可行的。**这里我们使用WebSoket协议作为客户端与服务端之间的交互协议**
## 1.Why websocket ?
* 第一，基于WebSocket的开发实在是太简单了。Websocket作为一个被各大浏览器，以及服务端框架支持的协议，其封装实在是太完善了。使用这些封装好的库，我们就不用考虑TCP协议中会遇到的粘包，UDP中的丢包问题。
* 第二，Websocket可以把我们的代理程序伪装成一个站点，因为这是网页与后端间常用的协议，已经被许多社交网站，视频网站，页游使用，所以显得更加正常。
* 第三，相较于HTTP这样的应用层协议，websocket占用的资源还是要更小一些的，毕竟我们自己用来跑代理的服务器往往配置不会那么高，资源能省则省。
  
## 2.How to do ?
基于Websocket的开发是非常简单的，当然前提是得有个够给劲的框架。按照我的博客的惯例，这篇文章，不出意外的会使用Vert.x和Kotlin作为技术栈(笑。顺便说一下，在GC，Cache等参数设置得当的情况下，是可以很大程度上降低JVM的内存占用量的。JVM的许多默认设置都是拿内存换CPU，所以会显得java程序很占内存。  
那么先贴一段Demo
```Kotlin
class ServerWebSocket:AbstractVerticle() {
    private val logger = LoggerFactory.getLogger(ServerWebSocket::class.java)
    private lateinit var netClient: NetClient
    private lateinit var httpServer:HttpServer
    override fun start(startFuture: Future<Void>){
        //初始化一个TCP客户端，后面要用
        netClient = vertx.createNetClient()
        //初始化HTTP服务端
        httpServer = vertx.createHttpServer()
        //设置websocket处理器,用于处理所有与websocket相关的功能
        httpServer.websocketHandler(this::socketHandler)
        httpServer.listen(port,it.completer()){
            logger.info("Proxy server listen at $port")
            startFuture.complete()
        }
    }
    private fun socketHandler(sock: ServerWebSocket){
        sock.binaryMessageHandler { buffer ->
            GlobalScope.launch(vertx.dispatcher()) {
                when (buffer.getIntLE(0)) {
                    //处理连接请求
                    Flag.CONNECT.ordinal -> clientConnectHandler(sock, ClientConnect(buffer))
                    //处理数据请求
                    Flag.RAW.ordinal -> clientRawHandler(sock, RawData(buffer))
                }
            }
        }
        //接受连接，可以在这之前做一些鉴权的工作
        sock.accept()
    }
}
```
服务端的整体结构就如Demo所示，在服务端接受了客户端的websocket握手之后，就会处理客户端发送的两种请求。下面是两种请求处理的实现
```Kotlin
//处理连接请求
private suspend fun clientConnectHandler(sock: ServerWebSocket, data:ClientConnect){
    try {
        //TCP Client尝试连接到目标服务器
        val net = netClient.connectAwait(data.port, data.host)
        //连接成功则设置handler
        net.handler { buffer->
        //把目标服务器返回的数据加密发送给客户端
            sock.writeBinaryMessage(RawData.create(data.uuid,buffer).toBuffer())
        }.closeHandler {
            localMap.remove(data.uuid)
        }
        localMap[data.uuid] = net
    }catch (e:Throwable){
      logger.warn(e.message)
      //连接失败则告诉客户端连接失败
      sock.writeBinaryMessage(Exception.create(data.uuid,e.localizedMessage).toBuffer())
      return
    }
    //告诉客户端连接成功
    sock.writeBinaryMessage(ConnectSuccess.create(data.uuid).toBuffer())
}
//处理数据请求
private fun clientRawHandler(sock: ServerWebSocket, data: RawData){
    val net = localMap[data.uuid]
    //把客户端的数据解密发送目标服务器
    net?.write(data.data)?:let{
        sock.writeBinaryMessage(Exception.create(data.uuid,"Remote socket has closed").toBuffer())
    }
}
```
其中uuid是为了保证数据在传输过程中能够找到请求发起者，不然客户端收到了返回的数据，会出现不知道是谁发起的问题。当然这个问题也可以用其他方式实现，理论上说，封装的再完善一点的话，是能够只靠闭包解决。  
以下是RawData类，展示数据加密过程,加密方式是`AES/CBC/PKCS5Padding`，javax库中提供的加密方式
```Kotlin
class RawData(private val buffer:Buffer) {
    private val decryptedBuffer = Buffer.buffer(Aes.decrypt(buffer.getBytes(Int.SIZE_BYTES,buffer.length())))
    private val uuidLength = decryptedBuffer.getIntLE(0)
    val uuid = decryptedBuffer.getString(Int.SIZE_BYTES,Int.SIZE_BYTES+uuidLength)
    val data = decryptedBuffer.getBuffer(Int.SIZE_BYTES+uuidLength,decryptedBuffer.length())
    fun toBuffer() = buffer
    companion object {
        fun create(uuid:String,data:Buffer):RawData {
        val encryptedBuffer = Aes.encrypt(Buffer.buffer()
            .appendIntLE(uuid.length)
            .appendString(uuid)
            .appendBuffer(data).bytes)

        return RawData(Buffer.buffer()
            .appendIntLE(Flag.RAW.ordinal)
            .appendBytes(encryptedBuffer))
        }
    }
}
```
到此，服务端的功能就实现了。接下来就是如何实现客户端

## 客户端
客户端与浏览器交互的部分，我们选择Socks5协议，这个Vert.x库不带支持，所以需要自己实现以下。实现代码可以看Wsocks的ClientSocks5类，这里只展示客户端与服务端的交互。
```Kotlin
//核心部分
httpClient.websocket(remotePort,remoteIp,"/proxy"){ webSocket ->
      webSocket.binaryMessageHandler {buffer->
        if (buffer.length() < 4) {
          return@binaryMessageHandler
        }
        when (buffer.getIntLE(0)) {
          //连接成功
          Flag.CONNECT_SUCCESS.ordinal -> wsConnectedHandler(ConnectSuccess(buffer).uuid)
          //出现异常
          Flag.EXCEPTION.ordinal -> wsExceptionHandler(Exception(buffer))
          //目标服务器返回数据
          Flag.RAW.ordinal -> wsReceivedRawHandler(RawData(buffer))
          else -> logger.warn(buffer.getIntLE(0))
        }
      }
    }
```
```Kotlin
//处理器部分
private fun wsConnectedHandler(uuid:String){
    val netSocket = connectMap[uuid]?:return
    netSocket.handler {
      //将浏览器的数据，加密发送到服务端
      ws.writeBinaryMessage(offset,RawData.create(uuid,it).toBuffer())
    }
    //告诉浏览器连接成功
    val buffer = Buffer.buffer()
      .appendByte(0x05.toByte())
      .appendByte(0x00.toByte())
      .appendByte(0x00.toByte())
      .appendByte(0x01.toByte())
      .appendBytes(ByteArray(6){0x0})
    netSocket.write(buffer)
}
private fun wsReceivedRawHandler(data: RawData){
    val netSocket = connectMap[data.uuid]?:return
    bufferSizeHistory+=data.data.length()
    //把解密后的数据发送给浏览器
    netSocket.write(data.data)
}
private fun wsExceptionHandler(e:Exception){
    //出现异常，断开浏览器本次连接
    connectMap.remove(e.uuid)?.close()
}
```
至此，浏览器和客户端，客户端和服务端，服务端和目标服务器之间的数据交互就完成了。

## 加密
由于我只是个普通写后台的，并不是专业的密码学研究者，所以对于加密这块内容，也不敢做过多的分析。但是结合GFW所处的实际情况，我觉得自己还是可以稍微评论一下的。实际上Github上也还有一些类Shadowsocks的产品，比如说`Lightsocks`，star数也挺高。它们有些产品采用了自己开发的加密算法，而不是`Aes`，`Rc4`，之类的主流加密。按照作者的意思是，采用自己开发的加密能够更加有效的方式GFW解密。  
这么说当然也是有一定道理的，但是实际上这些自研的算法往往比较脆弱，更容易遭到像词频分析之类的方法解密。但是其实我们还要考虑一个问题，GFW只是一个部署在主干网络上的计算机集群，它不是神，它的模型、运算量都是有限的， 每秒都有大量的流量经过它，要通过分析流量*解密*数据很明显是不可能的事情，就算我们总说Aes128过时了，Aes128有漏洞，但是针对Aes128的攻击依然条件苛刻。  
在这里还可以举个例子，SSL我们都认为它是安全的，但是针对SSL或者说HTTPS的*中间人*攻击是存在的，只是这种攻击实现的原理绝对不是分析加密后的数据，而是通过分析**握手环节**的数据拿到秘钥来解密后续的数据。同样的，GFW也是这么做的，而对于这种情况，只要秘钥不出现在流量中，GFW就很难有操作的空间了。  
还可以再举另一个例子，杀毒软件判断一个程序是否是病毒、木马，靠的是**特征码**和**行为分析**，**特征码**就是病毒为了执行某一系列操作而必定存在的代码，而**行为分析**则是把病毒放在沙箱环境内，观察病毒做了哪些操作。GFW也是如此，在流量中查找特征码比解密流量要容易的多，像Shadowsocks这样的程序产生的流量特征是很明显的，除此之外，GFW还拥有主动探测的能力，在发现特征后，它会尝试构造特殊的报文发送给目标，并根据目标的行为判断是否为Shadowsocks服务端。这种嗅探的成本也是非常低的。 

## 完善
就如我在前面写的，使用Websocket这样的协议能够把我们的流量伪装成正常的网站流量。当然这并不完善，因为这样的程序一旦多起来，GFW也一定会开发出针对Websocket的特征分析。这个时候就需要一种更加灵活的方式，比如在数据头部填充，在报文的某几个位置插入字节。这些都可以破坏报文的特征，就像我们给病毒修改入口点、加壳、加花，来起到免杀的效果一样。  
针对GFW的主动嗅探，则可以考虑动态白名单之类的机制，想要发送请求，就要到另一台服务器上登录，登录过程可以经过国内一台服务器做跳板，这样对于GFW来说，整个过程就是两组不相干的流量。如果要将这样的流量放在一起建立模型的话，也许要到2050年吧。

## 写在最后
国内普遍言论还是把GFW神化了，这东西确实很强，也不知道是哪些研究所在负责维护，但是毕竟算法再强，模型再完美也要受硬件所限。这方面，Google没有解决的问题，中国政府也尚且没有这样的实力。很多时候，解决问题还是不要硬刚，换个角度想一下，可能效果更好。