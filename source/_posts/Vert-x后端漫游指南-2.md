---
title: Vert.x后端漫游指南(2)
date: 2018-07-02 17:30:06
tags:
  - Vert.x
---

## **1.前情提要**
上一篇我们已经学会了怎么实现一个最简单的Http服务端。这一篇我们会介绍怎么写一个用于登录的API。

## **2.路由**
或许你们已经看过了HttpServerRequest这个接口，并且发现了这个接口中有一个返回String的方法叫做absoluteURI。稍微想一想就能发现，其实只需要对这个absoluteURI的返回值进行一些正则匹配，我们就能实现路由的功能。在实现了路由之后，我们还可以通过getParam方法获取到请求的参数。所以，如果只是做一个Demo，我们甚至不需要Vert.x的web扩展。  
但是我们都是要干一番大事业的对吧。所以在这里我们要引入io.vertx.ext.web.Router。这个接口的名称已经够说明问题了，它能让我们建立像express中提供的路由。那么让我们来创建第一个路由吧
```Kotlin
import io.vertx.ext.web.Router
import io.vertx.core.Vertx
fun main(args:Array<String>){
  val vertx = Vertx.vertx()
  ///////////////////Add Code here//////////////////////
  val router = Router.router(vertx)
  router.route("/login/:username/:password").handler{
    val username = it.request().getParam("username")
    val password = it.request().getParam("password")
    if(username=="admin" && password="admin123"){
      it.response().end("{status:0}")
    }else{
      it.response().end("{status:-1}")
    }
  }
  //////////////////////////////////////////////////////
  vertx.createHttpServer().requestHandler{
    router.accept(it)
  }.listen(80){
    if(it.failed())
      it.cause.printStackTrace()
    else
      println("Listening on 80")
  }
}
```
这样我们就创建好了一个登录路由。这个路由接受两个参数，分别是username和password，用户在请求的时候大致是这样的http://example.com/login/admin/admin123 。如果你对http协议比较熟悉的话，你可能会更加习惯http://example.com/login?username=admin&password=admin123 这样的写法。其实这两者在Vert.x中是一样的。都可以通过
```Kotlin
request.getParam("param name")
```
这样的方式来获取，只不过前者需要在创建路由的时候设定好参数的名称以及位置，而后者的参数名称只在handler中出现。近几年越来越流行使用第一种方式，确实这样的api会更加的简洁，同时也不用把参数名称暴露在url中多少起了点保密的作用。  
除了创建一个路由之外，我们还修改了requestHandler中的代码。我们将本来的
`response().end("......")`换成了`router.accept(it)` 。相信这部分也非常的好理解，我们要让我们的router参与到HttpServer的处理中来，于是Router就提供了accept这个方法，它接收一个HttpServerRequest参数，在经过许多内部处理之后将一个包装好的RoutingContext传给我们的handler。这里其实还有另一种写法
```Kotlin
vertx.createHttpServer.requestHandler(router::accept).listen(80)
```
当然，这只是Kotlin的一点小技巧，似乎在Java8里也有类似的写法。

## **3.更完整的登录**
路由在使用的过程中最核心的部分就是它的RoutingContext，这个Context把完全囊括了后端所需要的所有东西。对于用户的请求，我们可以通过request()获取，当我们想响应用户的时候，我们可以调用response()，而对于Cookie和Session，RoutingContext也为他们提供了相应的方法。  
现在就让我们把这个登录做的更加完善一点。我们都知道，一个正常的登录过程中，服务端应该在用户登录之后保存一个session，并给用户一个Cookie，这两者就是一个键值对的关系。登录成功之后用户发送的每个请求都会带着这个Cookie，而服务端就通过这个Cookie去查询它对应的数据，比如说:用户名，用户id，用户权限等。  
在使用Session之前，我们需要添加一点代码来初始化一些东西
```Kotlin
val store = LocalSessionStore.create(vertx)
router.route().handler(CookieHandler.create())
router.route().handler(SessionHandler.create(store))
```
Cookie的部分非常简单，没有什么可说的。但是在创建SessionHandler的时候，我们发现它需要一个SessionStore参数。Vert.x提供了两种SessionStore，一种是我们使用的_LocalSessionStore_而另一种是_ClusteredSessionStore_。字面来看，一种是本地存储的Session，而另一种是集群存储的Session，两者的区别也就非常明显了。本地存储的Session只存储在一个Vertx实例中，而集群存储则可以在多个实例中共享Session。
```Kotlin
//创建集群会话存储
val store = ClusteredSessionStore.create(vertx, "myclusteredapp3.sessionmap");
```
本系列中不会涉及到集群有关的内容，尽管分布式集群也是Vert.x一个非常明显的优势，但是由于作者本人也没有参与过这类分布式项目，在这方面经验浅薄，只能等之后有机会接触了再做补充。  
在创建完SessionStore和SessionHandler之后，我们就可以在自己的handler中处理Session了
```Kotlin
router.route("/login/:username/:password").handler{
  if(it.session().get<String>("username")=="admin"){
    it.response().end("{status:1,msg:\"You have logged in \"}")
    return@handler
  }
  val username = it.request().getParam("username")
  val password = it.request().getParam("password")
  if(username=="admin" && password="admin123"){
    it.session().put("username","admin")
    it.session().put("level","1")
    it.response().end("{status:0}")
  }else{
    it.response().end("{status:-1}")
  }
}
```
这样我们的登录api就可以判断用户是否已经登录。每当用户请求的时候，我们就会调用`it.session().get()`判断用户是否已经登录。而每当用户登录成功的时候，我们就将用户名和等级放入Session中。用于之后的权限控制。

