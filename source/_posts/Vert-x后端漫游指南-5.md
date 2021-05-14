---
title: Vert.x后端漫游指南(5)
date: 2019-10-28 19:10:14
tags:
    - Vert.x
    - OpenAPI
---

# 前言
半年更的*Vert.x后端漫游指南又来了* 。这次的tags除了熟悉的`Vert.x`以外，还多了一个 `OpenAPI`。要是看过上一篇的话，应该会记得在上一篇中，我们使用了`Swagger`和`Web API Contract`来进行API的开发。而本篇我们要用一种更加简单的方式来编写API

# OpenAPI Generator
github
> https://github.com/OpenAPITools/openapi-generator

首先介绍一下**openapi-generator**，这是一个代码生成工具，可以看到在github上已经有超过3.5k的star。这个工具其实就是swagger的升级版，并且采用了社区化的开发方式，也就是说其他开发者也可以为开发团队提供代码，将自己想要的功能添加到openapi-generator中去。  

在老版本的openapi-generator中，已经集成了**java-vertx**的生成器，而在最新的4.2.0版本(https://oss.sonatype.org/content/repositories/snapshots/org/openapitools/openapi-generator-cli/4.2.0-SNAPSHOT/openapi-generator-cli-4.2.0-20191018.031007-71.jar)中加入了**kotlin-vertx(beta)**。于是*Vert.x后端漫游指南5*就有了着落。

顺带一提，**kotlin-vertx**的贡献者是我。再顺带一提，**有bug**。

## usage
下载openapi-generator之后，从终端启动
```
java -jar openapi-generator-cli-4.2.0.jar generate -g kotlin-vertx -i example.yaml -o server
```
会生成出一个`server`文件夹，里面是一个完整的*maven*工程。在其中的verticle目录下，可以看到三种文件，分别是`XXApi`
,`XXApiVerticle`和`XXApiVertxProxyHandler`，而我们要做的只有新建一个`XXApiImpl`，继承`XXApi`，实现里面的所有方法。

# 继承Api
Api文件中大概是这个样子的
```kotlin
interface MyApi  {
    suspend fun listData(deviceId:kotlin.String?,sensorId:kotlin.Int?,offset:kotlin.Int?,limit:kotlin.Int?,start:kotlin.Long?,end:kotlin.Long?,context:OperationRequest):Response<kotlin.Array<Data>>
    companion object {
        const val address = "MyApi-service"
        suspend fun createRouterFactory(vertx: Vertx,path:String): io.vertx.ext.web.api.contract.openapi3.OpenAPI3RouterFactory {
            val routerFactory = OpenAPI3RouterFactory.createAwait(vertx,path)
            routerFactory.addGlobalHandler(CookieHandler.create())
            routerFactory.addGlobalHandler(SessionHandler.create(LocalSessionStore.create(vertx)))
            routerFactory.setExtraOperationContextPayloadMapper{
                JsonObject().put("files",JsonArray(it.fileUploads().map { it.uploadedFileName() }))
            }
            val opf = routerFactory::class.java.getDeclaredField("operations")
            opf.isAccessible = true
            val operations = opf.get(routerFactory) as Map<String, Any>
            for (m in LinkApi::class.java.methods) {
                val methodName = m.name
                val op = operations[methodName]
                if (op != null) {
                    val method = op::class.java.getDeclaredMethod("mountRouteToService",String::class.java,String::class.java)
                    method.isAccessible = true
                    method.invoke(op,address,methodName)
                }
            }
            routerFactory.mountServiceInterface(LinkApi::class.java, address)
            return routerFactory
        }
    }
}
```
其中，`companion object`是Api自己的东西，我们要自己重写`listData`方法。到这里就跟上一篇很像了，包括所使用的OperationRequest也是和上一篇中一模一样。
