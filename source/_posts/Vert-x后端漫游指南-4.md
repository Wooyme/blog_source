---
title: Vert.x后端漫游指南(4)
date: 2019-01-17 18:09:03
tags:
    - Vert.x
---

# 前言
非常非常久没有更新 *Vert.x后端漫游指南* 了。主要是因为之前觉得Vert.x的Web后端实在是没有什么好写的，都是一些常规操作。再者，也有很长一段时间没有碰Vert.x的后端开发了。但是呢，因为打算做一个Vert.x全栈的服务端应用，所以又把Web后端的部分拿出来了。

# API？
虽然Vertx-Web也支持模板引擎这类的东西，但是无论是从潮流还是从Vertx支持程度的角度来看，前后端分离的开发方式更加适合现在的Web开发。那么既然前后端分离，提供一套完整、合理的API就非常重要了。  
回顾一下之前的 *Vert.x后端漫游指南* ,我们使用了`Router.route("/api/path")`的方式来设置API。这种方式非常的简单粗暴，没有注解，没有配置文件，完完全全的硬编码。对于单人开发的简单服务来说，这样当然是没有问题的。但是当有了开发团队，需要前后端沟通的时候，这种硬编码的方式就变得很麻烦了。  
一方面这意味着还得再单独写一份API文档给前端，
> 程序员最讨厌两种事情，一是没有文档，二是写文档。

另一方面是一旦要做什么修改，就必须要重新编译程序。  

# Swagger & OpenAPI
> Swagger https://swagger.io/

> OpenApi https://www.openapis.org/

OpenAPI和Swagger给API开发带来了不一样的体验。Swagger是一套非常完善的API开发工具，最新的版本已经提供了对OpenAPI 3.0 的支持。

## yaml
Swagger使用yaml作为配置文件，
```yaml
openapi: 3.0.0
info:
    title: Sample API
    description: Optional multiline or single-line description in [CommonMark](http://commonmark.org/help/) or HTML.
    version: 0.1.9
servers:
  - url: /v1
    description: Optional server description, e.g. Main (production) server
  - url: http://staging-api.example.com
    description: Optional server description, e.g. Internal staging server for testing    
paths:
    /api/transactions/{id}:
        get:
            operationId: getTransactionsList
            parameters:
                -   name: id
                    in: path
                    required: true
                    schema:
                        type: string     
                -   name: from
                    in: query
                    required: false
                    schema:
                        type: string
                -   name: to
                    in: query
                    required: false
                    schema:
                        type: string
            responses: 
                200:
                    content:
                        application/json:
                            schema:
                                type: object
        put:
            operationId: putTransaction
            parameters:
                -   name: id
                    in: path
                    required: true
                    schema:
                        type: string
            requestBody:
                required: true
                content:
                    application/json:
                        schema:
                            type: object
                            properties:
                                from:
                                    type: string
                                    format: email
                                to:
                                    type: string
                                    format: email
                                value:
                                    type: number
                                    format: double
                            additionalProperties: false
                            required:
                                - from
                                - to
                                - value
            responses: ...
```
大致上就是这个样子,详细的关于swagger中yaml配置的编写可以看官方文档
> https://swagger.io/docs/specification/basic-structure/

总体来说就是把原来的硬编码改成了文件配置的模式。通过`paths`和`get`,`put`,`post`,`delete`一起组成各种API，同时也通过`parameters`和`requestBody`规定了各个API所需的参数，以及最后的`responses`用于规定服务端返回的数据结构。  

## Swagger-Codegen
有了配置文件之后当然还是要能够生成对应的代码才行。Swagger提供了Swagger-Codegen来生成各种语言、框架的服务端和客户端代码。
> https://github.com/swagger-api/swagger-codegen

如果要使用**OpenAPI 3.0**的话就必须要安装3.0.0版本以上的Swagger-Codegen。目前由于这个工具还在频繁的更新中，所以bug也是挺多的，特别是在客户端代码的部分，前几天就遇到生成的`typescript-angular`的代码里没有把body放到`post`和`put`的参数里的问题。  
如果开发的时候遇到了不可理喻的问题，就去issues里看看吧。