## **4.一个可以投入使用的登录**
一个正常的登录不应该是这样的对不对，没有人会把用户名和密码写在代码里。这么说可能不太对，其实也是有一些情况下，我们也会见到把用户名密码写在代码里的情况，比如说什么asp大马，php小马(不知道现在这种东西现在还有没有了)之类的东西或者是不靠谱的外包团队。  
那么抛开这些另类的情况，为了实现一个登录功能，我们得有个数据库对不对。假如说我们现在已经有一个注册页面了，那么用户注册后的信息都应该放在数据库里，而登录的过程就是读数据库的过程。  
Vert.x提供了很多操作数据库的包。这个系列里我们选择使用**vertx-mysql-postgresql-client**作为我们与数据库交互的客户端，同时呢，我们选择Mysql作为数据库。如果你从来没有接触过数据库的操作，那么你可以把Mysql想象成一个开放着3306端口的Excel表格，它允许连接到这个端口并且登录成功的客户端通过sql语句增删改查表格中的数据。  
那么首先，我们要让我**vertx-mysql-client**连接到Mysql上。
```Kotlin
import io.vertx.ext.asyncsql.AsyncSQLClient
import io.vertx.ext.asyncsql.MySQLClient
val sqlClient = MySQLClient.createShared(vertx,JsonObject("host" to "localhost"
        ,"username" to "root"
        ,"password" to "root"
        ,"database" to "example"))
```
这样我们就创建了一个_sqlClient_，如果连接成功的话。  
这是关于MysqlClient的详细配置
```JSON
{
  "host" : <主机地址>,
  "port" : <端口>,
  "maxPoolSize" : <最大连接数>,
  "username" : <用户名>,
  "password" : <密码>,
  "database" : <数据库名称>,
  "charset" : <编码>,
  "queryTimeout" : <查询超时时间-毫秒>
}
```
当然我们也可创建一个NonShared的sqlClient，创建方法是一样的，并且在这种只有一个Vert.x实例的工程中，两者的效果也是一样的。  
现在我们要把这个sqlClient放到我们的RouterHandler中。
```Kotlin
router.route("/login/:username/:password").handler{
  if(it.session().get<String>("username")=="admin"){
    it.response().end("{status:1,msg:\"You have logged in \"}")
    return@handler
  }
  val username = it.request().getParam("username")
  val password = it.request().getParam("password")
  sqlClient.querySingleWithParams("SELECT id,level FROM user WHERE username=? and password=?",JsonArray(username,password)){
    if(it.failed() || it.result()==null){
      it.response().end("{status:-1}")
    }else{
      it.session().put("username",username)
      it.session().put("uid",it.result().getInteger(0))
      it.session().put("level",it.result().getInteger(1))
      it.response().end("{status:0}")
    }
  }
}
```
我们对判断部分做了点修改，使我们能够使用数据库中的数据。当查询到用户名和密码都与用户输入匹配的结果时，我们就将查询结果存入到session中。

## **5.小结**
这一篇算是比较详细的讲了怎么实现一个能够使用的登录API，当然也不是很全面。在Router里还有很多内容可以写，会看情况在之后补充。下一篇应该会继续讲MysqlClient的使用，然后把curd全部写一下。
```Kotlin
//完整代码
import io.vertx.ext.web.Router
import io.vertx.core.Vertx
import io.vertx.ext.asyncsql.AsyncSQLClient
import io.vertx.ext.asyncsql.MySQLClient
//在本例中，sqlClient、vertx、router写在什么地方都无所谓。如果你用Java，请把他们放在他们应该在的地方
val sqlClient = MySQLClient.createShared(vertx,JsonObject("host" to "localhost"
        ,"username" to "root"
        ,"password" to "root"
        ,"database" to "example"))
fun main(args:Array<String>){
  val vertx = Vertx.vertx()
  val router = Router.router(vertx)
  val store = LocalSessionStore.create(vertx)
  router.route().handler(CookieHandler.create())
  router.route().handler(SessionHandler.create(store))
  router.route("/login/:username/:password").handler{
    if(it.session().get<String>("username")=="admin"){
      it.response().end("{status:1,msg:\"You have logged in \"}")
      return@handler
    }
    val username = it.request().getParam("username")
    val password = it.request().getParam("password")
    sqlClient.querySingleWithParams("SELECT id,level FROM user WHERE username=? and password=?",JsonArray(username,password)){
      if(it.failed() || it.result()==null){
        it.response().end("{status:-1}")
      }else{
        it.session().put("username",username)
        it.session().put("uid",it.result().getInteger(0))
        it.session().put("level",it.result().getInteger(1))
        it.response().end("{status:0}")
      }
    }
  }
  vertx.createHttpServer().requestHandler{
    router.accept(it)
  }.listen(80){
    if(it.failed())
      it.cause.printStackTrace()
    else
      println("Listening on 80")
  }
}
```
