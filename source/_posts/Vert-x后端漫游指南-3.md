---
title: Vert.x后端漫游指南(3)
date: 2018-07-03 02:20:09
tags:
  - Vert.x
---

## **1.异步之痛**
上一篇中，我们为登录部分加入了读取数据库的操作。如果你熟悉JDBC的话，应该很快就发现了Vert.x中异步的数据库操作与JDBC中同步操作的不同。我们不再像以前一样等待着数据库的返回值，相反的，我们将自己的后续操作包装在一个lambda中，作为参数传递给`query`方法。在查询完成之后，_SqlClient_ 会来调用我们传入的lambda并将查询结果作为参数传递给我们。  
这样的过程就是一个非常经典的异步过程，似乎非常的简单并且也没有什么不合理的地方。但是仔细想一想我们实际的需求，就会发现这个过程好像并没有这么美好。很多情况下，我们要在一个一个请求处理中多次操作数据库，现在假设我们在做一个社交网站。用户登录之后我们要返回用户的基本信息，用户的好友信息，还有用户曾经发表的图片、文章。很明显，我们要做三次查询操作，于是我们的代码就变成了这样
```Kotlin
sqlClient.querySingleWithParams("SELECT id,birthday,full_name,nick_name,email FROM user WHERE uname=? and pwd=?",JsonArray(uname,pwd)){
  if(it.failed()){
    //DO some
  }else{
    val uid = it.result().getInteger(0)
    sqlClient.queryWithParams("SELECT * FROM friends WHERE uid=?",JsonArray(uid)){
      if(it.failed()){
        //Do some
      }else{
        sqlClient.queryWithParams("SELECT * FROM posts WHERE uid=?",JsonArray(uid)){
          if(it.failed()){
            //Do some
          }else{
            context.response().end(/*All infomations*/)
          }
        }
      }
    }
  }
}
```
很多的业务逻辑要求我们必须保证代码的执行顺序，这对于同步的代码来说非常的简单，但是对于异步的构架，我们不得不做一点改变，否则我们的代码会像金字塔一样越叠越高。  
为了解决这种问题，我们有很多种选择。如果你不幸选择使用Java的话，RxJava应该会成为你最好的选择，但是如果你选择Kotlin，那么你就有了一个非常简单的解决方案。
## **2.协程**
#### 1）协程能干嘛
>协程把异步编程放入库中来简化这类操作。程序逻辑在协程中顺序表述，而底层的库会将其转换为异步操作。库会将相关的用户代码打包成回调，订阅相关事件，调度其执行到不同的线程（甚至不同的机器），而代码依然想顺序执行那么简单。

这段话似乎没有那么容易理解。让我们直接来看代码吧
```Kotlin
/*未使用协程*/
sql.querySingle("SELECT id FROM example"){
  if(it.failed()){
    context.response().end("{status:-1}")
  }else{
    contezt.response().end("{status:1,id:${it.result().getInteger(0)}")
  }
}
/* 使用协程 */
val result = sql.querySingle("SELECT id FROM example").awit()
if(it.failed()){
  context.response().end("{status:-1}")
}else{
  contezt.response().end("{status:1,id:${it.result().getInteger(0)}")
}
```
两者的区别显而易见。在使用了协程之后我们能够将异步的代码转变为“同步”的代码。熟悉的赋值，没有了lambda参数，代码的执行顺序一下子就变得非常清晰了。这就是协程的作用
#### 2) 协程怎么用
Kotlin的文档`http://kotlinlang.org/docs/reference/coroutines.html`中，对于协程的使用有非常详细的说明。这里就只提供一种代码。  
协程不是什么地方都能用的。我们需要一个launch来启动协程运行的上下文
```Kotlin
fun Async(block:suspend ()->Unit){
    block.startCoroutine(object : Continuation<Unit> {
        override val context: CoroutineContext
            get() = EmptyCoroutineContext

        override fun resumeWithException(exception: Throwable) {
            exception.printStackTrace()
        }

        override fun resume(value: Unit) {}

    })
}
```
有了启动代码还是不能直接使用Vert.x的异步API，我们需要将API做一些转换
```Kotlin
/* 柯理化 */
fun AsyncSQLClient.querySingleWithParamsCurry(query:String,json:JsonArray)
      =fun(handler:Handler<AsyncResult<JsonArray>>)
      = this.querySingleWithParams(query,json,handler)
```
在完成了柯理化之后，我们还需要提供一个awit方法
```Kotlin
suspend fun <T,R> ((Handler<T>)->R).awit():T= suspendCoroutine{ con->
    this(Handler {
        con.resume(it)
    })
}
```
在完成了上述代码过程之后，我们就可以使用协程了
```Kotlin
router.route("/login/:username/:password").handler{
  if(it.session().get<String>("username")=="admin"){
    it.response().end("{status:1,msg:\"You have logged in \"}")
    return@handler
  }
  val username = it.request().getParam("username")
  val password = it.request().getParam("password")
  Async{
    val result = sqlClient.querySingleWithParamsCurry("SELECT id,level FROM user WHERE username=? and password=?",JsonArray(username,password)).awit()
    if(result.failed() || result.result()==null){
      it.response().end("{status:-1}")
    }else{
      it.session().put("username",username)
      it.session().put("uid",result.result().getInteger(0))
      it.session().put("level",result.result().getInteger(1))
      it.response().end("{status:0}")
    }
  }
}
```
#### 3)协程本质
>协程的底层实现就是个状态机