# Vert.x的Web API Contract
在写完Swagger之后呢，就要来看看Vert.x的Web API Contract了。为什么明明有Swagger了，Vert.x的团队还要搞个Contract呢，因为**Swagger不支持Vert.x**( 笑 。Swagger虽然已经支持很多主流框架了，但是很明显Vert.x还不是那么的"主流"。

>官方文档 https://vertx.io/docs/vertx-web-api-contract/java/

## 构建
Maven
```xml
<dependency>
 <groupId>io.vertx</groupId>
 <artifactId>vertx-web-api-contract</artifactId>
 <version>3.6.2</version>
</dependency>
```
Gradle
```Groovy
dependencies {
 compile 'io.vertx:vertx-web-api-contract:3.6.2'
}
```
## 创建一个工厂
```Java
OpenAPI3RouterFactory.create(vertx, "yaml的路径,支持本地文件或是http、https的地址", ar -> {
  if (ar.succeeded()) {
    //得到了一个路由工厂
    OpenAPI3RouterFactory routerFactory = ar.result();
  } else {
    Throwable exception = ar.cause();
  }
});
```
得到这样一个工厂之后，就可以添加各种处理器
```Java
routerFactory.addHandlerByOperationId("awesomeOperation", routingContext -> {
  RequestParameters params = routingContext.get("parsedParameters");
  RequestParameter body = params.body();
  JsonObject jsonBody = body.getJsonObject();
  // Do something with body
});
routerFactory.addFailureHandlerByOperationId("awesomeOperation", routingContext -> {
  // Handle failure
});
```
这里的`OperationId`就对应了yaml文件中的`OperationId`，对于需要做权限控制的路由，还可以添加`Security Handler`
```Java
routerFactory.addSecurityHandler("security_scheme_name", securityHandler);
```
对于之前我们用到的SessionHandler、CookieHandler，可以通过调用`addGlobalHandler`的方式来添加。需要注意的是BodyHandler不能用`addGlobalHandler`，`OpenAPI3RouterFactory`为它提供了专门的方法。

在添加完处理器之后，就可以生产路由了
```Java
Router router = routerFactory.getRouter();

HttpServer server = vertx.createHttpServer(new HttpServerOptions().setPort(8080).setHost("localhost"));
server.requestHandler(router).listen();
```
这部分跟之前相差无几，唯一要注意的是`router::accept`已经被弃用了，现在直接向`requestHandler`传递`router`就可以了。

# Vert.x的Web API Service
> Contract之后再来个Service是要干什么。

各位肯定也都发现了，Web API Contract用起来是一点都不方便，先不说每个处理器都要通过`operationId`单独绑定，就说处理器得到的参数吧，一点都不智能，完全没有用到Yaml中配置的`parameters`，参数还是要像以前一样从`RoutingContext`里获取。  
为了获得更好的开发体验，就必须要引入Web API Service了。
> 官方文档 https://vertx.io/docs/vertx-web-api-service/java/

## 构建
Maven
```xml
<dependency>
 <groupId>io.vertx</groupId>
 <artifactId>vertx-codegen</artifactId>
 <version>3.6.2</version>
 <classifier>processor</classifier>
</dependency>
<dependency>
 <groupId>io.vertx</groupId>
 <artifactId>vertx-web-api-service</artifactId>
 <version>3.6.2</version>
</dependency>
```
Gradle
```Groovy
dependencies {
 compile 'io.vertx:vertx-codegen:3.6.2:processor'
 compile 'io.vertx:vertx-web-api-service:3.6.2'
}
```
可以看到，除了`vertx-web-api-service`以外，还引入了`vertx-codegen`。在codegen面前，没有什么问题是解决不了的。

## 使用Interface
```Java
@WebApiServiceGen
interface TransactionService {
 void getTransactionsList(String from, String to, OperationRequest context, Handler<AsyncResult<OperationResponse>> resultHandler);
 void putTransaction(JsonObject body, OperationRequest context, Handler<AsyncResult<OperationResponse>> resultHandler);
}
```
Web API Service提供了另一种绑定handler的方式。接口中的方法名称需要和`operationId`相同，其中的参数也需要和`parameters`相同，需要注意的是如果使用`requestBody`，则对应`JsonObject body`。最后的`OperationRequest context`和`Handler<AsyncResult<OperationResponse>> resultHandler`是必须写的部分。分别代表了请求的额外信息和响应的对象。

- OperationRequest
  - getHeaders 请求头信息
  - getParams 请求参数
  - getUser 如果使用Auth，User就在这里
  - getExtra 可以通过向RouterFactory添加`setExtraOperationContextPayloadMapper`来**额外设置extra**，这个很重要,通过这种方式可以把session带到handler里来。
- OperationResponses
  - 响应头
  - 响应状态
  - 响应主体
  
## 添加Service到RouterFactory
```Kotlin
OpenAPI3RouterFactory.create(vertx,"yaml文件路径"){
    val serviceImpl = ServiceImpl()
    ServiceBinder(vertx)
        .setAddress("my-service")
        .register(Service::class.java,ServiceImpl)
    it.result().mountServiceInterface(Service::class.java,"my-service")
    //因为Web API Contract并不会处理yaml中的servers，所以需要自己设置sub-router
    val router = Router.router(vertx).mountSubRouter("/api",it.result().router)
    vertx.createHttpServer().requestHandler(router).listen(80){
        logger.info("Listen at 80")
    }    
}
```
这样就可以非常方便的使用Vert.x的Web API Service了。

# 小结
在使用了OpenAPI之后，Vert.x的后端开发就算是比较完整了。这种开发方式也能给前端开发提供很多便利，比硬编码的模式要更加适合现在的开发形式。