想要把Kotlin中的协程引入现在的Java中是不可能的，协程是一种依赖编译器支持的语法糖。在上面这个例子中，Async内的代码经过编译之后会大致变成这个样子
```Kotlin
fun afunction(status:Int=0,result:AsyncResult<JsonArray>?=null):Unit{
  when(status){
    0->{
      sqlClient.querySingleWithParamsCurry("SELECT id,level FROM user WHERE username=? and password=?",JsonArray(username,password))({afunction(1,it)})
    }
    1->{
      if(result.failed() || result.result()==null){
        it.response().end("{status:-1}")
      }else{
        it.session().put("username",username)
        it.session().put("uid",result.result().getInteger(0))
        it.session().put("level",result.result().getInteger(1))
        it.response().end("{status:0}")
      }
    }
  }
}
```
这段代码并不准确，但是编译器的意图大致就是这样的。最终我们看到，该异步的还是异步了，只是编译器替我们做了许多事情，让我们的代码看起来像是同步的而已。

## **3.来个注册**
在写完了异步的痛处与Kotlin中针对异步问题的处理后，我们来写一个用于注册的路由吧。注册与登录在代码方面差不了太多，唯一要注意的就是，SQLClient为INSERT、UPDATE、DELETE提供了与SELECT不同的方法。
```Kotlin
AsyncSQLClient.updateWithParams("INSERT INTO example (a,b,c) VALUES (?,?,?)",JsonArray(a,b,c)){/* Do some */}
```
乍看与SELECT中唯一的不同就是`query`变成了`update`。但实际上除了这一点，函数接收的Handler也不一样。之前query接收的参数是`Handler<AsyncResult<ResultSet>>`,而update接收的参数却是 `Handler<AsyncResult<UpdateResult>>` 。这个UpdateResult接口提供了两个比较重要的方法，一个是getUpdated(),这个方法返回一个更新的条数。而另一个getKeys()则返回所有生成的主键。这一点在插入有父子关系的表时会非常的有用。
```Kotlin
/* 注册部分代码 */
router.route("/register/:email/:username/:password/").handler{
  val username = it.request().getParam("username")
  val email = it.request().getParam("email")
  val password = it.request().getParam("passwrd")
  if(username==null || email==null || password==null){
    it.response().end("{status:-1}")
    return@handler
  }
  Async{
    val result = sqlClient.updateWithParamsCurry("INSERT INTO user (email,username,password) VALUES (?,?,?)",JsonArray(email,username,password)).awit()
    if(result.failed() || result.result().updated==0){
      it.response().end("{status:-1}")
    }else{
      it.response().end("{status:1}")
    }
  }
}
```
非常简单，注册的部分也就完成了。

## **4.小结**
也许Spring中Java是最好的选择，但是在Vert.x中Java可以说是一无是处了。在Kotlin中使用协程是一件非常愉快的事情，尽管到现在为止，Kotlin标准库的协程部分还是带着“实验性”的名称，而且偶尔会出现编译期的bug，但是相信在(可能)明年的Kotlin1.3中，协程会去掉“实验性”的标志，并变得更加完善。
